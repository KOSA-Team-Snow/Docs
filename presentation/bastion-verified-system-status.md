# Bastion 실측 기반 시스템 상태 정리

확인 시각: 2026-05-26 07:14 KST  
접속 위치: `kosa@172.16.44.100` (`bastion`)  
Kubernetes context: `kubernetes-admin@kubernetes`

## 1. 확인 요약

현재 온프레미스 Kubernetes 클러스터와 FlaskApp 운영 경로는 정상 확인되었다.

확인 완료:

- Bastion 접속 가능: `172.16.44.100`
- Kubernetes API VIP 접속 가능: `172.16.43.99:6443`
- Kubernetes node 8대 모두 `Ready`
- Control Plane 3대, Worker 5대 구성 확인
- FlaskApp 2 replicas Running
- FlaskApp Ingress/VIP HTTP 200 확인
- MariaDB TCP 접속 확인: `172.16.43.160:3306`
- DB VM node_exporter 확인: `172.16.43.160:9100`
- DB VM mysqld_exporter 확인: `172.16.43.160:9104`
- ArgoCD 기준 `flaskapp`, `ingress-nginx`, `ceph-csi-rbd`, `logging`, `aiops` Healthy 확인
- HolmesGPT Pod Running 및 `/readyz` 응답 확인
- Loki/Alloy logging stack Running 확인

주의해서 말해야 할 부분:

- Grafana 외부 접속은 확인 시점에 `503 Service Temporarily Unavailable`이었다.
- 원인은 Ingress 문제가 아니라 `monitoring-grafana` Pod 내부 `grafana` 컨테이너 readiness 미완료였다.
- Prometheus와 Alertmanager는 Running이었다.
- ArgoCD 기준 `monitoring` Application은 `Synced / Progressing` 상태였다.
- AWS DR 실제 리소스 상태는 bastion에 AWS CLI가 없어 직접 조회하지 못했다. Terraform 코드와 DR runbook 스크립트는 로컬 repo에서 확인했다.

## 2. Kubernetes 클러스터 상태

확인 명령:

```bash
kubectl get nodes -o wide
```

확인 결과:

| Node | Role | Status | IP | Version |
| --- | --- | --- | --- | --- |
| `k8s-cp-1` | control-plane | Ready | `172.16.43.100` | `v1.30.1` |
| `k8s-cp-2` | control-plane | Ready | `172.16.43.101` | `v1.30.1` |
| `k8s-cp-3` | control-plane | Ready | `172.16.43.102` | `v1.30.1` |
| `k8s-worker-1` | worker | Ready | `172.16.43.110` | `v1.30.1` |
| `k8s-worker-2` | worker | Ready | `172.16.43.111` | `v1.30.1` |
| `k8s-worker-3` | worker | Ready | `172.16.43.112` | `v1.30.1` |
| `k8s-worker-4` | worker | Ready | `172.16.43.113` | `v1.30.1` |
| `k8s-worker-5` | worker | Ready | `172.16.43.114` | `v1.30.1` |

발표에서 말할 수 있는 내용:

> Kubernetes는 kubeadm 기반 HA control plane 3대와 worker 5대로 구성되어 있고, 확인 시점 기준 전체 node가 Ready 상태였다.

## 3. Namespace 구성

확인된 주요 namespace:

| Namespace | 용도 |
| --- | --- |
| `flaskapp-prod` | FlaskApp 운영 |
| `ingress-nginx` | NGINX Ingress Controller |
| `argocd` | GitOps 배포 |
| `monitoring` | Prometheus/Grafana/Alertmanager |
| `logging` | Loki/Alloy |
| `aiops` | HolmesGPT |
| `ceph-csi-rbd` | Ceph RBD CSI |
| `calico-system`, `calico-apiserver`, `tigera-operator` | Calico CNI |

## 4. FlaskApp 운영 상태

확인 명령:

```bash
kubectl get deploy,hpa,pdb,networkpolicy -n flaskapp-prod
kubectl describe deploy -n flaskapp-prod flaskapp
```

확인 결과:

- Deployment: `flaskapp`
- Replica: `2 desired / 2 available`
- Image: `080252689380.dkr.ecr.ap-northeast-2.amazonaws.com/flaskapp:18b68fe`
- ServiceAccount: `flaskapp-sa`
- HPA: `min 2`, `max 4`, CPU target `70%`
- PDB: `minAvailable: 1`
- NetworkPolicy: `flaskapp-networkpolicy`
- Probe:
  - startupProbe: `/info`
  - readinessProbe: `/info`
  - livenessProbe: `/info`

ConfigMap 확인:

```yaml
DATABASE_HOST: 172.16.43.160
DATABASE_USER: flaskapp
DATABASE_DB_NAME: flaskapp
AWS_DEFAULT_REGION: ap-northeast-2
PHOTOS_BUCKET: flaskapp-proddata-kosa-project-team3-snow-lai9z
```

발표에서 말할 수 있는 내용:

> FlaskApp은 Helm/ArgoCD로 배포되어 있고, 2개 replica가 Running 상태였다. HPA, PDB, probe, ConfigMap/Secret, NetworkPolicy가 적용되어 운영 안정성을 고려한 형태로 구성되어 있었다.

## 5. Ingress와 외부 접속 경로

현재 Ingress host:

| Service | Host | 경로 |
| --- | --- | --- |
| FlaskApp | `flaskapp.team.snow.internal` | `172.16.42.99` VIP -> NGINX Ingress -> `flaskapp-service` |
| Grafana | `grafana.team.snow.internal` | `172.16.42.99` VIP -> NGINX Ingress -> `monitoring-grafana` |

FlaskApp HTTP 확인:

```bash
curl -I http://172.16.42.99 -H 'Host: flaskapp.team.snow.internal'
```

결과:

```text
HTTP/1.1 200 OK
```

FlaskApp `/info` 응답 본문에서 `Employee Directory` 페이지 확인.

Grafana HTTP 확인:

```bash
curl -I http://172.16.42.99 -H 'Host: grafana.team.snow.internal'
```

결과:

```text
HTTP/1.1 503 Service Temporarily Unavailable
```

원인:

- `monitoring-grafana` Service endpoint가 비어 있었다.
- `monitoring-grafana` Pod는 `2/3 Running` 상태였고, `grafana` 컨테이너 readiness가 아직 통과하지 못했다.
- Pod 내부 `127.0.0.1:3000/api/health`도 확인 시점에는 connection refused였다.
- Grafana 로그는 DB migration을 수행 중이었다.

발표에서 말할 때 주의:

> FlaskApp 외부 접속은 VIP와 Ingress를 통해 정상 확인했다. Grafana는 배포되어 있으나 확인 시점에는 readiness가 완료되지 않아 외부 접속은 503이었다.

## 6. DB와 exporter 상태

MariaDB TCP 확인:

```bash
nc -vz -w 3 172.16.43.160 3306
```

결과:

```text
Connection to 172.16.43.160 3306 port [tcp/mysql] succeeded
```

DB VM exporter 확인:

```bash
curl http://172.16.43.160:9100/metrics | head
curl http://172.16.43.160:9104/metrics | head
```

결과:

- `node_exporter` 응답 확인
- `mysqld_exporter` 응답 확인

발표에서 말할 수 있는 내용:

> MariaDB는 Kubernetes 외부 VM `172.16.43.160`에서 운영되고 있으며, FlaskApp이 바라보는 DB endpoint와 일치한다. DB VM에는 node_exporter와 mysqld_exporter가 구성되어 Prometheus 수집 대상으로 사용할 수 있다.

## 7. ArgoCD 상태

확인 명령:

```bash
kubectl get applications -n argocd
```

최종 확인 상태:

| Application | Sync | Health |
| --- | --- | --- |
| `root-app` | Synced | Healthy |
| `flaskapp` | Synced | Healthy |
| `ingress-nginx` | Synced | Healthy |
| `ceph-csi-rbd` | Synced | Healthy |
| `logging` | Synced | Healthy |
| `aiops` | Synced | Healthy |
| `monitoring` | Synced | Progressing |

발표에서 말할 수 있는 내용:

> 주요 Application은 ArgoCD 기준 Synced 상태였다. FlaskApp, Ingress, Logging, AIOps는 Healthy였고, Monitoring은 Grafana readiness 때문에 Progressing 상태였다.

## 8. Logging과 AIOps 상태

Logging:

- Alloy DaemonSet Pod 5개 Running
- Loki gateway Running
- Loki StatefulSet `loki-0` 최종 `2/2 Running`
- Loki cache Pod도 Running

AIOps:

- `aiops-holmes` Pod `1/1 Running`
- Service: `aiops-holmes`
- Service port: `80 -> 5050`
- Port-forward로 `/readyz` 확인

확인 명령:

```bash
kubectl -n aiops port-forward svc/aiops-holmes 18081:80
curl http://127.0.0.1:18081/readyz
```

결과:

```json
{"status":"ready","models":["gpt-5","gpt-5.4"]}
```

발표에서 말할 수 있는 내용:

> HolmesGPT는 Kubernetes 내부 서비스로 배포되어 있고, 확인 시점 기준 ready 상태였다. Alertmanager, Kubernetes 상태, Loki 로그와 연결해 장애 분석을 보조하는 AIOps 컴포넌트로 설명할 수 있다.

## 9. Monitoring 상태

확인된 Running 컴포넌트:

- `alertmanager-monitoring-alertmanager-0`: `2/2 Running`
- `prometheus-monitoring-prometheus-0`: `2/2 Running`
- `monitoring-kube-state-metrics`: `1/1 Running`
- `monitoring-operator`: `1/1 Running`
- `monitoring-prometheus-node-exporter`: 전체 node에 배포되어 Running

주의:

- `monitoring-grafana`: 확인 시점 `2/3 Running`
- Grafana Service endpoint 없음
- Grafana Ingress HTTP 503

추가 확인:

```bash
kubectl top nodes
kubectl top pods -n flaskapp-prod
```

결과:

- 대부분 node metric 확인 가능
- `k8s-worker-5`는 `kubectl top nodes`에서 `<unknown>`으로 표시됨
- FlaskApp Pod metric은 정상 표시됨

발표에서 말할 수 있는 내용:

> Prometheus, Alertmanager, node-exporter, kube-state-metrics는 Running이었다. 다만 Grafana는 확인 시점에 readiness가 완료되지 않아 외부 UI 접속은 미완료 상태였다.

## 10. Ceph RBD CSI 상태

확인 결과:

- StorageClass: `ceph-rbd-team3`
- Provisioner: `rbd.csi.ceph.com`
- ReclaimPolicy: `Retain`
- VolumeExpansion: `true`
- Ceph CSI nodeplugin/provisioner Pod Running
- Grafana PVC가 `ceph-rbd-team3`로 Bound됨

확인된 PVC:

| Namespace | PVC | Status | Size | StorageClass |
| --- | --- | --- | --- | --- |
| `monitoring` | `monitoring-grafana` | Bound | 5Gi | `ceph-rbd-team3` |

발표에서 말할 수 있는 내용:

> Kubernetes에서도 Ceph RBD CSI가 구성되어 있으며, `ceph-rbd-team3` StorageClass로 PVC provisioning이 가능함을 확인했다.

## 11. AWS DR / Terraform 확인 범위

Bastion에서 확인한 제한:

- `aws` CLI가 설치되어 있지 않았다.
- Bastion에는 Proxmox VM용 Terraform 디렉터리만 있었다.
- AWS DR Terraform stack은 로컬 repo `infra/aws/terraform/envs/dr`에서 확인했다.

로컬 repo에서 확인한 AWS DR 구성:

- `module.network`: VPC, subnet, NAT, IGW, VPN, S3 endpoint
- `module.sg`: ALB/EKS/RDS/DMS Security Group
- `module.rds`: DR RDS MariaDB
- `module.dms`: On-prem MariaDB -> RDS Full Load + CDC
- `module.s3`: FlaskApp proddata bucket
- `module.ecr`: FlaskApp image repository
- `module.eks`: `dr_active=true`일 때만 EKS 생성
- `module.alb_ingress`: `dr_active=true`일 때만 AWS Load Balancer Controller 설치
- `module.argocd`, `module.external_secrets`, `module.metrics_server`, `module.cluster_autoscaler`: DR EKS 운영 컴포넌트
- `module.observability`: CloudWatch/SNS/DMS/RDS monitoring

DR 실행 스크립트 확인:

- `infra/aws/scripts/dr-execute/dr-control.sh`
- `infra/aws/scripts/dr-execute/dr-precheck.sh`
- `infra/aws/scripts/dr-execute/dr-up-argocd-local.sh`
- `infra/aws/scripts/dr-execute/dr-verify-app-traffic.sh`
- `infra/aws/scripts/dr-execute/dr-down-local.sh`

발표에서 말할 때 주의:

> AWS DR 구조와 Runbook 스크립트는 코드로 확인했다. 다만 bastion 환경에는 AWS CLI가 없어 현재 AWS 리소스의 실시간 상태는 bastion에서 직접 검증하지 못했다.

## 12. 발표 내용 수정 포인트

기존 초안에서 고쳐야 할 부분:

- FlaskApp Host는 `flaskapp.onprem.local`이 아니라 `flaskapp.team.snow.internal`로 말한다.
- Grafana Host는 `grafana.onprem.local`이 아니라 `grafana.team.snow.internal`로 말한다.
- FlaskApp은 실제 접속 200 OK 확인 완료라고 말해도 된다.
- Grafana는 UI 정상 접속까지 확인했다고 말하면 안 된다. 확인 시점에는 503이었다.
- AIOps/HolmesGPT는 배포 및 ready 상태 확인 완료라고 말해도 된다.
- Monitoring은 Prometheus/Alertmanager/exporter 중심으로 Running 확인, Grafana UI는 준비 중이라고 분리해서 말한다.
- AWS DR은 Terraform/IaC와 runbook 중심으로 설명하고, 실제 AWS 콘솔 상태는 별도 캡처나 AWS CLI 환경에서 추가 확인이 필요하다고 둔다.

## 13. 발표용 정정 멘트

현재 시스템 상태를 발표에서 안전하게 말하면 다음과 같다.

> 온프레미스 Kubernetes 클러스터는 control plane 3대와 worker 5대가 모두 Ready 상태였고, FlaskApp은 ArgoCD 기준 Healthy 상태로 2개 replica가 Running 중이었습니다. HAProxy/Ingress 경로를 통해 `flaskapp.team.snow.internal` Host로 HTTP 200 응답을 확인했고, 외부 MariaDB VM과 DB exporter도 접근 가능했습니다. Monitoring stack은 Prometheus, Alertmanager, node-exporter, kube-state-metrics가 Running이었으며, Grafana는 확인 시점에 readiness가 완료되지 않아 UI 접속은 추가 확인이 필요했습니다. AIOps의 HolmesGPT는 ready 상태로 확인했습니다.

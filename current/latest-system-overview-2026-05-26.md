# FlaskApp On-prem + AWS DR 최신 시스템 정리

확인 시각: 2026-05-26 19:03 KST  
AWS DMS 재확인: 2026-05-26 19:08 KST  
작성 기준: repo 문서/코드 확인 + bastion 실측 + AWS CLI 실측  
AWS 계정: `080252689380`  
AWS 리전: `ap-northeast-2`

## 1. 프로젝트 한 줄 요약

이 프로젝트는 5대 PC 기반 Proxmox 온프렘 인프라 위에 Kubernetes 운영 환경을 구성하고, FlaskApp을 GitOps 방식으로 배포하며, 장애 상황에서는 AWS의 Pilot Light DR 환경으로 복구할 수 있도록 설계한 하이브리드 인프라 프로젝트이다.

핵심은 단순히 FlaskApp을 실행하는 것이 아니라 다음 운영 요소를 함께 구성했다는 점이다.

- Proxmox 기반 온프렘 VM 인프라
- pfSense/VLAN 기반 네트워크 분리
- kubeadm 기반 Kubernetes HA control plane
- HAProxy/Keepalived VIP + NGINX Ingress 외부 진입 경로
- Kubernetes 외부 MariaDB VM
- Helm/ArgoCD 기반 GitOps 배포
- Ceph RBD CSI 기반 Kubernetes PVC
- Prometheus/Grafana/Alertmanager 모니터링
- Loki/Alloy 로깅
- HolmesGPT 기반 AIOps 분석 보조
- AWS VPC/VPN/RDS/DMS/S3/ECR 기반 Pilot Light DR

## 2. 전체 큰그림

```text
사용자
  -> pfSense / VLAN / 방화벽
  -> HAProxy + Keepalived VIP: 172.16.42.99
  -> NGINX Ingress Controller
  -> FlaskApp Service
  -> FlaskApp Pod 2개
  -> MariaDB VM: 172.16.43.160

운영/배포:
GitHub infra repo
  -> ArgoCD root-app
  -> Helm charts
  -> Kubernetes resources

관측/분석:
Prometheus + Grafana + Alertmanager
Loki + Alloy
HolmesGPT

스토리지:
Ceph RBD
  -> Ceph CSI RBD
  -> Kubernetes StorageClass/PVC

DR:
On-prem MariaDB
  -> pfSense/VPN
  -> AWS DMS
  -> AWS RDS
장애 시 Terraform dr_active=true
  -> EKS/ALB/DR FlaskApp 활성화
```

## 3. 현재 상태 요약

### On-prem 실측 상태

2026-05-26 19:03 KST 기준 bastion에서 확인한 상태이다.

| 영역 | 현재 상태 |
| --- | --- |
| Bastion | `172.16.44.100` 접속 가능 |
| Kubernetes API | `172.16.43.99:6443` context 사용 |
| Kubernetes Node | 8대 모두 `Ready` |
| ArgoCD Applications | `root-app`, `flaskapp`, `ingress-nginx`, `ceph-csi-rbd`, `monitoring`, `logging`, `aiops` 모두 `Synced / Healthy` |
| FlaskApp | Deployment `2/2`, `/info` HTTP 200, `/` HTTP 200 |
| MariaDB | `172.16.43.160:3306` 접속 성공 |
| DB Exporter | `172.16.43.160:9100`, `172.16.43.160:9104` 접속 성공 |
| Grafana | `grafana.team.snow.internal` 경로 HTTP 302 확인 |
| Monitoring | Prometheus, Grafana, Alertmanager, kube-state-metrics, node-exporter Running |
| Logging | Loki, Alloy, Loki gateway/cache/canary Running |
| AIOps | HolmesGPT Pod Running |
| Ceph CSI | `ceph-rbd-team3` StorageClass 존재, Grafana PVC Bound |

### AWS 실측 상태

2026-05-26 19:00 KST 이후 AWS CLI로 확인한 상태이다.

| 영역 | 현재 상태 |
| --- | --- |
| VPC | `flaskapp-dr-vpc`, `vpc-012db63a21da35d74`, `10.20.0.0/16`, available |
| Subnet | public/app/data 각 2개, `ap-northeast-2a`, `ap-northeast-2c` 분산 |
| NAT Gateway | `nat-08537bd5c6f96afee`, available, public IP `15.164.83.144` |
| VPN | `vpn-02820049cf4de6764`, available, route `172.16.0.0/16`, tunnel 2개 모두 `UP` |
| RDS | `flaskapp-dr-mariadb`, MariaDB `10.11.16`, `db.t4g.small`, 20GB, private, available |
| DMS Instance | `flaskapp-dr-dms`, `dms.t3.small`, available |
| DMS Task | `flaskapp-dr-full-load-cdc`, `full-load-and-cdc`, 현재 `running` |
| DMS Table Statistics | `flaskapp.employee`, `Table completed`, full load rows 3 |
| EKS | 현재 cluster 목록 없음. DR 앱 실행 계층은 꺼져 있거나 미생성 상태 |
| ALB | 현재 ALB 목록 없음 |
| ECR | `flaskapp`, immutable tag, scan on push enabled |
| ECR 최신 이미지 | `18b68fe`, pushed `2026-05-23 18:23:19 KST` |
| S3 | `flaskapp-proddata-kosa-project-team3-snow-lai9z`, `ap-northeast-2`, versioning enabled, public access block enabled |
| CloudWatch Alarm | `flaskapp-dr-rds-storage-low` OK, `flaskapp-dr-dms-cdc-lag` INSUFFICIENT_DATA |

## 4. 온프렘 인프라 계층

### 4.1 Proxmox / VM 계층

온프렘의 기반은 5대 PC로 구성한 Proxmox VE 클러스터이다. Proxmox는 Kubernetes 노드, DB, Bastion, LB, DNS, 운영 VM을 올리는 가상화 계층이다.

Ansible inventory 기준 주요 VM은 다음과 같다.

| VM | IP | VLAN | Proxmox 배치 | 역할 |
| --- | --- | --- | --- | --- |
| `bastion` | `172.16.44.100` | 40 | `kosa-team3-01` | 운영자 접속, kubectl/ansible 작업 |
| `dns` | `172.16.41.53` | 10 | `kosa-team3-02` | 내부 DNS |
| `k8s-cp-1` | `172.16.43.100` | 30 | `kosa-team3-01` | Kubernetes control plane |
| `k8s-cp-2` | `172.16.43.101` | 30 | `kosa-team3-02` | Kubernetes control plane |
| `k8s-cp-3` | `172.16.43.102` | 30 | `kosa-team3-03` | Kubernetes control plane |
| `k8s-worker-1` | `172.16.43.110` | 30 | `kosa-team3-01` | Kubernetes worker |
| `k8s-worker-2` | `172.16.43.111` | 30 | `kosa-team3-02` | Kubernetes worker |
| `k8s-worker-3` | `172.16.43.112` | 30 | `kosa-team3-03` | Kubernetes worker |
| `k8s-worker-4` | `172.16.43.113` | 30 | `kosa-team3-04` | Kubernetes worker |
| `k8s-worker-5` | `172.16.43.114` | 30 | `kosa-team3-05` | Kubernetes worker, monitoring 전용 label |
| `lb-1` | `172.16.42.100` | 20 | `kosa-team3-04` | HAProxy/Keepalived |
| `lb-2` | `172.16.42.101` | 20 | `kosa-team3-05` | HAProxy/Keepalived |
| `mariadb-1` | `172.16.43.160` | 30 | `kosa-team3-05` | On-prem Primary DB |
| `monitoring` | `172.16.44.101` | 40 | `kosa-team3-04` | 운영/모니터링 접근용 VM |
| `argocd` | `172.16.44.102` | 40 | `kosa-team3-03` | ArgoCD 관리 접근용 VM |

### 4.2 네트워크 계층

최신 기준으로 온프렘 네트워크는 역할별 VLAN으로 나뉜다.

| 대역 | 역할 |
| --- | --- |
| `172.16.30.0/24` | Proxmox 물리 관리망 |
| `172.16.41.0/24` | Public/DNS 영역 |
| `172.16.42.0/24` | DMZ, HAProxy/Keepalived VIP |
| `172.16.43.0/24` | Kubernetes node, MariaDB 내부망 |
| `172.16.44.0/24` | Bastion/Admin/운영 접근망 |
| `10.10.10.0/24` | Ceph storage network |

pfSense는 물리 방화벽 대신 라우터, NAT, 방화벽, VLAN gateway 역할을 맡는다. 외부 요청은 직접 Pod로 들어가지 않고 DMZ VIP와 Ingress를 거쳐 FlaskApp에 도달한다.

## 5. Kubernetes 계층

Kubernetes는 kubeadm 기반 HA control plane 구조이다.

| Node | Role | IP | 확인 상태 |
| --- | --- | --- | --- |
| `k8s-cp-1` | control-plane | `172.16.43.100` | Ready |
| `k8s-cp-2` | control-plane | `172.16.43.101` | Ready |
| `k8s-cp-3` | control-plane | `172.16.43.102` | Ready |
| `k8s-worker-1` | worker | `172.16.43.110` | Ready |
| `k8s-worker-2` | worker | `172.16.43.111` | Ready |
| `k8s-worker-3` | worker | `172.16.43.112` | Ready |
| `k8s-worker-4` | worker | `172.16.43.113` | Ready |
| `k8s-worker-5` | worker | `172.16.43.114` | Ready |

Kubernetes API endpoint는 kube-vip 기반 VIP `172.16.43.99:6443`을 사용한다. CNI는 Calico를 사용하며, 주요 namespace는 다음과 같다.

| Namespace | 역할 |
| --- | --- |
| `flaskapp-prod` | FlaskApp 운영 |
| `ingress-nginx` | NGINX Ingress Controller |
| `argocd` | GitOps 배포 |
| `monitoring` | Prometheus/Grafana/Alertmanager |
| `logging` | Loki/Alloy |
| `aiops` | HolmesGPT |
| `ceph-csi-rbd` | Ceph RBD CSI |
| `calico-system`, `calico-apiserver`, `tigera-operator` | Calico CNI |

## 6. 서비스 진입 경로

최신 사용자 진입 경로는 다음과 같다.

```text
사용자
  -> HAProxy/Keepalived VIP 172.16.42.99
  -> ingress-nginx-controller NodePort
  -> NGINX Ingress
  -> flaskapp-service ClusterIP
  -> FlaskApp Pod
```

현재 확인된 host는 다음과 같다.

| 서비스 | Host | 경로 | 현재 확인 |
| --- | --- | --- | --- |
| FlaskApp | `flaskapp.team.snow.internal` | `172.16.42.99` -> NGINX Ingress -> `flaskapp-service` | `/info` 200, `/` 200 |
| Grafana | `grafana.team.snow.internal` | `172.16.42.99` -> NGINX Ingress -> `monitoring-grafana` | 302 |

오래된 문서에는 `flaskapp.onprem.local`, `grafana.onprem.local`, MetalLB 기반 진입점 등이 남아 있다. 현재 실측 및 Helm values 기준으로는 `flaskapp.team.snow.internal`, `grafana.team.snow.internal`, HAProxy/Keepalived VIP + NGINX Ingress 경로가 최신이다.

## 7. FlaskApp 애플리케이션

FlaskApp은 ECR 이미지와 Helm chart로 배포된다.

| 항목 | 값 |
| --- | --- |
| Namespace | `flaskapp-prod` |
| Deployment | `flaskapp` |
| Replica | 2 |
| Image | `080252689380.dkr.ecr.ap-northeast-2.amazonaws.com/flaskapp:18b68fe` |
| Service | `flaskapp-service`, ClusterIP, port 80 |
| Ingress class | `nginx` |
| Host | `flaskapp.team.snow.internal` |
| DB host | `172.16.43.160` |
| DB name/user | `flaskapp` / `flaskapp` |
| S3 bucket | `flaskapp-proddata-kosa-project-team3-snow-lai9z` |

운영 안정성을 위해 다음 Kubernetes 정책이 적용되어 있다.

| 정책 | 현재 값 / 역할 |
| --- | --- |
| startup/readiness/liveness probe | `/info` |
| HPA | min 2, max 4, CPU 70% |
| PDB | `minAvailable: 1` |
| ServiceAccount | `flaskapp-sa`, token automount false |
| NetworkPolicy | ingress-nginx -> FlaskApp, DNS, DB `172.16.43.160:3306`, HTTPS egress 허용 |
| imagePullSecrets | `ecr-regcred` |
| ECR Secret Refresh | 6시간마다 ECR pull secret 갱신 |

## 8. 데이터베이스

운영 DB는 Kubernetes 내부 DB Pod가 아니라 외부 VM `mariadb-1`이다.

| 항목 | 값 |
| --- | --- |
| VM | `mariadb-1` |
| IP | `172.16.43.160` |
| VLAN | 30 Internal |
| 역할 | On-prem Primary DB |
| 앱 연결 | FlaskApp ConfigMap/Secret 기준 |
| exporter | node_exporter `9100`, mysqld_exporter `9104` |
| 현재 확인 | 3306, 9100, 9104 접속 성공 |

이 구조의 의미는 앱 Pod와 DB 생명주기를 분리하고, AWS DMS가 온프렘 DB의 binlog를 읽어 RDS로 복제할 수 있게 하는 것이다.

## 9. GitOps / IaC

### 9.1 ArgoCD

ArgoCD는 infra repo를 기준으로 Helm chart를 동기화한다.

```text
root-app
  -> argocd/apps
    -> flaskapp
    -> ingress-nginx
    -> ceph-csi-rbd
    -> monitoring
    -> logging
    -> aiops
```

현재 모든 ArgoCD Application은 `Synced / Healthy`이다.

### 9.2 Helm

Helm chart는 다음 배포 단위를 관리한다.

| Chart | 역할 |
| --- | --- |
| `infra/helm/flaskapp` | FlaskApp, Service, Ingress, HPA, PDB, NetworkPolicy, RBAC |
| `infra/helm/monitoring` | kube-prometheus-stack, Grafana, Alertmanager |
| `infra/helm/logging` | Loki, Alloy |
| `infra/helm/aiops` | HolmesGPT, Prometheus/Loki/Kubernetes toolset |

### 9.3 Terraform / Ansible

| 도구 | 사용 영역 |
| --- | --- |
| Terraform on-prem | Proxmox VM 생성 계획/자동화 |
| Ansible on-prem | Kubernetes node 준비, kubeadm 구성, LB, DNS, exporter 등 |
| Terraform AWS | VPC, VPN, RDS, DMS, ECR, S3, EKS, ALB Controller, 관측 리소스 |

## 10. Ceph / Kubernetes Storage

Ceph는 온프렘 기본 스토리지 계층이다. Kubernetes에서는 Ceph RBD CSI를 통해 PVC를 사용할 수 있다.

현재 확인된 Kubernetes StorageClass:

| 항목 | 값 |
| --- | --- |
| StorageClass | `ceph-rbd-team3` |
| Provisioner | `rbd.csi.ceph.com` |
| ReclaimPolicy | `Retain` |
| VolumeExpansion | true |

현재 Bound PVC:

| Namespace | PVC | Size | StorageClass |
| --- | --- | --- | --- |
| `monitoring` | `monitoring-grafana` | 5Gi | `ceph-rbd-team3` |

Released PV도 남아 있으므로, 발표나 운영 정리에서는 “Ceph CSI provisioning 가능, Grafana PVC Bound 확인”까지만 최신 사실로 말하는 것이 안전하다.

## 11. Monitoring / Logging / AIOps

### 11.1 Monitoring

`monitoring` namespace에는 다음이 배포되어 있다.

| 컴포넌트 | 역할 | 현재 상태 |
| --- | --- | --- |
| Prometheus | 메트릭 수집 | Running |
| Grafana | 대시보드 | Running, Ingress 302 |
| Alertmanager | 알림 라우팅 | Running |
| node-exporter | Kubernetes node metrics | 모든 노드 Running |
| kube-state-metrics | Kubernetes object metrics | Running |

Prometheus는 Kubernetes 내부뿐 아니라 MariaDB VM의 exporter도 정적 scrape target으로 수집하도록 구성되어 있다.

### 11.2 Logging

`logging` namespace에는 다음이 배포되어 있다.

| 컴포넌트 | 역할 | 현재 상태 |
| --- | --- | --- |
| Alloy | Pod 로그와 Kubernetes event 수집 | DaemonSet Running |
| Loki | 로그 저장/조회 | Running |
| Loki gateway/cache/canary | Loki 접근 및 검증 보조 | Running |

Loki storage는 Helm values 기준 Ceph RGW S3 endpoint `http://10.10.10.11:7480`와 bucket `team3-loki-chunks`, `team3-loki-ruler`, `team3-loki-admin`을 바라본다.

### 11.3 AIOps

`aiops` namespace에는 HolmesGPT가 배포되어 있다.

HolmesGPT는 다음 정보를 read-only로 활용해 장애 분석을 보조한다.

- Kubernetes resources/events/logs
- Prometheus metrics
- Loki logs
- Alertmanager alert

현재 HolmesGPT Pod는 Running이다. 이 도구는 DR을 자동 실행하는 도구가 아니라, 운영자가 장애 범위와 원인을 빠르게 판단하도록 돕는 분석 보조 도구로 보는 것이 정확하다.

## 12. AWS DR 최신 상태

AWS DR은 Pilot Light 방식이다. 평시에는 네트워크, DB, 복제, 이미지/파일 저장소 같은 기반을 유지하고, 장애 시 EKS/ALB 앱 실행 계층을 켜는 구조이다.

### 12.1 네트워크

| 항목 | 현재 값 |
| --- | --- |
| VPC | `flaskapp-dr-vpc`, `vpc-012db63a21da35d74` |
| CIDR | `10.20.0.0/16` |
| Public subnet | `10.20.0.0/24`, `10.20.1.0/24` |
| App subnet | `10.20.10.0/24`, `10.20.11.0/24` |
| Data subnet | `10.20.20.0/24`, `10.20.21.0/24` |
| NAT Gateway | `nat-08537bd5c6f96afee`, available |
| VPN | `vpn-02820049cf4de6764`, available, tunnel 2개 UP |

현재 VPN connection 리소스는 존재하고 tunnel 2개 모두 UP이다. 따라서 AWS와 온프렘 간 IPsec 경로는 재확인 시점 기준 활성 상태이다.

### 12.2 RDS / DMS

| 항목 | 현재 값 |
| --- | --- |
| RDS | `flaskapp-dr-mariadb` |
| Engine | MariaDB `10.11.16` |
| Instance class | `db.t4g.small` |
| Storage | 20GB |
| Public access | false |
| Status | available |
| Endpoint | `flaskapp-dr-mariadb.cfy4qsq4kamy.ap-northeast-2.rds.amazonaws.com` |
| DMS instance | `flaskapp-dr-dms`, `dms.t3.small`, available |
| DMS task | `flaskapp-dr-full-load-cdc`, `full-load-and-cdc`, running |
| DMS table statistics | `flaskapp.employee`, `Table completed`, full load rows 3 |

현재 DMS task는 `running`이다. Table statistics 기준 `flaskapp.employee` 테이블은 `Table completed` 상태이며 full load rows는 3으로 확인되었다. Validation은 `Not enabled` 상태이므로, 데이터 정합성 검증을 별도 검증 쿼리로 확인하면 더 안전하다.

### 12.3 S3 / ECR

| 항목 | 현재 값 |
| --- | --- |
| S3 bucket | `flaskapp-proddata-kosa-project-team3-snow-lai9z` |
| Region | `ap-northeast-2` |
| Versioning | enabled |
| Public access block | enabled |
| ECR repository | `flaskapp` |
| ECR URI | `080252689380.dkr.ecr.ap-northeast-2.amazonaws.com/flaskapp` |
| ECR tag mutability | immutable |
| ECR scan on push | enabled |
| Latest referenced tag | `18b68fe` |

FlaskApp 온프렘 배포도 이 ECR 이미지 `18b68fe`를 사용한다.

### 12.4 EKS / ALB

현재 AWS EKS cluster 목록은 비어 있고, ALB도 조회되지 않았다. 이는 Terraform 설계의 `dr_active=false` 상태와 맞는다.

| 상태 | 의미 |
| --- | --- |
| EKS 없음 | DR 앱 실행 계층이 현재 꺼져 있거나 미생성 |
| ALB 없음 | AWS 쪽 사용자 진입점이 현재 활성화되지 않음 |
| RDS/DMS/VPC/S3/ECR 존재 | Pilot Light 기반은 존재 |

### 12.5 CloudWatch

| Alarm | Namespace | State | 의미 |
| --- | --- | --- | --- |
| `flaskapp-dr-rds-storage-low` | AWS/RDS | OK | RDS 저장공간 알람 정상 |
| `flaskapp-dr-dms-cdc-lag` | AWS/DMS | INSUFFICIENT_DATA | DMS task 재시작 직후 metric 부족 가능 |

## 13. 설계 변경/오래된 문서와 최신 기준 차이

repo에는 여러 시점의 설계 문서가 함께 남아 있다. 최신 기준으로 볼 때 주의할 차이는 다음과 같다.

| 항목 | 오래된 문서에서 보이는 내용 | 최신 기준 |
| --- | --- | --- |
| FlaskApp host | `flaskapp.onprem.local` | `flaskapp.team.snow.internal` |
| Grafana host | `grafana.onprem.local` | `grafana.team.snow.internal` |
| 외부 진입 | MetalLB 직접 노출 또는 NodePort 테스트 | HAProxy/Keepalived VIP `172.16.42.99` -> NGINX Ingress |
| FlaskApp Service | NodePort 언급 | FlaskApp Service는 ClusterIP, NodePort는 ingress-nginx controller |
| Kubernetes worker 수 | 초기 3대 계획 | 현재 worker 5대 |
| Monitoring 상태 | Grafana 503였던 시점 있음 | 현재 Grafana Running, Ingress 302 |
| Logging/AIOps 상태 | Progressing였던 시점 있음 | 현재 Healthy/Running |
| AWS DMS | 복제 구조 설계/실행 기록 | 현재 task는 running, `flaskapp.employee` full load completed |
| AWS EKS/ALB | 장애 시 활성화 | 현재는 미생성/꺼짐 |

## 14. 지금 기준으로 말할 수 있는 것과 조심할 것

### 말해도 되는 것

- 온프렘 Kubernetes는 control plane 3대, worker 5대의 HA 구조로 구성되어 있고 현재 8대 모두 Ready이다.
- FlaskApp은 ArgoCD/Helm으로 배포되며, 현재 2 replica Running이다.
- 사용자 진입은 HAProxy/Keepalived VIP `172.16.42.99`와 NGINX Ingress를 거친다.
- FlaskApp `/info`와 `/`는 현재 HTTP 200이다.
- MariaDB는 Kubernetes 외부 VM `172.16.43.160`에서 운영되며 현재 3306 접속이 가능하다.
- Grafana, Prometheus, Alertmanager, Loki, Alloy, HolmesGPT는 현재 Running/Healthy이다.
- Ceph RBD CSI StorageClass `ceph-rbd-team3`가 있고 Grafana PVC가 Bound되어 있다.
- AWS에는 VPC, subnet, NAT, VPN, RDS, DMS instance, S3, ECR, CloudWatch alarm이 존재한다.
- AWS VPN tunnel 2개는 현재 UP이고, DMS task는 running 상태이다.
- DMS table statistics 기준 `flaskapp.employee` 테이블 full load가 완료되었고 3 rows가 확인되었다.
- AWS EKS/ALB는 현재 없으며, 이는 DR 앱 계층이 평시에는 꺼지는 Pilot Light 설계와 맞다.

### 조심해야 할 것

- “DMS 데이터 검증까지 완료됐다”고 말하면 안 된다. 현재 task는 running이고 table load는 완료됐지만 DMS validation은 `Not enabled`이다.
- “AWS DR 앱이 지금 서비스 중”이라고 말하면 안 된다. 현재 EKS/ALB가 없다.
- “AWS DR 서비스가 지금 바로 ALB로 접속 가능”하다고 말하면 안 된다. 현재 EKS/ALB가 없다.
- 오래된 host인 `flaskapp.onprem.local`, `grafana.onprem.local`을 최신 발표/문서 기준으로 쓰면 혼선이 생긴다.
- MetalLB가 최종 사용자 진입점인 것처럼 설명하면 현재 구조와 다르다.

## 15. 장애/복구 관점 이해

이 시스템에서 장애를 볼 때는 다음 계층으로 나눠야 한다.

| 계층 | 장애 예시 | 영향 |
| --- | --- | --- |
| Proxmox/물리 | 특정 pve 노드 네트워크 장애 | 해당 노드 위 VM/Pod/DB/monitoring 영향 |
| 네트워크 | pfSense/VLAN/LB/VIP 장애 | 사용자 진입 또는 내부 통신 장애 |
| Kubernetes | Node NotReady, Pod Crash | 앱 replica 감소, monitoring/logging 영향 |
| DB | `mariadb-1` 장애 | FlaskApp 메인 기능 장애 |
| GitOps | ArgoCD sync 실패 | 신규 배포/복구 지연 |
| Storage | Ceph RBD/CSI 장애 | PVC 사용하는 Grafana 등 영향 |
| AWS DR | VPN/DMS/RDS/EKS 장애 | DR 전환 가능성 또는 RPO 영향 |

이번 확인 중 pve-5 네트워크가 잠깐 나갔을 때 `k8s-worker-5`, MariaDB, monitoring/logging/AIOps에 영향이 있었다. 복구 후 node와 앱, DB, monitoring/logging/AIOps가 정상으로 돌아왔다. 이것은 pve-5에 DB와 monitoring 전용 worker가 함께 걸려 있기 때문에, 물리 노드 장애가 DB와 관측 계층에 동시에 영향을 줄 수 있음을 보여준다.

## 16. 다음 점검 우선순위

최신 상태 기준으로 다음 우선순위가 높다.

1. DMS 복제 정합성 확인: RDS에서 `flaskapp.employee` 데이터 row count와 샘플 데이터를 온프렘 MariaDB와 비교한다.
2. DMS CDC 지속 동작 확인: 온프렘 DB에 테스트 row를 추가/수정한 뒤 RDS 반영 여부와 latency metric을 확인한다.
3. CloudWatch DMS CDC lag 알람 확인: task 재시작 후 metric이 들어와 `INSUFFICIENT_DATA`가 해소되는지 본다.
4. pve-5 장애 시 영향 완화: DB VM, monitoring node, LB-2가 pve-5에 몰린 구조의 리스크 검토.
5. DR 전환 리허설: `dr_active=true`로 EKS/ALB 활성화 가능 여부 재검증.
6. Grafana/Loki/HolmesGPT 대시보드/질의 시연 자료 고정.

## 17. 참고한 주요 파일

- `Docs/architecture/시스템-아키텍쳐-설계-최종본.md`
- `Docs/aws/aws-architecture-final.md`
- `Docs/presentation/final-project-topic-structure.md`
- `Docs/presentation/bastion-verified-system-status.md`
- `infra/onprem/ansible/inventories/onprem/hosts.yml`
- `infra/helm/flaskapp/values.yaml`
- `infra/helm/monitoring/values.yaml`
- `infra/helm/logging/values.yaml`
- `infra/helm/aiops/values.yaml`
- `infra/argocd/root-app.yaml`
- `infra/argocd/apps/*.yaml`
- `infra/aws/terraform/envs/dr/main.tf`
- `infra/aws/terraform/envs/dr/variables.tf`
- `infra/aws/scripts/dr-execute/dr-control.sh`

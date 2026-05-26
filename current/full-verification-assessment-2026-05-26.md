# FlaskApp On-prem + AWS DR 구역별 검증 및 평가

확인 시각: 2026-05-26 19:20 KST  
확인 방식: repo 문서/코드 확인 + bastion 실측 + AWS CLI 실측  
범위: On-prem 네트워크/DNS/LB, Kubernetes, FlaskApp GitOps/CI-CD, DB/스토리지, 모니터링/로깅/AIOps, AWS DR

## 1. 최상위 결론

현재 시스템은 온프렘 운영 경로가 정상 동작하고, AWS DR의 기반 자원도 대부분 준비되어 있다. 다만 AWS DR 앱 실행 계층은 Pilot Light 설계대로 현재 꺼져 있으며, DMS 복제는 running이지만 validation은 활성화되어 있지 않다.

가장 중요한 현재 판단은 다음과 같다.

| 구역 | 평가 |
| --- | --- |
| On-prem DNS/LB/Ingress | 정상. Bind9가 FlaskApp/Grafana host를 VIP로 해석하고, HAProxy/Keepalived VIP가 Ingress로 전달한다. |
| On-prem Kubernetes | 정상. 3 control plane + 5 worker 모두 Ready, Calico/Ingress/kube-vip 정상. |
| FlaskApp | 정상. ArgoCD/Helm 배포 Healthy, `/info`와 `/` 모두 200. |
| DB | 정상. MariaDB VM, 3306, node_exporter, mysqld_exporter 모두 접근 가능. |
| Ceph/K8s Storage | 부분 정상. Ceph RBD CSI와 Grafana PVC Bound 확인. Released PV 잔여물은 정리 포인트. |
| Monitoring/Logging/AIOps | 정상. Prometheus/Grafana/Alertmanager/Loki/Alloy/HolmesGPT Running. |
| Monitoring VM | 별도 Grafana/Prometheus/Loki 서버는 확인되지 않음. 실제 관측 stack은 Kubernetes 내부에서 동작. |
| AWS Network/DR Base | 정상. VPC/Subnet/NAT/VPN/RDS/DMS/S3/ECR 존재, VPN tunnel 2개 UP. |
| AWS DMS | 동작 중. Source/target connection successful, task running, `flaskapp.employee` full load completed. Validation은 Not enabled. |
| AWS EKS/ALB | 현재 없음. `dr_active=false`의 Pilot Light 상태로 판단 가능. |

## 2. On-prem 네트워크, DNS, LB

### 설계상 의도

On-prem은 VLAN을 역할별로 나누고, pfSense가 라우터/방화벽/VLAN gateway 역할을 맡는다. 외부 사용자는 직접 Kubernetes Pod에 접근하지 않고 다음 경로로 들어온다.

```text
사용자
  -> Bind9 DNS
  -> HAProxy/Keepalived VIP 172.16.42.99
  -> ingress-nginx NodePort
  -> NGINX Ingress
  -> FlaskApp Service
  -> FlaskApp Pod
```

VLAN 역할은 다음과 같다.

| 대역 | 역할 |
| --- | --- |
| `172.16.30.0/24` | Proxmox 물리 관리망 |
| `172.16.41.0/24` | Public/DNS |
| `172.16.42.0/24` | DMZ/LB/VIP |
| `172.16.43.0/24` | Kubernetes node, DB 내부망 |
| `172.16.44.0/24` | Bastion/Admin/운영 접근 |

### 실제 구현/검증

Bind9 DNS 확인:

| Host | 결과 |
| --- | --- |
| `flaskapp.team.snow.internal` | `172.16.42.99` |
| `grafana.team.snow.internal` | `172.16.42.99` |
| `db.team.snow.internal` | Bind9 기준 NXDOMAIN |

LB/VIP 확인:

| 대상 | 결과 |
| --- | --- |
| `172.16.41.53:53` | 접속 성공 |
| `172.16.42.99:80` | 접속 성공 |
| `172.16.42.100:80` | 접속 성공 |
| `172.16.42.101:80` | 접속 성공 |
| `flaskapp.team.snow.internal` via VIP | `/info` 200 |
| `grafana.team.snow.internal` via VIP | 302 |

`lb-1`에서 확인한 상태:

- `eth0`: `172.16.42.100/24`, `172.16.42.99/24`
- `haproxy`: active
- `keepalived`: active
- HAProxy listen: `0.0.0.0:80`, `0.0.0.0:443`

### 평가

설계 의도와 실제 구현이 잘 맞는다. 현재 VIP는 `lb-1`이 들고 있고, HAProxy/Keepalived 경로가 살아 있다. FlaskApp과 Grafana 모두 동일 VIP를 사용하고 Host header/Ingress로 분기된다.

주의할 점:

- `db.team.snow.internal`은 현재 Bind9에 없다. DB는 DNS 이름이 아니라 `172.16.43.160` IP 기준으로 연결된다.
- `lb-2`는 80 포트 접근은 확인됐지만 SSH 상세 점검은 지연되어 중단했다. HAProxy/Keepalived 백업 노드 상태는 별도 확인하면 좋다.
- 오래된 문서의 `flaskapp.onprem.local`, `grafana.onprem.local`, MetalLB 직접 노출 설명은 최신 구조와 다르다.

## 3. On-prem Kubernetes

### 설계상 의도

Kubernetes는 kubeadm 기반 HA control plane으로 구성하고, 서비스 실행은 worker node에서 담당한다. Control plane 접근은 kube-vip 기반 API VIP로 안정화한다. Pod network는 Calico가 담당한다.

### 실제 구현/검증

Kubernetes node:

| Node | Role | IP | Status |
| --- | --- | --- | --- |
| `k8s-cp-1` | control-plane | `172.16.43.100` | Ready |
| `k8s-cp-2` | control-plane | `172.16.43.101` | Ready |
| `k8s-cp-3` | control-plane | `172.16.43.102` | Ready |
| `k8s-worker-1` | worker | `172.16.43.110` | Ready |
| `k8s-worker-2` | worker | `172.16.43.111` | Ready |
| `k8s-worker-3` | worker | `172.16.43.112` | Ready |
| `k8s-worker-4` | worker | `172.16.43.113` | Ready |
| `k8s-worker-5` | worker | `172.16.43.114` | Ready |

Control plane 구성:

- `kube-apiserver`, `etcd`, `controller-manager`, `scheduler`가 3 control plane에서 Running
- `kube-vip`가 3 control plane에서 Running
- Kubernetes API endpoints: `172.16.43.100:6443`, `172.16.43.101:6443`, `172.16.43.102:6443`
- kubectl context server: `172.16.43.99:6443`

Calico:

- `calico-node`가 모든 node에서 Running
- `calico-typha` 3개 Running
- `calico-apiserver` 2개 Running
- Calico CSI node driver도 각 node에 배포됨

Ingress:

- `ingress-nginx-controller` Service type: NodePort
- HTTP NodePort: `30080`
- HTTPS NodePort: `30443`
- FlaskApp Ingress host: `flaskapp.team.snow.internal`
- Grafana Ingress host: `grafana.team.snow.internal`

### 평가

Kubernetes 핵심 구조는 정상이다. 특히 “FlaskApp Service를 NodePort로 노출”한 것이 아니라 “Ingress Controller를 NodePort로 노출하고 HAProxy가 이 NodePort로 전달”하는 구조가 최신 기준이다.

리스크/주의:

- `k8s-worker-5`는 monitoring 성격의 workload가 몰려 있다. 앞서 pve-5 네트워크가 나갔을 때 monitoring/logging/AIOps가 같이 흔들렸으므로, 단일 노드 집중 리스크가 있다.
- Control plane 컴포넌트는 Running이지만 restart count가 많은 항목이 있다. 발표에서는 현재 Running/Ready 중심으로 말하고, 운영적으로는 재시작 원인 분석 여지를 남겨두면 좋다.

## 4. FlaskApp, CI/CD, ArgoCD

### 설계상 의도

FlaskApp은 Docker 이미지로 빌드되어 AWS ECR에 저장되고, infra repo의 Helm values가 이미지 태그를 참조한다. ArgoCD는 infra repo를 감시하며 Helm chart를 Kubernetes에 동기화한다.

```text
FlaskApp code
  -> Docker build
  -> ECR push
  -> infra Helm values image tag update
  -> ArgoCD sync
  -> Kubernetes rollout
```

### 실제 구현/검증

ArgoCD Application:

| App | Sync | Health | Revision |
| --- | --- | --- | --- |
| `root-app` | Synced | Healthy | `7a8ad62...` |
| `flaskapp` | Synced | Healthy | `7a8ad62...` |
| `ingress-nginx` | Synced | Healthy | `4.11.3` |
| `ceph-csi-rbd` | Synced | Healthy | `3.16.2` |
| `monitoring` | Synced | Healthy | `7a8ad62...` |
| `logging` | Synced | Healthy | `7a8ad62...` |
| `aiops` | Synced | Healthy | `7a8ad62...` |

FlaskApp runtime:

| 항목 | 값 |
| --- | --- |
| Namespace | `flaskapp-prod` |
| Deployment | `flaskapp` |
| Replica | `2/2` |
| Image | `080252689380.dkr.ecr.ap-northeast-2.amazonaws.com/flaskapp:18b68fe` |
| Service | ClusterIP `10.109.124.18`, port 80 |
| Ingress | `flaskapp.team.snow.internal` |
| HPA | min 2, max 4, CPU target 70% |
| PDB | minAvailable 1 |
| Probe | `/info` startup/readiness/liveness |
| NetworkPolicy | ingress-nginx, DNS, DB, HTTPS egress 허용 |

검증:

- VIP 경유 `/info`: 200
- VIP 경유 `/`: 200
- FlaskApp logs: `/info`, `/` 요청 200 확인

ECR secret refresh:

- CronJob: `flaskapp-ecr-secret-refresh`, `0 */6 * * *`
- 최근 Job: Complete
- 과거 Failed Job 3개 잔존

### 평가

FlaskApp 배포와 운영은 정상이다. ArgoCD/Helm/GitOps 구조가 실제 cluster에 반영되어 있고, 앱은 DB까지 연결되어 `/`도 200이다.

주의할 점:

- ECR secret refresh는 현재 성공했지만 과거 실패 Job이 남아 있다. “완전히 문제 없었다”보다 “현재 최신 실행은 성공, 이전 실패 이력은 있음”이 정확하다.
- CI/CD 전체 GitHub Actions 실행 상태까지는 이번 실측에 포함하지 않았다. 현재 Kubernetes와 ECR 기준으로는 배포 결과가 정상임을 확인했다.

## 5. DB, DB 저장소, Ceph, RDS

### 설계상 의도

운영 DB는 Kubernetes 내부 Pod가 아니라 별도 VM에서 운영한다. DB VM은 On-prem Primary이고, AWS RDS는 DR target이다. 온프렘 DB 변경분은 AWS DMS가 읽어 RDS에 반영한다.

```text
FlaskApp Pod
  -> MariaDB VM 172.16.43.160
  -> DMS source endpoint
  -> VPN
  -> AWS DMS
  -> AWS RDS MariaDB
```

### 실제 On-prem DB 검증

MariaDB VM:

| 항목 | 값 |
| --- | --- |
| Hostname | `mariadb-1` |
| IP | `172.16.43.160` |
| SSH | 접속 성공 |
| MariaDB | active, `0.0.0.0:3306` listen |
| node_exporter | active, `:9100` listen |
| mysqld_exporter | active, `:9104` listen |

접속 확인:

- `172.16.43.160:22`: success
- `172.16.43.160:3306`: success
- `172.16.43.160:9100`: success
- `172.16.43.160:9104`: success

### Ceph / Kubernetes storage 검증

Ceph RBD CSI:

- `ceph-csi-rbd-nodeplugin`: worker 1~5에서 Running
- `ceph-csi-rbd-provisioner`: 3개 Running
- StorageClass: `ceph-rbd-team3`
- Provisioner: `rbd.csi.ceph.com`
- ReclaimPolicy: `Retain`
- VolumeExpansion: true

PVC/PV:

| PVC | Namespace | 상태 | 크기 | StorageClass |
| --- | --- | --- | --- | --- |
| `monitoring-grafana` | `monitoring` | Bound | 5Gi | `ceph-rbd-team3` |

잔여 PV:

- `logging/rbd-test` Released 1Gi
- 이전 `monitoring-grafana` Released 5Gi

### AWS RDS / DMS 검증

RDS:

| 항목 | 값 |
| --- | --- |
| DB | `flaskapp-dr-mariadb` |
| Engine | MariaDB `10.11.16` |
| Class | `db.t4g.small` |
| Storage | 20GB |
| Multi-AZ | false |
| Public | false |
| Status | available |

DMS:

| 항목 | 값 |
| --- | --- |
| Source endpoint | `flaskapp-dr-source`, `172.16.43.160:3306`, active |
| Target endpoint | `flaskapp-dr-target`, RDS endpoint, active |
| Source connection | successful |
| Target connection | successful |
| Task | `flaskapp-dr-full-load-cdc` |
| Type | full-load-and-cdc |
| Status | running |
| Table stats | `flaskapp.employee`, Table completed, full load rows 3 |
| Validation | Not enabled |

### 평가

DB 경로는 현재 정상이다. 온프렘 앱이 DB에 붙고 있고, AWS DMS도 source/target connection이 successful이며 task가 running이다.

주의할 점:

- DMS validation은 `Not enabled`라 데이터 정합성 검증은 별도로 해야 한다.
- RDS는 Single-AZ, 20GB, private 구성이다. MVP/DR 실습으로는 적절하지만 운영 고가용성 관점에서는 Multi-AZ가 아니다.
- Ceph RBD CSI는 정상이나 Released PV 잔여물이 있어 스토리지 정리 작업이 필요할 수 있다.

## 6. Monitoring, Logging, AIOps

### 설계상 의도

Kubernetes 내부에는 Prometheus/Grafana/Alertmanager를 구성하고, DB VM 같은 외부 대상도 exporter로 수집한다. 로그는 Alloy가 수집해 Loki에 저장하고, HolmesGPT는 Prometheus/Loki/Kubernetes 정보를 활용해 장애 분석을 보조한다.

### Kubernetes 내부 모니터링 검증

`monitoring` namespace:

| 컴포넌트 | 상태 | 비고 |
| --- | --- | --- |
| Prometheus | Running | `/ready` OK |
| Grafana | Running | VIP 경유 302 |
| Alertmanager | Running | Pod 2/2 |
| kube-state-metrics | Running | Kubernetes object metrics |
| node-exporter | Running | 모든 node 배포 |
| Prometheus Operator | Running | CR 관리 |

Prometheus active targets에서 확인:

- `mariadb-vm-node-exporter`: `172.16.43.160:9100`, health `up`
- `mariadb-vm-mysqld-exporter`: `172.16.43.160:9104`, health `up`

### Logging 검증

`logging` namespace:

| 컴포넌트 | 상태 |
| --- | --- |
| Alloy DaemonSet | worker 1~5 Running |
| Loki | Running |
| Loki gateway | Running |
| Loki cache/canary | Running |

Loki API:

- `GET /loki/api/v1/labels`: success
- 확인 label: `namespace`, `pod`, `container`, `app`, `job` 등

참고: `loki-gateway`의 root/ready 경로는 404/connection refused가 나올 수 있으나, 실제 Loki API 경로는 정상 응답했다.

### Monitoring VM 검증

`monitoring` VM `172.16.44.101`:

- SSH 접속 성공
- IP: `172.16.44.101`
- Grafana/Prometheus/Loki/Alertmanager가 VM systemd service로 별도 실행되는 흔적은 이번 확인에서 발견되지 않음

따라서 최신 기준으로는 “monitoring VM이 별도 모니터링 서버를 직접 운영한다”기보다, 실제 모니터링 stack은 Kubernetes 내부 `monitoring`/`logging` namespace에서 운영된다고 보는 것이 정확하다.

### AIOps / HolmesGPT 검증

`aiops` namespace:

- HolmesGPT Pod Running
- `/readyz` 응답: `{"status":"ready","models":["gpt-5","gpt-5.4"]}`

HolmesGPT toolset 설계:

- Kubernetes core/resources
- Kubernetes logs
- Prometheus metrics
- Grafana/Loki logs
- Alertmanager URL 연동

### 평가

관측/AIOps는 현재 정상이다. Prometheus가 내부 Kubernetes뿐 아니라 외부 DB VM exporter까지 수집하고 있는 점이 좋다. Loki API도 정상이고 HolmesGPT ready 상태도 확인됐다.

주의할 점:

- monitoring/logging/AIOps 주요 workload가 `k8s-worker-5`에 많이 몰려 있다. pve-5 장애 시 관측 계층이 흔들릴 수 있다.
- Monitoring VM은 이름과 달리 독립 실행형 Grafana/Prometheus 서버가 아니라, 현재는 Kubernetes 내부 관측 stack 접근/운영 맥락으로 보는 것이 맞다.

## 7. AWS 구역별 상세 검증

### 7.1 AWS 네트워크

설계상 AWS는 `10.20.0.0/16` VPC를 만들고 public/app/data subnet을 2개 AZ에 나눈다. 온프렘 `172.16.0.0/16`은 VPN/VGW로 라우팅한다.

검증 결과:

| 항목 | 값 |
| --- | --- |
| VPC | `flaskapp-dr-vpc`, `vpc-012db63a21da35d74`, available |
| CIDR | `10.20.0.0/16` |
| Public subnet | `10.20.0.0/24`, `10.20.1.0/24` |
| App subnet | `10.20.10.0/24`, `10.20.11.0/24` |
| Data subnet | `10.20.20.0/24`, `10.20.21.0/24` |
| NAT Gateway | `nat-08537bd5c6f96afee`, available |
| VPN | `vpn-02820049cf4de6764`, tunnel 1/2 모두 UP |

라우팅:

- public subnet: `0.0.0.0/0` -> IGW
- app subnet: `0.0.0.0/0` -> NAT, `172.16.0.0/16` -> VGW
- data subnet: `172.16.0.0/16` -> VGW
- S3 endpoint route 존재

평가:

- 설계대로 subnet 분리와 VPN 라우팅이 구성되어 있다.
- VPN tunnel 2개가 UP이라 AWS-온프렘 경로는 현재 정상이다.

### 7.2 AWS 보안그룹

확인된 주요 SG:

| SG | 역할 | 평가 |
| --- | --- | --- |
| `flaskapp-dr-alb-sg` | 80/443 public ingress | DR ALB 생성 시 사용자 진입 허용 |
| `flaskapp-dr-eks-node-sg` | ALB -> EKS node, cluster -> node | DR EKS용 사전 SG |
| `flaskapp-dr-cluster-sg` | EKS control plane | DR EKS용 사전 SG |
| `flaskapp-dr-dms-sg` | DMS egress 3306 | 온프렘/RDS DB 연결용 |
| `flaskapp-dr-rds-sg` | RDS MariaDB | 현재 DMS SG에서 3306 허용 |

주의:

- 현재 조회 기준 RDS SG ingress는 DMS SG만 보인다. DR EKS 앱이 RDS에 직접 접속할 때 EKS node SG가 RDS 3306에 허용되는지 재확인이 필요하다.
- 코드상 `sg` 모듈은 DR active와 EKS node 접근을 고려하지만, 실제 현재 SG 상태가 앱 전환 시 충분한지는 리허설로 검증해야 한다.

### 7.3 AWS RDS / DMS

설계상 RDS는 DR target DB이고, DMS는 On-prem MariaDB -> RDS Full Load + CDC를 수행한다.

검증:

- RDS available
- DMS instance available
- DMS source/target endpoint active
- DMS source/target connection successful
- DMS task running
- `flaskapp.employee` full load completed, 3 rows
- CDC checkpoint 존재

평가:

- 현재 DMS는 다시 정상 가동 중이다.
- 단, validation이 꺼져 있으므로 “복제 task가 running이고 테이블 load는 완료”라고 말해야 한다.
- “데이터 정합성 검증 완료”라고 말하려면 온프렘/RDS row count와 샘플 데이터를 직접 비교해야 한다.

### 7.4 AWS S3 / ECR

S3:

- Bucket: `flaskapp-proddata-kosa-project-team3-snow-lai9z`
- Versioning: Enabled
- PublicAccessBlock: all true

ECR:

- Repository: `flaskapp`
- URI: `080252689380.dkr.ecr.ap-northeast-2.amazonaws.com/flaskapp`
- Tag mutability: IMMUTABLE
- Scan on push: true
- 온프렘 FlaskApp image tag: `18b68fe`

평가:

- S3/ECR은 설계대로 상시 기반 자원으로 존재한다.
- ECR immutable tag와 scan on push는 이미지 추적성 측면에서 좋다.

### 7.5 AWS EKS / ALB

설계상 EKS/ALB는 `dr_active=true`일 때만 켜지는 앱 실행 계층이다.

검증:

- EKS cluster 목록 없음
- ALB 목록 없음

평가:

- 현재는 Pilot Light의 평시 상태와 맞다.
- AWS로 실제 서비스가 전환되어 있다고 말하면 안 된다.
- DR 전환 가능성을 입증하려면 `dr_active=true` 리허설과 ALB -> FlaskApp -> RDS/S3 경로 검증이 필요하다.

### 7.6 AWS CloudWatch

CloudWatch alarms:

| Alarm | State | 평가 |
| --- | --- | --- |
| `flaskapp-dr-rds-storage-low` | OK | RDS 저장공간 알람 정상 |
| `flaskapp-dr-dms-cdc-lag` | INSUFFICIENT_DATA | DMS 재시작 직후 metric 부족 가능 |

평가:

- CloudWatch alarm 리소스는 존재한다.
- DMS lag 알람은 아직 정상/경보 판단을 할 enough datapoints가 없다.

## 8. 전체적으로 잘된 점

- 온프렘 운영 경로가 실제로 동작한다.
- DNS -> VIP -> HAProxy -> Ingress -> Service -> Pod -> DB 흐름이 검증됐다.
- Kubernetes HA control plane, Calico, Ingress, HPA, PDB, NetworkPolicy가 실제 적용되어 있다.
- ArgoCD 기반 GitOps 상태가 전체 Healthy다.
- MariaDB VM을 Kubernetes 외부로 분리했고 exporter까지 붙였다.
- Prometheus가 DB VM exporter를 실제 `up` target으로 수집한다.
- Loki API와 HolmesGPT ready 상태가 확인됐다.
- AWS DR 기반 자원과 DMS 복제 task가 현재 살아 있다.

## 9. 우회/변경된 점

- 오래된 설계의 MetalLB 중심 외부 노출 대신, 최신 구조는 HAProxy/Keepalived VIP + ingress-nginx NodePort를 사용한다.
- FlaskApp Service 자체는 NodePort가 아니라 ClusterIP다.
- DB는 DNS 이름이 아니라 IP `172.16.43.160` 기준으로 앱에 연결된다.
- Monitoring VM에서 직접 Grafana/Prometheus/Loki를 운영하기보다, Kubernetes 내부 monitoring/logging namespace에서 운영한다.
- AWS EKS/ALB는 상시 운영이 아니라 Pilot Light 방식으로 꺼져 있다.

## 10. 문제/리스크

| 항목 | 리스크 | 권장 조치 |
| --- | --- | --- |
| `db.team.snow.internal` 없음 | DB DNS 기반 운영 설명 불가 | Bind9에 DB record 추가하거나 IP 기반이라고 명확히 문서화 |
| `lb-2` 상세 확인 미완료 | LB 이중화 검증 부족 | `lb-2` SSH/systemctl/keepalived priority 상태 확인 |
| pve-5 의존도 | DB, LB-2, worker-5/monitoring workload 영향 | DB/monitoring workload 분산 또는 장애 시나리오 명확화 |
| ECR refresh 과거 실패 Job | 이미지 pull secret 갱신 안정성 이력 혼재 | 실패 원인 정리, 완료 Job 기준으로 history 정리 |
| Released PV 잔여 | Ceph RBD 정리 필요 가능 | 불필요 PV/RBD image 정리 기준 마련 |
| DMS validation off | 복제 정합성 확정 불가 | 온프렘/RDS row count 및 샘플 비교 |
| RDS SG | DR EKS -> RDS 접속 허용 여부 재검증 필요 | `dr_active=true` 리허설 또는 SG rule 추가 확인 |
| EKS/ALB 없음 | AWS DR 앱 경로 현재 미검증 | DR 전환 리허설 수행 |
| DMS lag alarm | INSUFFICIENT_DATA | task running 후 metric 수집 안정화 확인 |

## 11. 발표/설명 시 안전한 표현

안전하게 말할 수 있는 표현:

- “온프렘은 현재 Kubernetes 기반으로 FlaskApp이 운영 중이며, VIP/Ingress 경로로 200 응답을 확인했습니다.”
- “DB는 Kubernetes 내부가 아니라 별도 MariaDB VM으로 분리했고, 앱과 Prometheus exporter 접근을 확인했습니다.”
- “GitOps는 ArgoCD root-app 구조로 관리하며 현재 주요 Application은 Synced/Healthy입니다.”
- “Prometheus/Grafana/Loki/HolmesGPT는 Kubernetes 내부에 배포되어 있고, DB VM exporter도 Prometheus target으로 수집 중입니다.”
- “AWS는 Pilot Light DR 기반으로 VPC/VPN/RDS/DMS/S3/ECR을 유지하고 있으며, DMS task는 현재 running입니다.”
- “EKS/ALB는 평시에는 꺼져 있고, 장애 전환 시 Terraform으로 활성화하는 구조입니다.”

피해야 할 표현:

- “AWS DR 서비스가 지금 ALB로 운영 중입니다.” 현재 EKS/ALB가 없다.
- “DMS 데이터 정합성 검증까지 완료했습니다.” validation은 Not enabled다.
- “Monitoring VM에서 Grafana/Prometheus/Loki가 직접 실행됩니다.” 현재 확인 기준 실제 stack은 Kubernetes 내부다.
- “DB DNS가 구성되어 있습니다.” `db.team.snow.internal`은 Bind9 기준 NXDOMAIN이다.

## 12. 다음 실전 검증 순서

1. `lb-2`의 HAProxy/Keepalived 상태를 직접 확인한다.
2. DMS 정합성 검증: 온프렘 MariaDB와 RDS의 `flaskapp.employee` row count와 샘플 데이터를 비교한다.
3. CDC 검증: 온프렘 DB에 테스트 row 추가/수정 후 RDS 반영 여부를 본다.
4. CloudWatch DMS lag alarm이 `OK` 또는 정상 datapoint 상태로 바뀌는지 확인한다.
5. `dr_active=true`로 EKS/ALB 리허설을 수행한다.
6. AWS DR FlaskApp이 RDS와 S3에 실제 접근 가능한지 검증한다.

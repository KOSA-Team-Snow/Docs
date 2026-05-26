# 최종 발표 대주제별 정리

## 발표 전체 메시지

우리 프로젝트는 온프레미스에서 FlaskApp을 운영하되, 장애가 발생했을 때 AWS로 복구할 수 있는 하이브리드 DR 아키텍처를 설계하고 구현한 프로젝트이다.

핵심 설계 방향은 다음과 같다.

> 평소에는 온프레미스를 Primary로 운영하고, AWS에는 데이터와 핵심 기반만 유지하는 Pilot Light DR 방식을 적용해 비용을 줄이면서도 복구 가능한 구조를 만든다.

발표는 기능 나열이 아니라 다음 질문에 답하는 흐름으로 가져간다.

- 왜 이런 아키텍처를 선택했는가?
- 장애 시 데이터는 어떻게 보호되는가?
- 장애 상황은 어떻게 판단하는가?
- 인프라는 어떻게 반복 가능하게 관리하는가?
- 실제로 복구할 수 있음을 어떻게 검증했는가?

---

## 1. 설계 이유와 DR 아키텍처

### 발표 핵심

이 프로젝트의 출발점은 단순 배포가 아니라 "서비스 장애 시 복구 가능한 운영 구조"를 만드는 것이다. 온프레미스가 기본 운영 환경이고, AWS는 장애 시 서비스를 이어받는 DR 환경이다.

### 우리 프로젝트에 맞는 내용

- 기본 서비스는 On-prem Proxmox/Kubernetes에서 운영한다.
- 온프렘에는 5대 PC 기반 Proxmox VE 클러스터를 구성했다.
- Kubernetes 위에서 FlaskApp을 운영하고, 외부 진입은 HAProxy/Keepalived VIP와 Ingress를 통해 처리한다.
- DB는 Kubernetes 내부가 아니라 별도 MariaDB VM에서 운영한다.
- AWS에는 VPC, VPN, RDS, DMS, S3, ECR, Terraform 기반 DR 환경을 준비한다.
- 장애 시 EKS/ALB를 활성화해 FlaskApp을 AWS에서 실행할 수 있도록 설계했다.

### Pilot Light 방식을 선택한 이유

Pilot Light는 평소에는 꼭 필요한 기반만 켜두고, 장애가 발생했을 때 애플리케이션 실행 계층을 켜는 방식이다.

우리 프로젝트에서 평소 유지하는 자원:

- AWS VPC/Subnet/Security Group
- Site-to-Site VPN
- RDS
- DMS
- S3
- ECR
- CloudWatch/SNS 등 관측 자원

장애 시 활성화하는 자원:

- EKS Cluster
- EKS Node Group
- AWS Load Balancer Controller
- ALB
- FlaskApp on EKS

선택 이유:

- Active-Active보다 비용이 낮다.
- 15일 프로젝트 범위에서 구현과 설명이 현실적이다.
- 데이터 복제와 네트워크 연결은 평소에 유지해 복구 기반을 준비할 수 있다.
- 장애 시 `dr_active=true` 같은 Terraform 변수로 앱 계층을 활성화하는 구조를 만들 수 있다.

### 발표용 한 문장

> 저희는 온프렘을 Primary 운영 환경으로 두고, AWS에는 데이터와 복구 기반만 유지하는 Pilot Light DR 방식을 선택했습니다. 이를 통해 비용을 줄이면서도 장애 시 EKS와 ALB를 활성화해 서비스를 복구할 수 있는 구조를 설계했습니다.

### 추천 슬라이드

- 전체 아키텍처 다이어그램
- 평시와 장애 시 켜지는 자원 비교표
- Pilot Light 선택 이유 비교표

---

## 2. On-prem DB, DMS CDC, RDS 복제, Site-to-Site VPN, pfSense

### 발표 핵심

DR에서 가장 중요한 것은 애플리케이션보다 데이터이다. 온프렘 DB의 변경분을 AWS RDS로 복제하기 위해 AWS DMS CDC를 사용하고, 온프렘과 AWS 간 통신은 Site-to-Site VPN으로 연결한다.

### 우리 프로젝트에 맞는 내용

On-prem DB:

- DB는 Kubernetes 내부가 아닌 `mariadb-1` 외부 VM에서 운영한다.
- DB IP는 `172.16.43.160`이다.
- DB VM은 VLAN 30 Internal에 배치했다.
- DB 데이터 디스크는 Ceph RBD를 사용한다.
- FlaskApp Pod는 환경변수와 Kubernetes Secret을 통해 DB에 접속한다.

AWS 복제 구조:

- AWS RDS는 DR용 Replica 역할이다.
- AWS DMS가 On-prem MariaDB의 binlog를 읽어 RDS에 변경분을 적용한다.
- 복제 방식은 Full Load + CDC이다.
- Full Load는 기존 데이터를 한 번 복사하고, CDC는 이후 변경분을 계속 반영한다.

VPN과 pfSense:

- 온프렘과 AWS VPC는 Site-to-Site VPN으로 연결한다.
- AWS VPC 대역은 `10.20.0.0/16`이다.
- 온프렘 대역은 `172.16.0.0/16` 기준으로 라우팅한다.
- 물리 방화벽 장비가 없기 때문에 `pve-1` 위 pfSense VM이 라우터, 방화벽, NAT, VLAN gateway 역할을 담당한다.
- pfSense에서 AWS VPN 터널과 온프렘 VLAN 간 통신 경로를 열어줘야 DMS가 MariaDB에 접근할 수 있다.

### 이 구조에서 중요한 포인트

- 터널이 UP이어도 방화벽 정책이 막혀 있으면 DMS는 DB에 접근할 수 없다.
- AWS 라우팅, pfSense 방화벽, DB 서버 UFW, MariaDB 계정 Host 권한이 모두 맞아야 한다.
- DMS 전용 계정은 최소 권한으로 구성해야 한다.
- 정상 운영 중에는 On-prem MariaDB가 Primary이고, AWS RDS는 DR 대기 상태이다.
- 장애 전환 시에는 AWS RDS를 쓰기 가능한 DB로 사용하는 절차가 필요하다.

### 발표용 한 문장

> DR에서 핵심은 데이터이기 때문에 On-prem MariaDB의 binlog를 AWS DMS CDC로 읽어 RDS에 복제했습니다. 이 복제 트래픽은 pfSense를 통해 구성한 Site-to-Site VPN 경로를 지나며, 라우팅과 방화벽 정책까지 함께 설계해야 실제 복제가 동작합니다.

### 추천 슬라이드

- On-prem MariaDB → pfSense → VPN → DMS → RDS 흐름도
- Full Load + CDC 설명
- 통신이 성립하기 위한 체크포인트 표

---

## 3. Prometheus/Grafana 모니터링과 AIOps 기반 DR 판단

### 발표 핵심

DR은 자동으로 켜는 것보다 "언제 DR을 선언할 것인가"가 중요하다. 이를 위해 Prometheus/Grafana로 인프라와 애플리케이션 상태를 관측하고, Alertmanager/Loki/HolmesGPT를 연결해 장애 원인 분석과 DR 판단을 보조한다.

### 우리 프로젝트에 맞는 내용

모니터링 구성:

- Prometheus, Grafana, Alertmanager는 Kubernetes 내부 `monitoring` namespace에 배포한다.
- node-exporter로 Kubernetes 노드 상태를 수집한다.
- kube-state-metrics로 Deployment, Pod, Node 등 Kubernetes object 상태를 수집한다.
- DB VM에는 node_exporter와 mysqld_exporter를 설치해 외부 MariaDB 상태도 수집한다.
- Grafana는 Kubernetes 노드, Pod, DB 상태를 시각화한다.

로그/AIOps 구성:

- Loki는 Kubernetes 로그 저장소 역할을 한다.
- Alloy가 Kubernetes 로그를 수집해 Loki로 전달한다.
- Alertmanager가 장애 알림을 이메일로 보낸다.
- HolmesGPT는 alert 정보, Kubernetes 상태, Loki 로그를 바탕으로 원인 분석을 보조한다.

DR 판단에서 보는 항목:

- FlaskApp Pod replica가 0인지
- Ingress/VIP 경로가 응답하는지 (`flaskapp.team.snow.internal` 기준)
- Kubernetes Node가 NotReady인지
- MariaDB 접속이 가능한지
- DMS replication lag가 큰지
- VPN 터널이 살아 있는지
- 장애가 애플리케이션 단일 장애인지, 온프렘 전체 장애인지

### AIOps 역할

HolmesGPT는 DR을 자동 실행하는 도구가 아니라, 운영자가 빠르게 상황을 판단하도록 돕는 분석 도구이다.

예시:

- FlaskApp replica unavailable 알림 발생
- HolmesGPT에 alertname, namespace, service, description을 전달
- HolmesGPT가 Deployment, Pod, Event, Log를 함께 보고 원인을 요약
- 운영자는 단순 앱 오류인지, 노드 장애인지, DR 전환이 필요한지 판단

### 발표용 한 문장

> DR은 버튼을 누르는 것보다 먼저 장애 범위를 정확히 판단하는 것이 중요합니다. 그래서 Prometheus/Grafana로 상태를 관측하고, Alertmanager와 Loki, HolmesGPT를 연결해 장애가 앱 문제인지, 노드 문제인지, 온프렘 전체 장애인지 판단할 수 있도록 설계했습니다.

### 추천 슬라이드

- Monitoring namespace 구성도
- Alertmanager → HolmesGPT 분석 흐름
- DR 판단 기준 체크리스트

---

## 4. GitOps, Terraform, IaC

### 발표 핵심

운영 환경은 수동으로 만들면 재현이 어렵고 장애 복구도 느리다. 그래서 Kubernetes 배포는 GitOps로, 인프라 생성은 Terraform과 Ansible로 관리했다.

### 우리 프로젝트에 맞는 내용

GitOps:

- FlaskApp 코드는 GitHub Actions를 통해 Docker 이미지로 빌드된다.
- 이미지는 AWS ECR에 push된다.
- CD workflow가 infra repo의 Helm values image tag를 갱신한다.
- ArgoCD가 Git 변경을 감지하고 Kubernetes에 자동 반영한다.
- Helm chart는 Deployment, Service, Ingress, ConfigMap, Secret, HPA, PDB, RBAC, NetworkPolicy를 관리한다.

Terraform:

- On-prem VM 생성은 Proxmox Terraform으로 관리한다.
- AWS DR 인프라는 Terraform으로 관리한다.
- VPC, Subnet, Security Group, RDS, DMS, ECR, S3, EKS, ALB 관련 자원을 코드로 정의한다.
- `dr_active=false`일 때는 평시 Pilot Light 자원만 유지한다.
- `dr_active=true`일 때는 EKS/ALB 등 앱 실행 계층을 활성화한다.

Ansible:

- Kubernetes 노드 OS 설정, containerd, kubeadm 설치를 자동화한다.
- kubeadm init/join, kube-vip, Calico 구성 지원에 사용한다.
- DB VM exporter와 monitoring external probe 설정에도 사용한다.

### 왜 IaC가 중요한가

- 같은 인프라를 반복해서 만들 수 있다.
- 변경 이력이 Git에 남는다.
- 실수로 수동 변경한 내용을 줄일 수 있다.
- 장애 상황에서 복구 절차를 명령어와 코드 기준으로 실행할 수 있다.
- 발표와 검증에서 "설계만 한 것"이 아니라 "재현 가능한 코드"가 있음을 보여줄 수 있다.

### 발표용 한 문장

> 저희는 Kubernetes 배포는 GitOps로, 인프라 구성은 Terraform과 Ansible로 관리했습니다. 즉 애플리케이션과 인프라 모두 Git을 기준으로 재현 가능하게 만들었고, DR 환경도 Terraform 변수로 평시와 장애 시 상태를 전환할 수 있도록 설계했습니다.

### 추천 슬라이드

- GitHub Actions → ECR → infra values → ArgoCD 흐름도
- Terraform `dr_active=false/true` 비교
- IaC 도구별 책임 분리 표

---

## 5. 테스트, 검증, DR Runbook

### 발표 핵심

DR 아키텍처는 그려놓는 것만으로는 의미가 없다. 실제로 어떤 조건을 확인하고, 어떤 순서로 실행하며, 어디까지 검증했는지를 Runbook으로 정리해야 한다.

### 정상 운영 검증

검증 항목:

- Kubernetes node가 Ready 상태인지 확인
- FlaskApp Pod가 Running이고 readiness가 통과하는지 확인
- Ingress 또는 VIP를 통해 FlaskApp 접속이 가능한지 확인 (`172.16.42.99` + `Host: flaskapp.team.snow.internal`)
- FlaskApp에서 MariaDB insert/select가 가능한지 확인
- ConfigMap/Secret 기반 DB 설정이 Pod에 주입되는지 확인
- S3 bucket 설정이 적용되어 파일 업로드 경로가 준비되었는지 확인
- ArgoCD Application이 Synced/Healthy 상태인지 확인
- Grafana에서 node/pod/db metric이 보이는지 확인

대표 명령:

```bash
kubectl get nodes -o wide
kubectl get pods -n flaskapp-prod
kubectl rollout status deployment/flaskapp -n flaskapp-prod
kubectl get ingress -A
kubectl get applications -n argocd
```

### DR 전환 전 확인

DR 선언 전 확인할 것:

- 장애가 앱 단일 장애인지 온프렘 전체 장애인지 판단
- Grafana/Alertmanager 알림 확인
- HolmesGPT 분석 결과 확인
- VPN 터널 상태 확인
- DMS replication lag 확인
- RDS 데이터 최신성 확인
- On-prem DB write 중단 가능 여부 확인

### DR Runbook 초안

1. 장애 감지
   - Grafana, Alertmanager, external probe, 사용자 접속 실패를 확인한다.

2. 장애 범위 판단
   - FlaskApp Pod 문제인지, Kubernetes Node 문제인지, DB 문제인지, 온프렘 네트워크 문제인지 구분한다.
   - HolmesGPT로 alert와 log를 분석한다.

3. DR 전환 결정
   - 온프렘 복구가 어렵거나 예상 복구 시간이 길면 DR 전환을 선언한다.

4. 데이터 상태 확인
   - DMS replication lag를 확인한다.
   - RDS에 필요한 데이터가 복제되었는지 확인한다.

5. On-prem write 중단
   - 가능하다면 On-prem FlaskApp 또는 DB write를 중단해 split-brain을 방지한다.

6. AWS DR 앱 계층 활성화
   - Terraform으로 `dr_active=true`를 적용한다.

```bash
terraform plan -var="dr_active=true"
terraform apply -var="dr_active=true"
```

7. EKS 배포 확인
   - EKS cluster와 node가 준비되었는지 확인한다.
   - FlaskApp이 RDS/S3 설정으로 실행되는지 확인한다.

8. DNS 전환
   - Route 53 또는 내부 DNS/hosts를 AWS ALB로 전환한다.

9. 서비스 검증
   - AWS ALB로 FlaskApp 접속을 확인한다.
   - DB 조회/쓰기와 S3 업로드를 확인한다.

10. 사후 기록
    - 장애 원인, 전환 시각, DMS lag, RPO/RTO, 수동 조치 내역을 기록한다.

### 검증 결과 발표 방식

단순히 "테스트했습니다"라고 말하지 말고 다음 구조로 보여준다.

| 검증 항목 | 확인 방법 | 성공 기준 |
| --- | --- | --- |
| K8s 상태 | `kubectl get nodes/pods` | Node Ready, Pod Running |
| 앱 접속 | `curl -I http://172.16.42.99 -H 'Host: flaskapp.team.snow.internal'` | HTTP 200 |
| DB 연결 | 앱 insert/select | MariaDB 데이터 반영 |
| GitOps | ArgoCD UI | Synced/Healthy |
| 모니터링 | Grafana | node/pod/db metric 표시 |
| DR 데이터 | DMS/RDS 확인 | RDS에 데이터 복제 |
| DR 앱 | EKS/ALB 접속 | AWS FlaskApp 응답 |

### 발표용 한 문장

> 저희는 DR을 단순 설계로 끝내지 않고, 정상 운영 검증과 장애 전환 Runbook으로 정리했습니다. 장애 감지, 원인 판단, DMS/RDS 데이터 확인, Terraform을 통한 AWS 앱 계층 활성화, DNS 전환, 서비스 검증까지 순서화해 실제 운영자가 따라 할 수 있도록 만들었습니다.

### 추천 슬라이드

- 정상 운영 검증 체크리스트
- DR Runbook 10단계
- 테스트 결과 표

---

## 최종 발표 흐름 추천

| 순서 | 대주제 | 발표 목적 |
| --- | --- | --- |
| 1 | 설계 이유와 DR 아키텍처 | 왜 이 프로젝트를 이렇게 설계했는지 설명 |
| 2 | DB 복제와 VPN | DR의 핵심인 데이터 보호 구조 설명 |
| 3 | 모니터링과 AIOps | DR 전환 판단 근거 설명 |
| 4 | GitOps, Terraform, IaC | 반복 가능한 배포/복구 구조 설명 |
| 5 | 테스트, 검증, Runbook | 실제 운영 가능성과 검증 결과 설명 |

## 최종 결론 멘트

이번 프로젝트는 FlaskApp을 단순히 Kubernetes에 배포하는 것을 넘어서, 온프레미스 운영 환경과 AWS DR 환경을 하나의 서비스 운영 흐름으로 연결한 프로젝트입니다. Pilot Light 방식을 선택해 비용과 복구 가능성의 균형을 맞췄고, DMS CDC와 Site-to-Site VPN으로 데이터를 복제했으며, Prometheus/Grafana/AIOps로 장애 상황을 판단할 수 있게 했습니다. 또한 GitOps와 Terraform 기반 IaC를 적용해 배포와 복구 절차를 코드로 재현 가능하게 만들었고, 마지막으로 Runbook을 통해 DR 상황에서 실제로 어떤 순서로 대응할지 정리했습니다.

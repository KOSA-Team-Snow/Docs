# 🚀 15 Working Days Technical WBS: FlaskApp On-Premise to AWS DR

## 1. 프로젝트 개요

- **기간:** 15 Working Days
- **인원:** 인프라 엔지니어 5명
- **목표:** Proxmox + Kubernetes 기반의 FlaskApp 온프레미스 운영 및 AWS DR(Disaster Recovery) 전환 구조 구현
- **핵심 기술:** Docker, K8s, MetalLB, Ingress, MariaDB, ArgoCD, Helm, Ansible, Terraform, Prometheus, Grafana
- **데이터/스토리지:** On-prem(Primary DB) → AWS RDS(Replica), AWS S3(단일 저장소)

---

## 2. 기술 역할 분배

| 역할  | 담당 분야                     | 핵심 키워드                           |
| :---- | :---------------------------- | :------------------------------------ |
| **A** | On-prem / Proxmox / Network   | Proxmox, VLAN, LACP, MetalLB          |
| **B** | K8s Platform / RBAC / Ingress | kubeadm, HA Control Plane, RBAC       |
| **C** | App / Docker / Helm / ArgoCD  | Docker, Helm, ArgoCD, GitOps          |
| **D** | DB / Backup / Monitoring      | MariaDB, CronJob, Prometheus, Grafana |
| **E** | AWS DR / Terraform / EKS      | Terraform, EKS, RDS, S3, Route 53     |

---

## 4. 팀원별 WBS

### 팀원 A - On-prem 인프라 / Proxmox / 네트워크

| Day | 작업                                           | 완료 기준                           |
| --- | ---------------------------------------------- | ----------------------------------- |
| 1   | 물리 PC 5대 역할 정의, IP/VLAN 초안 작성       | IP/VLAN 계획 작성                   |
| 2   | VM 사양 및 배치표 작성                         | VM 10대 배치표 확정                 |
| 3   | Proxmox 클러스터 생성, VM 템플릿 준비          | 5대 Proxmox 클러스터 확인           |
| 4   | VM 생성 및 SSH 키 배포 지원                    | 모든 VM SSH 접속 가능               |
| 5   | Kubernetes 노드 간 네트워크 확인               | control plane/worker 통신 확인      |
| 6   | MetalLB IP pool 설계 및 적용 지원              | LoadBalancer IP 할당 확인           |
| 7   | ArgoCD UI와 Ingress 외부 접근 경로 점검        | 브라우저에서 ArgoCD/Ingress IP 접속 |
| 8   | 네트워크 장애 포인트 정리                      | 장애 대응 체크리스트 작성           |
| 9   | Taint 적용할 infra node 선정 지원              | infra node label/taint 적용         |
| 10  | Grafana/ArgoCD 외부 접근용 DNS 또는 hosts 정리 | 접속 주소 정리                      |
| 11  | AWS VPN/라우팅 및 S3 접근 경로 설계 협업       | On-prem/AWS/S3 통신 흐름 정리       |
| 12  | EKS 전환 시 네트워크 비교표 작성               | On-prem vs AWS 진입 경로 비교       |
| 13  | 장애 시 On-prem 차단 시나리오 준비             | 데모용 장애 절차 정리               |
| 14  | 네트워크 구성도 보강                           | 발표용 다이어그램 완성              |
| 15  | 데모 중 네트워크/접속 담당                     | 정상/DR 접속 확인                   |

### 팀원 B - Kubernetes 플랫폼

| Day | 작업                                                                     | 완료 기준                                                                 |
| --- | ------------------------------------------------------------------------ | ------------------------------------------------------------------------- |
| 1   | Kubernetes 구성 방식 결정                                                | kubeadm 기준 설치 절차 작성                                               |
| 2   | control plane/worker 노드 계획 확정                                      | 노드명, role, label 계획 작성                                             |
| 3   | kubeadm 설치 스크립트 초안 작성                                          | 설치 명령어 정리                                                          |
| 4   | Ansible로 Kubernetes 사전 패키지 설치                                    | container runtime, kubeadm 준비                                           |
| 5   | Kubernetes HA 클러스터 구성                                              | kubectl get nodes 정상                                                    |
| 6   | namespace, RBAC, ServiceAccount 기본 구성, Nginx Ingress Controller 설치 | flaskapp-prod, db, monitoring, argocd, test 생성, Ingress Controller 정상 |
| 7   | ArgoCD 설치 지원 및 bootstrap 권한 검토                                  | ArgoCD pod 정상, root app 적용 가능                                       |
| 8   | ClusterRole/RoleBinding 정리                                             | Prometheus/ArgoCD용 RBAC 적용                                             |
| 9   | requests/limits, HPA 적용                                                | FlaskApp HPA 동작 확인                                                    |
| 10  | Taints/Tolerations 적용                                                  | infra pod만 infra node 배치                                               |
| 11  | BusyBox 테스트 Pod 작성                                                  | DNS, Service, DB 접속 테스트                                              |
| 12  | EKS manifest/Helm values 호환성 점검                                     | On-prem/EKS 공통 chart 배포 가능                                          |
| 13  | 장애 전환 시 Kubernetes 검증 명령어 정리                                 | DR 검증 체크리스트 작성                                                   |
| 14  | Kubernetes 발표 파트 정리                                                | 명령어 캡처, 설명 자료 완성                                               |
| 15  | 데모 중 K8s 상태 확인 담당                                               | pod/service/ingress 상태 확인                                             |

### 팀원 C - Application / Docker / Helm / ArgoCD

| Day | 작업                                                                         | 완료 기준                                                 |
| --- | ---------------------------------------------------------------------------- | --------------------------------------------------------- |
| 1   | FlaskApp 구조 파악                                                           | 필요한 env 목록 정리                                      |
| 2   | Dockerfile 작성                                                              | 로컬 이미지 빌드 성공                                     |
| 3   | FlaskApp 컨테이너 실행 테스트                                                | 컨테이너에서 웹 접속 성공                                 |
| 4   | Kubernetes Deployment/Service 초안 작성                                      | manifest 초안 작성                                        |
| 5   | ConfigMap/Secret 연동                                                        | env 기반 앱 설정 가능, PHOTOS_BUCKET은 AWS S3 bucket 사용 |
| 6   | Ingress 경로 연결 및 Helm chart 초안 작성                                    | / 접속 시 FlaskApp 응답, chart 초안 작성                  |
| 7   | ArgoCD root app/Application 작성, 이미지 태그 전략 및 ECR push 스크립트 작성 | root-app.yaml, flaskapp.yaml, build-push.sh 동작          |
| 8   | Helm chart 완성 및 ArgoCD 배포 전환                                          | Git commit 후 ArgoCD sync로 FlaskApp 배포                 |
| 9   | requests/limits, readiness/liveness probe 추가                               | rollout 안정화                                            |
| 10  | ArgoCD Application 확장 및 sync 정책 정리                                    | monitoring/db-demo Application 추가 또는 설계 반영        |
| 11  | EKS 배포용 values 파일 작성                                                  | onprem/aws values 분리                                    |
| 12  | EKS에 FlaskApp 배포                                                          | AWS ALB 통해 접속 성공                                    |
| 13  | DR 전환 후 앱 설정 검증                                                      | RDS Primary/S3 단일 bucket 연동 확인                      |
| 14  | 앱 배포 문서 및 포트폴리오 설명 작성                                         | README 보강                                               |
| 15  | 데모 중 앱 기능 시연 담당                                                    | 생성/조회/이미지 업로드 시연                              |

### 팀원 D - Database / Backup / Monitoring

RBD

| Day | 작업                                                       | 완료 기준                                                     |
| --- | ---------------------------------------------------------- | ------------------------------------------------------------- |
| 1   | MariaDB 운영 방식 결정                                     | 외부 VM MariaDB Primary + K8s demo DB 여부 결정               |
| 2   | DB 스키마 및 계정 계획 작성                                | DB명, user, 권한 정리                                         |
| 3   | MariaDB VM 설치 절차 작성                                  | 수동 설치 성공                                                |
| 4   | Ansible MariaDB playbook 작성                              | MariaDB 자동 설치 가능                                        |
| 5   | FlaskApp DB 연결 지원                                      | 앱에서 DB 조회 가능                                           |
| 6   | Secret 기반 DB 접속 정보 관리                              | DB password Secret 적용, S3 관련 Secret은 앱 담당과 분리 확인 |
| 7   | Job으로 초기 테이블 생성                                   | schema init Job 성공                                          |
| 8   | CronJob으로 DB 백업 구성                                   | 백업 파일 생성 확인                                           |
| 9   | StatefulSet + Headless Service 데모 구성                   | mariadb-demo namespace 또는 db namespace에서 동작             |
| 10  | Prometheus/Grafana Helm values 작성 및 ArgoCD 배포         | Grafana 접속 가능, monitoring Application sync                |
| 11  | node-exporter DaemonSet 확인                               | 노드 메트릭 수집 확인                                         |
| 12  | FlaskApp/DB 모니터링 대시보드 작성                         | CPU, memory, pod, node 지표 표시                              |
| 13  | AWS DMS CDC/RDS Replica 상태 또는 대체 복제 확인 절차 작성 | replication lag, RDS 승격 검증 문서화                         |
| 14  | 모니터링 발표 화면 준비                                    | Grafana 캡처와 설명 완성                                      |
| 15  | 데모 중 DB/모니터링 담당                                   | DB 쓰기, 백업, 대시보드 시연                                  |

### 팀원 E - AWS DR / Terraform

| Day | 작업                                       | 완료 기준                                        |
| --- | ------------------------------------------ | ------------------------------------------------ |
| 1   | AWS DR 범위 확정, S3 단일 저장소 방침 정리 | 사용할 AWS 리소스 목록 확정                      |
| 2   | Terraform 디렉터리 및 backend 설계         | terraform/aws-dr skeleton 작성                   |
| 3   | VPC/subnet/security group 모듈 작성        | terraform plan 성공                              |
| 4   | S3/ECR 모듈 작성                           | AWS S3 단일 bucket, ECR repository 생성          |
| 5   | RDS 모듈 작성                              | RDS MySQL/MariaDB Replica 대상 생성 가능         |
| 6   | Route 53/ACM/ALB 설계                      | DNS 전환 방식 문서화                             |
| 7   | AWS DMS CDC 또는 대체 복제 방식 검토       | On-prem MariaDB Primary -> RDS Replica 방식 결정 |
| 8   | ECR push 권한 및 이미지 저장소 연동        | FlaskApp 이미지 ECR 저장                         |
| 9   | EKS Terraform 모듈 작성                    | EKS plan 성공                                    |
| 10  | EKS 생성 및 kubectl 연결                   | AWS cluster 접속 가능                            |
| 11  | AWS Load Balancer Controller 구성          | ALB Ingress 생성                                 |
| 12  | EKS에 FlaskApp 배포                        | ALB 주소로 앱 접속, 동일 AWS S3 bucket 사용      |
| 13  | DR failover runbook 작성                   | RDS Replica 승격, EKS, Route 53 전환 절차        |
| 14  | Terraform 발표 자료 정리                   | apply 결과, 리소스 다이어그램                    |
| 15  | 데모 중 AWS DR 담당                        | AWS 전환 시연                                    |

---

## 3. 전체 일정 로드맵 (Daily Goals)

|  Day   | 목표                           |  Day   | 목표                       |
| :----: | :----------------------------- | :----: | :------------------------- |
| **01** | 기술 설계 및 역할 분배         | **09** | HPA, Taint/Toleration 적용 |
| **02** | VM 및 AWS DR 기본 설계         | **10** | 모니터링 및 ArgoCD 확장    |
| **03** | Proxmox 클러스터 및 VM 생성    | **11** | Terraform AWS 1차 구성     |
| **04** | Ansible 자동화 및 K8s 준비     | **12** | EKS 및 DR 배포 경로 구성   |
| **05** | Kubernetes 클러스터 구성       | **13** | 장애 전환 리허설 (Runbook) |
| **06** | MetalLB, Ingress, RBAC         | **14** | 문서화 및 발표 자료 보강   |
| **07** | ArgoCD, FlaskApp, MariaDB 구성 | **15** | 최종 발표 및 데모 시연     |
| **08** | GitOps 기반 배포 (Helm)        |   -    | -                          |

---

## 4. 팀원별 상세 WBS (주요 작업)

### 👨‍💻 팀원별 상세 태스크 요약

- **A (Infra/Network):** 물리 환경 기반 구축, [[VLAN 및 IP 계획]], 장애 시나리오 설계, 네트워크 시각화. **권순호**
- **B (K8s Platform):** `kubeadm` 기반 HA 클러스터 설치, Namespace/RBAC 관리, 인프라 격리. **안지오님**
- **C (App/GitOps):** FlaskApp 컨테이너화, Helm 차트 작성, ArgoCD를 통한 CI/CD 파이프라인. **이민희님**
- **D (DB/Monitoring):** MariaDB 설치 및 백업 전략(CronJob), Prometheus/Grafana 시각화. **정현욱님**
- **E (AWS DR):** Terraform을 이용한 VPC/RDS/EKS 인프라 구축, DR Failover 프로세스. **최재혁님**

---

## 5. Kubernetes 적용 기술 스택

- **클러스터링/배포:** `kubeadm`, `ArgoCD`, `Helm`, `Ansible`
- **앱 구성:** `Dockerfile`, `ConfigMap`, `Secret`, `HPA`, `Requests/Limits`
- **서비스/네트워킹:** `MetalLB`, `Ingress`, `Headless Service`
- **운영/모니터링:** `Prometheus`, `Grafana`, `node-exporter`
- **인프라 운영:** `Job`, `CronJob`, `StatefulSet`, `DaemonSet`, `Taints/Tolerations`

---

## 7. Day 15 최종 데모 시나리오

1. Grafana에서 On-prem Kubernetes 노드와 Pod 상태를 보여준다.
2. Route 53 또는 hosts 기준으로 On-prem FlaskApp에 접속한다.
3. FlaskApp에서 데이터를 생성하고 On-prem MariaDB Primary 저장을 확인한다.
4. 이미지/파일 업로드가 AWS S3 단일 bucket에 저장되는 것을 확인한다.
5. BusyBox Pod로 Service DNS와 DB 포트 연결을 확인한다.
6. HPA 부하 테스트로 FlaskApp Pod가 증가하는 것을 보여준다.
7. ArgoCD에서 FlaskApp/monitoring Application이 Synced 상태인 것을 보여준다.
8. 장애 상황을 선언하고 On-prem 접속을 차단한다.
9. RDS Replica를 Primary로 사용하는 DR 절차를 설명하거나 시연한다.
10. Terraform으로 AWS EKS 또는 준비된 DR 리소스를 활성화한다.
11. EKS에 FlaskApp을 배포하고 RDS/S3 설정으로 실행한다.
12. Route 53을 AWS ALB로 전환한다.
13. 동일한 도메인으로 AWS FlaskApp 접속을 확인한다.
14. 발표에서 RPO/RTO와 한계점을 설명한다.

---

## 7. 구현 우선순위 (가이드)

#### **Must-Have (필수):**

- Proxmox VM 구성
- Kubernetes cluster
- MetalLB
- Ingress
- ArgoCD bootstrap
- FlaskApp Docker/K8s 배포
- MariaDB 연동
- ConfigMap/Secret
- AWS S3 단일 bucket
- On-prem MariaDB Primary -> AWS RDS Replica 복제 설계
- Terraform AWS 기본 리소스
- EKS DR 배포
- Prometheus/Grafana 기본 대시보드

### **Nice-to-Have (심화):**

- HPA 부하 테스트
- Taints/Tolerations
- DB backup CronJob
- Job 기반 schema init
- StatefulSet/Headless Service demo DB
- ServiceMonitor/PrometheusRule
- DMS CDC 실제 구성 및 replication lag 대시보드

### **Out-of-Scope (축소 가능):**

- 직접 Custom Controller 개발 제외
- MariaDB HA 제외
- 완전 자동 failover 제외
- 복잡한 CRD 직접 제작 제외
- Service Mesh 제외
- On-prem S3 호환 스토리지 제외

---

## 8. 역할별 15 Working Days 로드맵

| 역할                            | WD1               | WD2             | WD3              | WD4             | WD5              | WD6                    | WD7                 | WD8              | WD9                   | WD10              | WD11             | WD12                  | WD13               | WD14                | WD15             |
| ------------------------------- | ----------------- | --------------- | ---------------- | --------------- | ---------------- | ---------------------- | ------------------- | ---------------- | --------------------- | ----------------- | ---------------- | --------------------- | ------------------ | ------------------- | ---------------- |
| **A** On-prem / Network         | PC 역할/IP 초안   | VM 배치 확정    | Proxmox 클러스터 | VM/SSH 지원     | K8s 노드망 확인  | MetalLB IP pool        | ArgoCD/Ingress 접근 | 장애 포인트 정리 | infra node taint 지원 | 운영 UI 접근 정리 | AWS/S3 경로 협업 | On-prem/AWS 경로 비교 | 장애 차단 시나리오 | 네트워크 다이어그램 | 접속 데모 담당   |
| **B** Kubernetes Platform       | kubeadm 방식 결정 | 노드 role/label | 설치 스크립트    | K8s 사전 패키지 | HA 클러스터 구성 | namespace/RBAC/Ingress | ArgoCD 설치 지원    | RBAC 정리        | HPA/limits 적용       | taint/toleration  | BusyBox 테스트   | EKS 호환성 점검       | DR 검증 명령어     | K8s 발표 정리       | K8s 상태 담당    |
| **C** Container / Helm / ArgoCD | 앱 실행조건 파악  | Dockerfile      | 컨테이너 테스트  | Deploy/Svc 초안 | ConfigMap/Secret | Ingress/Helm 초안      | ArgoCD App/ECR push | Helm/ArgoCD 배포 | probe/limits          | App 확장 sync     | AWS values 분리  | EKS 앱 배포           | RDS/S3 앱 검증     | 배포 문서 정리      | 앱 기능 시연     |
| **D** DB / Monitoring           | DB 방식 결정      | 스키마/계정     | MariaDB 설치     | Ansible DB      | 앱 DB 연결       | DB Secret              | schema Job          | backup CronJob   | StatefulSet demo      | Prom/Grafana 배포 | node-exporter    | 대시보드 작성         | DMS/RDS 검증       | 모니터링 발표       | DB/모니터링 시연 |
| **E** AWS DR / Terraform        | AWS 범위/S3 방침  | TF skeleton     | VPC/SG 모듈      | S3/ECR 모듈     | RDS 모듈         | DNS/ALB 설계           | DMS CDC 검토        | ECR 연동         | EKS 모듈              | EKS 접속          | ALB Controller   | EKS 앱 검증           | DR runbook         | Terraform 발표      | AWS DR 시연      |

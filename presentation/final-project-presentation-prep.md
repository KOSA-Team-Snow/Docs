# 최종 프로젝트 총정리 및 발표 준비안

## 1. 발표 핵심 메시지

우리 프로젝트는 단순히 Flask 애플리케이션을 Kubernetes에 올린 것이 아니라, 제한된 온프레미스 장비 위에 실제 운영 가능한 서비스 플랫폼을 만들고, 장애 발생 시 AWS DR 환경으로 전환할 수 있는 구조까지 설계하고 구현한 프로젝트이다.

한 문장으로 정리하면 다음과 같다.

> Proxmox/Ceph 기반 온프레미스 인프라에서 FlaskApp을 Kubernetes와 GitOps로 운영하고, 데이터는 AWS로 복제해 장애 시 Pilot Light DR 환경으로 서비스를 복구한다.

발표에서 계속 밀고 가야 할 키워드:

- 온프레미스 운영 환경 구축
- Kubernetes 기반 애플리케이션 플랫폼
- GitOps 기반 배포 자동화
- MariaDB 외부 VM 운영과 AWS RDS DR 복제
- Prometheus/Grafana/Alertmanager/AIOps 기반 운영 관측
- Terraform 기반 AWS Pilot Light DR

## 2. 발표 스토리라인

### 도입

문제의식:

- 서비스는 평소에는 온프레미스에서 운영되어야 한다.
- 장애가 나도 애플리케이션과 데이터가 완전히 사라지면 안 된다.
- 15일 안에 인프라, 애플리케이션 배포, 모니터링, DR까지 보여줘야 한다.

해결 방향:

- 5대 PC를 Proxmox 클러스터로 묶고, Ceph를 VM/DB 디스크 저장소로 사용한다.
- Kubernetes HA 클러스터 위에서 FlaskApp을 컨테이너로 운영한다.
- GitHub Actions, ECR, Helm, ArgoCD로 배포 흐름을 자동화한다.
- DB는 Kubernetes 밖의 MariaDB VM에서 운영하고, AWS DMS CDC로 RDS에 복제한다.
- 장애 시 Terraform으로 AWS EKS/ALB를 활성화하고 Route 53 또는 DNS 전환으로 서비스를 AWS에서 받는다.

### 본론

1. 물리 인프라와 네트워크를 먼저 보여준다.
2. 그 위에 Kubernetes 플랫폼을 어떻게 올렸는지 설명한다.
3. FlaskApp이 어떻게 배포되고 운영되는지 설명한다.
4. DB와 백업/복제 구조를 설명한다.
5. 모니터링과 AIOps가 장애 판단을 어떻게 돕는지 보여준다.
6. 마지막으로 AWS DR 전환 흐름을 시연하거나 설명한다.

### 결론

이 프로젝트의 결과물은 "앱 배포"가 아니라 "운영 가능한 하이브리드 인프라"이다. 자동화, 관측, 복구 전략까지 포함했기 때문에 실제 서비스 운영 관점의 설계와 구현 경험을 보여줄 수 있다.

## 3. 추천 슬라이드 구성

| # | 제목 | 핵심 메시지 | 보여줄 자료 |
| --- | --- | --- | --- |
| 1 | 프로젝트 개요 | 온프렘 운영 + AWS DR을 구현한 프로젝트 | 전체 한 줄 아키텍처 |
| 2 | 요구사항과 목표 | 제한된 장비에서 운영성과 복구성을 확보 | 15일 WBS, 역할 분담 |
| 3 | 전체 아키텍처 | 사용자 요청, 온프렘, AWS DR의 큰 흐름 | `Docs/architecture/시스템-아키텍쳐-설계-최종본.md` 다이어그램 |
| 4 | 물리/네트워크 설계 | 관리망, 서비스망, 스토리지망을 분리 | VLAN/IP 계획표 |
| 5 | Proxmox와 Ceph | 5대 노드 클러스터와 RBD 기반 저장소 | Proxmox/Ceph 상태 캡처 |
| 6 | Kubernetes 플랫폼 | kubeadm HA control plane과 Calico 구성 | `kubectl get nodes -o wide` |
| 7 | 서비스 진입 구조 | HAProxy/Keepalived VIP와 Ingress를 통한 외부 접근 | 요청 흐름도 |
| 8 | FlaskApp 배포 구조 | Docker, Helm, ConfigMap/Secret, probe, HPA | Helm chart 구조 |
| 9 | GitOps/CI/CD | GitHub Actions → ECR → infra values → ArgoCD Sync | CI/CD 흐름도 |
| 10 | DB 운영 구조 | MariaDB는 K8s 외부 VM, 데이터 디스크는 Ceph RBD | DB 연결 흐름 |
| 11 | 모니터링/로깅/AIOps | Prometheus/Grafana/Alertmanager/Loki/HolmesGPT | Grafana, Alert 예시 |
| 12 | AWS Pilot Light DR | 평소에는 핵심 자원만 유지, 장애 시 EKS/ALB 활성화 | AWS DR 구조도 |
| 13 | 장애 전환 시나리오 | On-prem 장애 선언 후 RDS/EKS/ALB로 서비스 복구 | 데모 순서 |
| 14 | 트러블슈팅과 의사결정 | Calico, DB 권한, VPN/방화벽 등 실제 문제 해결 | 문제/원인/해결 표 |
| 15 | 성과와 한계 | 운영 플랫폼 구현, 수동 전환 한계와 개선점 | RPO/RTO, 향후 과제 |

## 4. 팀원별 발표 포인트

### A. On-prem / Proxmox / Network

말할 핵심:

- 5대 PC를 Proxmox VE 클러스터로 구성했다.
- `172.16.30.0/24` 관리망, VLAN 10/20/30/40 서비스망, `10.10.10.0/24` Ceph 스토리지망을 분리했다.
- 물리 방화벽이 없어서 `pve-1` 위 pfSense VM이 라우터, 방화벽, NAT, VLAN gateway 역할을 한다.
- HAProxy/Keepalived VIP를 통해 외부 진입점을 구성했다.

강조할 가치:

- 서비스 트래픽, 관리 트래픽, 스토리지 트래픽을 분리해 장애 범위와 보안 경계를 명확히 했다.

### B. Kubernetes Platform

말할 핵심:

- Kubernetes `v1.30.1`을 kubeadm 기반 HA control plane으로 구성했다.
- Control Plane은 `k8s-cp-1~3`, Worker는 `k8s-worker-1~5`로 구성했다.
- API VIP는 `172.16.43.99:6443`이며, kube-vip로 control plane endpoint 가용성을 확보했다.
- CNI는 Calico `v3.28.5`, Pod CIDR은 `10.244.0.0/16`, Service CIDR은 `10.96.0.0/12`이다.

강조할 가치:

- Ansible은 OS/Kubernetes bootstrap을 담당하고, Kubernetes 리소스는 Helm/manifest/GitOps로 분리했다.

### C. Application / Docker / Helm / ArgoCD

말할 핵심:

- FlaskApp은 Docker 이미지로 빌드되어 ECR에 저장된다.
- Helm chart는 `flaskapp-prod` namespace에 Deployment, Service, Ingress, ConfigMap, Secret, HPA, PDB, RBAC, NetworkPolicy를 배포한다.
- ConfigMap에는 공개 설정, Secret에는 DB 비밀번호처럼 민감한 값을 분리했다.
- readiness/liveness/startup probe와 HPA를 적용해 운영 안정성을 높였다.

강조할 가치:

- GitHub Actions가 이미지 태그를 갱신하고, ArgoCD가 이를 감지해 클러스터에 롤링 업데이트한다.

### D. Database / Backup / Monitoring

말할 핵심:

- MariaDB는 Kubernetes 내부가 아니라 `mariadb-1` 외부 VM에서 운영한다.
- DB VM은 VLAN 30 Internal의 `172.16.43.160`에 있으며, 데이터 디스크는 Ceph RBD를 사용한다.
- FlaskApp은 `DATABASE_HOST`, `DATABASE_USER`, `DATABASE_DB_NAME`, `DATABASE_PASSWORD` 환경변수로 DB에 접속한다.
- DB VM에는 node_exporter와 mysqld_exporter를 설치해 Kubernetes 내부 Prometheus가 외부 DB 상태도 수집할 수 있게 했다.

강조할 가치:

- Stateless 앱은 Kubernetes에, Stateful DB는 외부 VM에 두어 운영 책임과 장애 범위를 분리했다.

### E. AWS DR / Terraform

말할 핵심:

- AWS는 Pilot Light DR 구조로 설계했다.
- 평시에는 VPC, VPN, RDS, DMS, S3, ECR, 관측 자원을 유지하고, EKS/ALB는 꺼둔다.
- 장애 시 `dr_active=true`로 EKS와 ALB를 활성화한다.
- On-prem MariaDB 변경분은 DMS CDC로 AWS RDS에 복제한다.
- FlaskApp 파일 업로드는 AWS S3 단일 버킷을 사용해 온프렘과 DR 모두 같은 객체 저장소를 바라본다.

강조할 가치:

- 비용을 줄이면서도 장애 시 복구 가능한 구조를 만들기 위해 Pilot Light 방식을 선택했다.

## 5. 데모 시나리오

### 정상 운영 데모

1. Grafana에서 Kubernetes node/pod 상태를 보여준다.
2. ArgoCD에서 `flaskapp`, `monitoring`, `logging`, `aiops` Application 상태를 보여준다.
3. On-prem 도메인 또는 VIP로 FlaskApp에 접속한다.
4. FlaskApp에서 데이터 생성 또는 조회를 수행한다.
5. MariaDB `flaskapp.employee` 테이블에 데이터가 들어간 것을 확인한다.
6. 이미지/파일 업로드가 AWS S3 bucket을 사용하는 구조를 설명한다.
7. `kubectl get pods -n flaskapp-prod`로 replica와 rollout 상태를 확인한다.

### 장애/DR 데모

1. On-prem 장애 상황을 선언한다. 실제 차단을 하지 못하면 시나리오 기반으로 설명한다.
2. DMS replication lag 또는 RDS 데이터 상태를 확인한다.
3. RDS를 DR Primary로 사용하는 절차를 설명한다.
4. Terraform에서 `dr_active=true`로 EKS/ALB가 활성화되는 구조를 보여준다.
5. EKS에 FlaskApp이 배포되고 RDS/S3 설정으로 동작하는 흐름을 설명한다.
6. DNS 또는 hosts 전환 후 AWS ALB로 접속하는 구조를 설명한다.
7. 마지막에 RPO/RTO와 수동 전환 한계를 짚는다.

## 6. 발표 중 사용할 핵심 명령어

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get pods -n flaskapp-prod
kubectl rollout status deployment/flaskapp -n flaskapp-prod
kubectl get ingress -A
kubectl get applications -n argocd
kubectl get pods -n monitoring
kubectl get pods -n logging
kubectl get pods -n aiops
```

DB 확인:

```bash
kubectl exec -it -n flaskapp-prod deploy/flaskapp -- env | grep DATABASE
mysql -h 172.16.43.160 -u flaskapp -p flaskapp
```

모니터링 확인:

```bash
curl -I http://172.16.42.99 -H "Host: grafana.team.snow.internal"
curl http://172.16.43.160:9100/metrics | head
curl http://172.16.43.160:9104/metrics | head
```

AWS DR 설명용:

```bash
terraform plan -var="dr_active=false"
terraform plan -var="dr_active=true"
terraform apply -var="dr_active=true"
```

## 7. 예상 질문과 답변

### Q1. 왜 DB를 Kubernetes 안에 두지 않았나요?

DB는 상태 데이터가 핵심이기 때문에 Kubernetes 재구축이나 노드 장애의 영향을 줄이기 위해 외부 VM에서 운영했다. 데이터 디스크는 Ceph RBD를 사용해 VM OS 디스크와 분리했고, Kubernetes는 Stateless 애플리케이션 운영에 집중하도록 역할을 나눴다.

### Q2. 왜 AWS DR은 Active-Active가 아니라 Pilot Light인가요?

15일 프로젝트 범위와 비용을 고려하면 Active-Active는 운영 복잡도와 비용이 크다. Pilot Light는 평시에는 RDS, DMS, VPN, S3, ECR 같은 핵심 자원만 유지하고, 장애 시 EKS/ALB를 켜는 방식이라 비용과 복구 가능성의 균형이 좋다.

### Q3. RPO/RTO는 어떻게 설명하나요?

RPO는 DMS CDC 복제 지연 시간에 좌우된다. 완전 동기식 복제가 아니므로 데이터 손실 가능성을 0으로 보장하지는 않는다. RTO는 EKS/ALB 활성화, RDS 승격, DNS 전환, 앱 검증에 걸리는 시간으로 결정되며, 현재는 수동 절차이므로 자동 failover보다 길다.

### Q4. GitOps를 쓴 이유는 무엇인가요?

클러스터 상태를 사람이 직접 수정하면 현재 상태를 추적하기 어렵다. Helm values와 manifest를 Git에 두고 ArgoCD가 동기화하도록 만들면, 변경 이력과 배포 상태가 Git 기준으로 정리된다.

### Q5. 모니터링 VM은 왜 Prometheus/Grafana 설치용으로 쓰지 않았나요?

Kubernetes 상태를 수집하는 것이 주 목적이므로 Prometheus/Grafana는 클러스터 내부에 배포했다. 대신 monitoring VM은 외부 사용자 관점에서 VIP, Ingress, DB, Kubernetes API 상태를 확인하는 external probe 역할로 정리했다.

### Q6. HolmesGPT는 어떤 역할인가요?

Alertmanager가 장애 알림을 보내면 운영자가 alertname, namespace, service, description 같은 정보를 바탕으로 HolmesGPT에 질문한다. HolmesGPT는 Kubernetes 상태와 Loki 로그를 함께 참고해 원인 분석을 돕는다. 현재는 수동 분석 기반이고, 추후 webhook 기반 자동 분석으로 확장할 수 있다.

## 8. 발표에서 꼭 피해야 할 표현

- "그냥 해봤습니다" 대신 "운영 책임을 분리하기 위해 선택했습니다"라고 말한다.
- "AWS에 올렸습니다" 대신 "장애 시 필요한 계층만 활성화하는 Pilot Light DR 구조입니다"라고 말한다.
- "모니터링도 했습니다" 대신 "장애 판단을 위해 metric, log, alert, AI 분석 흐름을 연결했습니다"라고 말한다.
- "DB는 한 대입니다"만 말하지 말고 "On-prem 단일 Primary + AWS RDS DR Replica 모델입니다"라고 말한다.

## 9. 남은 준비 체크리스트

- [ ] 최종 아키텍처 다이어그램 1장 확정
- [ ] `kubectl get nodes -o wide` 캡처
- [ ] ArgoCD Applications Synced 화면 캡처
- [ ] Grafana 대시보드 캡처
- [ ] FlaskApp 정상 접속 화면 캡처
- [ ] DB insert/select 확인 캡처
- [ ] AWS Terraform plan/apply 결과 캡처
- [ ] RDS/DMS/S3/ECR/EKS 구성 화면 캡처
- [ ] 장애 전환 시나리오 리허설
- [ ] 팀원별 2분 내 발표 원고 정리

## 10. 10분 발표 시간 배분안

| 시간 | 담당 | 내용 |
| --- | --- | --- |
| 0:00-1:00 | 공통 | 프로젝트 목표와 전체 아키텍처 |
| 1:00-2:20 | A | Proxmox, 네트워크, Ceph |
| 2:20-3:40 | B | Kubernetes HA 플랫폼 |
| 3:40-5:00 | C | FlaskApp, Helm, ArgoCD, CI/CD |
| 5:00-6:20 | D | MariaDB, 백업/모니터링, AIOps |
| 6:20-7:50 | E | AWS DR, Terraform, Pilot Light |
| 7:50-9:20 | 공통 | 정상 운영 및 DR 데모 |
| 9:20-10:00 | 공통 | 성과, 한계, 개선 방향 |

## 11. 15분 발표 시간 배분안

| 시간 | 담당 | 내용 |
| --- | --- | --- |
| 0:00-1:30 | 공통 | 문제 정의, 목표, 전체 구조 |
| 1:30-3:00 | A | 물리 인프라, 네트워크, Proxmox/Ceph |
| 3:00-4:40 | B | Kubernetes HA, kube-vip, Calico, Ansible |
| 4:40-6:30 | C | FlaskApp 배포, Helm chart, GitOps/CI/CD |
| 6:30-8:10 | D | MariaDB 외부 VM, DB 연결, exporter, 모니터링 |
| 8:10-9:20 | D 또는 공통 | Alertmanager, Loki, HolmesGPT AIOps |
| 9:20-11:20 | E | AWS Pilot Light DR, DMS CDC, EKS/ALB 활성화 |
| 11:20-13:30 | 공통 | 정상 운영 데모 + DR 전환 시나리오 |
| 13:30-15:00 | 공통 | 트러블슈팅, 한계, 향후 개선 |

## 12. 마무리 멘트 초안

이번 프로젝트는 한 번에 모든 것을 완전 자동화한 DR 시스템은 아니지만, 온프레미스 운영 환경에서 실제 서비스를 배포하고, 데이터 복제와 AWS 복구 경로까지 연결했다는 점이 핵심입니다. 특히 인프라 구성, Kubernetes 플랫폼, GitOps 배포, DB 운영, 모니터링, DR 전환이 각각 따로 존재하는 것이 아니라 하나의 서비스 운영 흐름으로 연결되도록 설계했습니다. 향후에는 DMS 복제 상태 기반 자동 전환, DNS 자동화, DR 리허설 자동 검증까지 확장할 수 있습니다.

# KOSA Project Docs

이 저장소는 FlaskApp On-prem + AWS DR 프로젝트 문서를 관리한다.

## 먼저 읽을 문서

1. [current/latest-system-overview-2026-05-26.md](./current/latest-system-overview-2026-05-26.md)  
   현재 구현 기준 전체 시스템 큰그림이다.

2. [current/full-verification-assessment-2026-05-26.md](./current/full-verification-assessment-2026-05-26.md)  
   On-prem과 AWS를 구역별로 검증/평가한 최신 문서다.

3. [architecture/current-architecture-summary.md](./architecture/current-architecture-summary.md)  
   최신 아키텍처 흐름만 빠르게 볼 때 사용한다.

## 디렉터리 안내

| 디렉터리 | 역할 |
| --- | --- |
| [current](./current) | 최신 실측/검증 기준 문서 |
| [architecture](./architecture) | 전체 아키텍처, 네트워크, DNS, ADR, 과거 후보안 |
| [application](./application) | FlaskApp 실행 구조, Dockerfile, 환경변수, 배포 기준 |
| [aws](./aws) | AWS DR, Terraform, VPN, DMS/RDS/EKS/ALB 설계 |
| [database-backup-monitoring](./database-backup-monitoring) | DB 운영, 백업, 모니터링 설계 |
| [operation](./operation) | Kubernetes/App/모니터링 운영 Runbook |
| [setup](./setup) | Proxmox, Ceph, VM template, Terraform, Ansible 초기 구축 |
| [troubleshooting](./troubleshooting) | 구축 중 장애와 해결 기록 |
| [assets](./assets) | Mermaid 다이어그램/구성도 |
| [presentation](./presentation) | 발표 준비용 요약과 실측 상태 정리 |
| [github](./github) | GitHub 협업 방식, WBS, 프로젝트 사용법 |
| [meetings](./meetings) | 회의록 |

## 최신 기준 주의

- 최신 서비스 host는 `flaskapp.team.snow.internal`, `grafana.team.snow.internal`이다.
- 최신 On-prem 사용자 진입점은 HAProxy/Keepalived VIP `172.16.42.99`이다.
- ingress-nginx는 NodePort `30080/30443`으로 노출되고, FlaskApp Service는 ClusterIP다.
- `db.team.snow.internal`은 현재 Bind9에 없으며, FlaskApp은 DB IP `172.16.43.160` 기준으로 연결한다.
- 오래된 문서의 `flaskapp.onprem.local`, `grafana.onprem.local`, MetalLB 직접 진입, AWS Front Door/EC2 HAProxy 상시 진입 구조는 최신 구현과 다를 수 있다.

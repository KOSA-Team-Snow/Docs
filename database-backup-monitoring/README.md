# Database / Backup / Monitoring 문서 인덱스

이 디렉터리는 MariaDB 운영, Ceph RBD 저장소, DMS/RDS 복제, 모니터링 설계를 정리한다.

## 문서 상태

| 문서 | 상태 | 설명 |
| --- | --- | --- |
| [DBM-01-데이터베이스-운영-방식.md](./DBM-01-데이터베이스-운영-방식.md) | 최신 기준에 가까움 | MariaDB 외부 VM, Ceph RBD, DMS CDC 운영 결정 |
| [데이터베이스-백업-모니터링-구조-설계도.md](./데이터베이스-백업-모니터링-구조-설계도.md) | 과거 설계 포함 | Grafana MetalLB/`grafana.onprem.local` 표현은 최신 기준과 다름 |

## 최신 기준

- On-prem DB: MariaDB VM `172.16.43.160`
- FlaskApp DB 연결: IP `172.16.43.160` 기준
- `db.team.snow.internal`: 현재 Bind9 기준 없음
- DB 저장소: Proxmox/Ceph RBD
- DR 복제: AWS DMS Full Load + CDC -> RDS
- Monitoring/Logging/AIOps: Kubernetes 내부 `monitoring`, `logging`, `aiops` namespace 중심
- Grafana host: `grafana.team.snow.internal` -> `172.16.42.99`

# AWS 문서 인덱스

이 디렉터리는 AWS Pilot Light DR 설계, Terraform 작업 규칙, VPN 구성 문서를 관리한다.

## 먼저 읽을 문서

| 문서 | 상태 | 설명 |
| --- | --- | --- |
| [aws-architecture-final.md](./aws-architecture-final.md) | 최신 기준에 가까움 | v4 stack/Terraform 구조 기준 AWS DR 전체 설명 |
| [pfsense-vpn-터널-구성.md](./pfsense-vpn-터널-구성.md) | 구현 기록 | pfSense와 AWS Site-to-Site VPN 구성 기록 |
| [aws-teraform-작업-규칙.md](./aws-teraform-작업-규칙.md) | 운영 규칙 | Terraform plan/apply/destroy 기준 |
| [aws-dr-mvp-architecture-v2-re.md](./aws-dr-mvp-architecture-v2-re.md) | 참고 설계 | MVP 리뷰 보강판. 일부 Route 53 표현은 Bind9 데모 기준과 대조 필요 |
| [aws-old](./aws-old) | 과거 문서 | 이전 설계/구현안 보관 |

## 최신 실측 기준

- AWS 계정: `080252689380`
- Region: `ap-northeast-2`
- DR VPC: `10.20.0.0/16`
- VPN tunnel 2개 UP
- RDS: `flaskapp-dr-mariadb`
- DMS task: `flaskapp-dr-full-load-cdc`, running
- DMS validation: `Not enabled`
- ECR: `080252689380.dkr.ecr.ap-northeast-2.amazonaws.com/flaskapp`
- S3: `flaskapp-proddata-kosa-project-team3-snow-lai9z`
- EKS/ALB: 평시 `dr_active=false` 기준 미생성

상세 검증은 [../current/full-verification-assessment-2026-05-26.md](../current/full-verification-assessment-2026-05-26.md)를 우선한다.

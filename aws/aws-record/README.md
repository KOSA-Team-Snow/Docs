# v4 Apply Records

이 폴더는 AWS DR Terraform stack을 실제로 적용하기 전후의 기록을 남기는 곳입니다.

원칙:

- 실제 AWS 적용 전에는 `plan`, 입력값, 선행 조건을 먼저 기록합니다.
- Bind9, pfSense, HAProxy, Keepalived 같은 온프렘 작업은 이 stack의 적용 대상이 아니며, AWS 측 handoff 정보만 기록합니다.
- 민감값은 기록하지 않습니다. 비밀번호, access key, private key, VPN pre-shared key는 기록하지 않습니다.
- `*.log`, `*.txt`, VPN XML 같은 실행 산출물은 Git에 남기지 않습니다.

현재 적용 상태:

| 단계 | 파일 | 상태 |
|---|---|---|
| 01 | `step-01-pre-apply-readiness.md` | WSL/CLI 확인 |
| 02 | `step-02-apply-attempt.md` | Codex 셸과 사용자 WSL 차이 확인 |
| 03 | `step-03-input-values-check.md` | tfvars 주요 입력값 확인 |
| 04 | `step-04-wsl-cli-check.md` | 사용자 WSL CLI 확인 |
| 05 | `step-05-plan-summary.md` | 최초 plan 요약 |
| 06 | `step-06-existing-resources-check.md` | S3/ECR 기존 자원 확인 |
| 07 | `step-07-network-vpn.md` | network VPN apply 완료, 비용 추정 포함 |
| 08 | `step-08-remaining-base.md` | DMS 자동 시작 비활성화 후 나머지 기본 자원 적용 준비 |
| 09 | `step-09-current-progress-and-cost.md` | 현재 적용 진도와 전체 예상 비용 |
| 10 | `step-10-resource-health-check.md` | AWS CLI 직접 조회 기반 자원 활성화 점검 |
| 11 | `step-11-backend-and-remaining-work.md` | Terraform backend 사용 여부와 남은 작업 목록 |

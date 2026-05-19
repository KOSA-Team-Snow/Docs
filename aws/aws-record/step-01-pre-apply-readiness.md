# Step 01 - Pre-Apply Readiness

목표: Terraform apply 전에 AWS 담당 범위의 실행환경, 입력값, 선행 의존성을 확인한다.

## 확인 결과

| 항목 | 상태 |
|---|---|
| Terraform | 사용자 WSL에서 Terraform v1.15.3 확인 |
| AWS CLI | `aws sts get-caller-identity` 성공 |
| kubectl | client v1.30.14 확인 |
| jq | 초기에는 미설치였고, 이후 설치 대상 |
| Codex PowerShell | `terraform`, `aws` PATH 없음 |
| Codex WSL 접근 | 배포판 미탐지. 사용자 일반 WSL과 실행 컨텍스트가 다름 |

## 범위

v4는 AWS 전용 stack이다. Bind9 zone 파일 작성, SSH, `rndc reload`, pfSense 설정은 온프렘 담당 범위다.

## 다음 단계

사용자 WSL 터미널에서 Terraform/AWS 명령을 실행하고, Codex는 결과를 바탕으로 기록과 다음 단계를 안내한다.


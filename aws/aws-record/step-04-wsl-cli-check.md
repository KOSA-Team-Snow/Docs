# Step 04 - WSL CLI Check

목표: 사용자 WSL 터미널에서 Terraform 적용에 필요한 CLI와 입력값 상태를 확인한다.

## 실행 위치

작업 디렉터리:

```bash
/mnt/c/Users/ChlWoGur/Desktop/클라우드/프젝/terraform_pr/v4
```

## CLI 확인 결과

| 항목 | 결과 |
|---|---|
| Terraform | v1.15.3 linux_amd64 |
| AWS 인증 | `aws sts get-caller-identity` 성공, `project-admin` 사용자 |
| kubectl | client v1.30.14 |
| jq | 초기에는 미설치. helper script 사용 전 설치 필요 |

## 결론

사용자 WSL에서 Terraform/AWS 적용을 진행할 수 있다. Codex는 로컬 PowerShell에서 직접 apply하지 않고, 사용자 실행 결과를 바탕으로 다음 단계를 관리한다.


# Step 05 - Terraform Init, Validate, Plan Summary

목표: 실제 apply 전 Terraform 초기화, 검증, plan 결과를 요약한다.

## 실행 결과

| 항목 | 결과 |
|---|---|
| `terraform fmt -check -recursive terraform` | `terraform/envs/dr/terraform.tfvars` 포맷 차이 감지 |
| `terraform -chdir=terraform/envs/dr init -backend-config=backend.hcl` | 성공. S3 backend 설정 완료 |
| `terraform -chdir=terraform/envs/dr validate` | 성공. configuration valid |
| 최초 전체 plan | 성공. `55 to add, 0 to change, 0 to destroy` |

## 최초 Plan 요약

| 모듈 | 생성 예정 수 | 주요 자원 |
|---|---:|---|
| `network` | 26 | VPC, subnets, route tables, NAT, S3 endpoint, CGW, VGW, VPN |
| `sg` | 5 | ALB, EKS cluster, EKS node, DMS, RDS security groups |
| `s3` | 6 | proddata bucket 및 관련 설정 |
| `ecr` | 1 | FlaskApp ECR repository |
| `iam` | 4 | DMS IAM roles 및 attachments |
| `rds` | 3 | RDS MariaDB, subnet group, parameter group |
| `dms` | 5 | DMS endpoints, instance, subnet group, task |
| `observability` | 5 | SNS, alarms, dashboard |

## 주의

S3 bucket과 ECR repository가 생성 대상으로 떠서, 기존 자원이 있다면 import가 필요하다고 판단했다.


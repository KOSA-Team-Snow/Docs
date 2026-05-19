# Step 06 - Existing Resources Check

목표: Terraform apply 전에 기존 AWS 자원 존재 여부를 확인하고 import 대상을 결정한다.

## 확인 결과

| 자원 | 결과 | 조치 |
|---|---|---|
| S3 proddata bucket | 존재. `arn:aws:s3:::flaskapp-proddata-kosa-project-team3-snow-lai9z` | Terraform state로 import 필요 |
| ECR repository `flaskapp` | 존재. account `080252689380`, immutable, scan on push true | Terraform state로 import 필요 |
| IAM role `dms-vpc-role` | 없음 (`NoSuchEntity`) | Terraform이 새로 생성 |
| IAM role `dms-cloudwatch-logs-role` | 없음 (`NoSuchEntity`) | Terraform이 새로 생성 |

## 결론

- S3/ECR은 기존 자원을 import한다.
- DMS IAM role은 `manage_dms_roles = true` 상태로 Terraform이 새로 생성해도 된다.


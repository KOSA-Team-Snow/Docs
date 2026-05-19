# Step 08 - Remaining Base Resources

목표: VPN/pfSense 연결이 아직 없는 상태에서 DMS 연결은 시작하지 않고, 나머지 AWS 기본 자원을 `dr_active=false` 기준으로 생성한다.

## 적용 정책

- `dr_active=false`를 유지한다.
- EKS cluster, node group, ALB Controller, Kubernetes ALB는 이번 단계에서 생성하지 않는다.
- DMS replication instance, source endpoint, target endpoint, replication task는 만들 수 있다.
- 단, DMS replication task는 자동 시작하지 않는다.
- VPN 연결 후 DMS source/target connection test를 하고, 그 다음 replication task를 수동 시작한다.

## Terraform 변경

| 파일 | 변경 |
|---|---|
| `terraform/modules/dms/variables.tf` | `start_replication_task` 변수 추가, 기본값 `false` |
| `terraform/modules/dms/main.tf` | `start_replication_task = var.start_replication_task`로 변경 |
| `terraform/modules/dms/outputs.tf` | 최초 시작용 `start_task` operator command 추가 |
| `terraform/envs/dr/variables.tf` | `dms_start_replication_task` 변수 추가, 기본값 `false` |
| `terraform/envs/dr/main.tf` | root variable을 DMS module로 전달 |
| `terraform/envs/dr/outputs.tf` | `operator_runbook.step_5_start_dms` 추가 |
| `terraform/envs/dr/terraform.tfvars` | `dms_start_replication_task = false` 명시 |
| `terraform/modules/dms/main.tf` | 현재 리전에서 지원되지 않는 DMS engine `3.5.2` 하드코딩 제거 |
| `terraform/modules/rds/main.tf` | RDS static parameter 변경을 `pending-reboot`로 적용 |

## Apply 중 발견된 오류와 조치

첫 apply에서 아래 두 오류가 발생했다.

| 오류 | 원인 | 조치 |
|---|---|---|
| `No replication engine found with version: 3.5.2` | `ap-northeast-2`에서 해당 DMS replication engine version을 생성할 수 없음 | `engine_version` 하드코딩을 제거하고 AWS 기본 지원 버전을 사용 |
| `cannot use immediate apply method for static parameter` | RDS parameter group의 static parameter가 기본 `immediate` 방식으로 수정됨 | `binlog_*` parameter에 `apply_method = "pending-reboot"` 명시 |

수정 후 `terraform validate`는 성공했다.

## 재시도 명령

위 오류 수정 후에는 기존 plan 파일을 재사용하지 말고 새 plan을 만든다.

```bash
cd /mnt/c/Users/ChlWoGur/Desktop/클라우드/프젝/terraform_pr/v4

aws sts get-caller-identity

terraform -chdir=terraform/envs/dr fmt
terraform -chdir=terraform/envs/dr validate

terraform -chdir=terraform/envs/dr plan \
  -out=remaining-base-retry.tfplan

terraform -chdir=terraform/envs/dr apply remaining-base-retry.tfplan
```

Codex WSL 세션에서는 AWS login session expired 상태라 remote state plan/apply까지 진행하지 못했다. 사용자 WSL 터미널에서 인증이 살아있는 상태로 재시도한다.

## 현재 의도한 상태

| 항목 | 값 |
|---|---|
| `dr_active` | `false` |
| `dms_start_replication_task` | `false` |
| VPN tunnel | pfSense 설정 전이므로 `DOWN` 예상 |
| DMS task | 생성되더라도 자동 시작하지 않음 |

## WSL 적용 명령

```bash
cd /mnt/c/Users/ChlWoGur/Desktop/클라우드/프젝/terraform_pr/v4

terraform -chdir=terraform/envs/dr fmt
terraform -chdir=terraform/envs/dr validate

terraform -chdir=terraform/envs/dr state list | sort

terraform -chdir=terraform/envs/dr plan \
  -out=remaining-base.tfplan

terraform -chdir=terraform/envs/dr apply remaining-base.tfplan

terraform -chdir=terraform/envs/dr output
```

## 기존 S3/ECR state 확인

S3 bucket과 ECR repository는 이미 AWS에 존재하므로 Terraform state에 import되어 있어야 한다. `state list`에 아래 주소가 없으면 apply 전에 import한다.

```bash
terraform -chdir=terraform/envs/dr state list | grep -E 'module\.s3\.aws_s3_bucket\.proddata|module\.ecr\.aws_ecr_repository\.flaskapp'
```

필요 시 import:

```bash
terraform -chdir=terraform/envs/dr import \
  'module.s3.aws_s3_bucket.proddata' \
  flaskapp-proddata-kosa-project-team3-snow-lai9z

terraform -chdir=terraform/envs/dr import \
  'module.ecr.aws_ecr_repository.flaskapp' \
  flaskapp
```

## Apply 후 확인

```bash
terraform -chdir=terraform/envs/dr output dms_replication_task_id

aws dms describe-replication-tasks \
  --region ap-northeast-2 \
  --filters Name=replication-task-id,Values=flaskapp-dr-full-load-cdc \
  --query 'ReplicationTasks[0].Status' \
  --output text
```

기대 상태:

- Terraform apply는 성공한다.
- DMS replication task status는 보통 `ready` 또는 이에 준하는 미시작 상태여야 한다.
- `running`이면 안 된다.

## VPN 이후 DMS 시작

VPN tunnel이 `UP`이고 온프렘 MariaDB 접근이 확인된 뒤 아래 흐름으로 진행한다.

```bash
terraform -chdir=terraform/envs/dr output operator_runbook
```

`step_5_start_dms` 명령을 사용해 최초 replication task를 시작한다.

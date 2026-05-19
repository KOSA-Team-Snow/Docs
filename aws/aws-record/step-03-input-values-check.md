# Step 03 - Input Values Check

목표: 실제 `terraform apply` 전에 `backend.hcl`과 `terraform.tfvars` 준비 상태를 확인한다.

## Backend

`terraform/envs/dr/backend.hcl`:

```hcl
bucket  = "flaskapp-tfstate-kosa-project-team3-snow-a3asx"
key     = "dr/v4/terraform.tfstate"
region  = "ap-northeast-2"
encrypt = true
```

`backend.tf`에서 S3 native lock `use_lockfile = true`를 사용한다. Terraform 1.10 이상 권장이고, 현재 사용자 WSL은 v1.15.3이다.

## 주요 입력값

| 변수 | 값/상태 |
|---|---|
| `pfsense_public_ip` | `125.131.208.229` |
| `admin_allowed_cidrs` | `["125.131.208.229/32"]` |
| `onprem_cidr` | `172.16.0.0/16` |
| `onprem_db_ip` | `172.16.43.160` |
| `proddata_bucket_name` | `flaskapp-proddata-kosa-project-team3-snow-lai9z` |
| `ecr_repository_name` | `flaskapp` |
| `bind9_vm_ip` | `172.16.40.10` |
| `service_domain` | `flaskapp.onprem.local` |

민감값은 record에 기록하지 않는다.


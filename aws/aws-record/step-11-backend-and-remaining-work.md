# Step 11 - Backend and Remaining Work

기준일: 2026-05-19

목표:

- v4 Terraform이 tfstate S3 bucket과 DynamoDB lock table을 실제로 어떻게 쓰고 있는지 확인한다.
- 지금부터 남은 작업을 실행 순서 기준으로 정리한다.

## 결론

| 항목 | 현재 사용 여부 | 판단 |
|---|---|---|
| S3 bucket `flaskapp-tfstate-kosa-project-team3-snow-a3asx` | 사용 중 | v4 remote state backend bucket |
| S3 object `dr/v4/terraform.tfstate` | 사용 중 | v4 state object가 존재하고 마지막 수정됨 |
| S3 native lockfile | 미사용 | `use_lockfile` 설정 제거 |
| DynamoDB table `terraform-state-lock` | 사용 중 | v4 backend의 `dynamodb_table`로 설정 |

정리하면, **v4는 기존 S3 tfstate bucket과 기존 DynamoDB `terraform-state-lock` table을 함께 사용한다.**

state key는 `dr/v4/terraform.tfstate`로 분리되어 있으므로 기존 v3-dr state와 충돌하지 않는다. lock table은 공용이지만 LockID가 state 경로 기준으로 잡히므로 같은 state를 동시에 만지는 경우만 막는다.

## Backend 설정 확인

Command:

```powershell
Get-Content -LiteralPath terraform\envs\dr\backend.tf -Encoding UTF8
```

Result:

```hcl
terraform {
  backend "s3" {}
}
```

Command:

```powershell
Get-Content -LiteralPath terraform\envs\dr\backend.hcl -Encoding UTF8
```

Result:

```hcl
bucket  = "flaskapp-tfstate-kosa-project-team3-snow-a3asx"
key     = "dr/v4/terraform.tfstate"
region  = "ap-northeast-2"
encrypt = true
dynamodb_table = "terraform-state-lock"
```

Command:

```powershell
Get-Content -LiteralPath terraform\envs\dr\.terraform\terraform.tfstate -Encoding UTF8
```

Result 요약:

```json
{
  "backend": {
    "type": "s3",
    "config": {
      "bucket": "flaskapp-tfstate-kosa-project-team3-snow-a3asx",
      "key": "dr/v4/terraform.tfstate",
      "region": "ap-northeast-2",
      "encrypt": true,
      "dynamodb_table": "terraform-state-lock",
      "use_lockfile": null
    }
  }
}
```

판정:

- v4 backend는 S3 remote state를 사용한다.
- v4 backend는 DynamoDB `terraform-state-lock` table을 state lock으로 사용한다.
- v4 locking은 S3 native lockfile이 아니라 기존 팀 공용 DynamoDB lock 방식이다.
- Terraform 1.15.3에서는 `dynamodb_table` backend parameter가 deprecated warning을 출력하지만, 현재 요청한 기존 lock table 방식으로 동작한다.

참고 메모:

- Terraform의 S3 backend는 예전부터 DynamoDB table로 state lock을 잡는 방식을 많이 써왔다.
- Terraform 1.10 이후에는 S3 자체 lockfile 방식인 `use_lockfile = true`가 도입되면서 DynamoDB lock 방식이 deprecated로 표시된다.
- deprecated는 지금 당장 실패한다는 뜻이 아니라, 앞으로는 새 방식 사용을 권장하고 장기적으로 제거될 수 있다는 신호다.
- 이번 v4에서는 팀의 기존 S3 tfstate/DynamoDB lock 운영 방식과 맞추기 위해 `dynamodb_table = "terraform-state-lock"`를 사용한다.

## AWS에서 tfstate bucket 확인

Command:

```bash
AWS_PROFILE=project-admin aws s3api head-bucket \
  --bucket flaskapp-tfstate-kosa-project-team3-snow-a3asx \
  --region ap-northeast-2 \
  --output json
```

Result:

```json
{
    "BucketArn": "arn:aws:s3:::flaskapp-tfstate-kosa-project-team3-snow-a3asx",
    "BucketRegion": "ap-northeast-2",
    "AccessPointAlias": false
}
```

판정: tfstate bucket은 존재하고 접근 가능하다.

## AWS에서 v4 state object 확인

Command:

```bash
AWS_PROFILE=project-admin aws s3api head-object \
  --bucket flaskapp-tfstate-kosa-project-team3-snow-a3asx \
  --key dr/v4/terraform.tfstate \
  --region ap-northeast-2 \
  --query '{ContentLength:ContentLength,LastModified:LastModified,ServerSideEncryption:ServerSideEncryption,VersionId:VersionId}' \
  --output json
```

Result:

```json
{
    "ContentLength": 131776,
    "LastModified": "2026-05-19T09:27:54+00:00",
    "ServerSideEncryption": "AES256",
    "VersionId": "YNoEV52CZz9BmB_FS0LZ4.O_ee1Yflmi"
}
```

판정: v4 state object가 실제 S3에 저장되어 있고, AES256 server-side encryption이 적용되어 있다.

Command:

```bash
AWS_PROFILE=project-admin aws s3api list-objects-v2 \
  --bucket flaskapp-tfstate-kosa-project-team3-snow-a3asx \
  --prefix dr/v4/ \
  --region ap-northeast-2 \
  --query 'Contents[].{Key:Key,Size:Size,LastModified:LastModified}' \
  --output table
```

Result:

```text
--------------------------------------------------------------------
|                           ListObjectsV2                          |
+--------------------------+-----------------------------+---------+
|            Key           |        LastModified         |  Size   |
+--------------------------+-----------------------------+---------+
|  dr/v4/terraform.tfstate |  2026-05-19T09:27:54+00:00  |  131776 |
+--------------------------+-----------------------------+---------+
```

판정: 현재 prefix에는 state object만 남아 있다. S3 native lockfile은 사용하지 않으므로 `.tflock` 객체가 없는 것이 정상이다.

## AWS에서 DynamoDB lock table 확인

Command:

```bash
AWS_PROFILE=project-admin aws dynamodb describe-table \
  --table-name terraform-state-lock \
  --region ap-northeast-2 \
  --query 'Table.{TableName:TableName,TableStatus:TableStatus,ItemCount:ItemCount,BillingMode:BillingModeSummary.BillingMode,KeySchema:KeySchema,CreationDateTime:CreationDateTime}' \
  --output json
```

Result:

```json
{
    "TableName": "terraform-state-lock",
    "TableStatus": "ACTIVE",
    "ItemCount": 0,
    "BillingMode": "PAY_PER_REQUEST",
    "KeySchema": [
        {
            "AttributeName": "LockID",
            "KeyType": "HASH"
        }
    ],
    "CreationDateTime": "2026-05-08T16:03:35.353000+09:00"
}
```

판정:

- table은 존재하고 `ACTIVE`다.
- 현재 item count는 0이다.
- v4 backend 설정에 `dynamodb_table = "terraform-state-lock"`가 들어갔으므로 v4에서도 이 table을 사용한다.
- 현재 item count가 0인 것은 남아 있는 stale lock이 없다는 의미다.

## Backend 변경 적용 결과

Command:

```bash
AWS_PROFILE=project-admin terraform -chdir=terraform/envs/dr init \
  -reconfigure \
  -backend-config=backend.hcl
```

Result 요약:

```text
Successfully configured the backend "s3"!
Terraform has been successfully initialized!
```

Warning:

```text
The parameter "dynamodb_table" is deprecated. Use parameter "use_lockfile" instead.
```

판정:

- backend 재초기화는 성공했다.
- 같은 S3 bucket/key를 유지했으므로 state migration은 발생하지 않았다.
- lock 방식만 S3 native lockfile에서 DynamoDB table로 변경됐다.

Command:

```bash
AWS_PROFILE=project-admin terraform -chdir=terraform/envs/dr state list | head -n 10
```

Result:

```text
module.dms.aws_dms_endpoint.source
module.dms.aws_dms_endpoint.target
module.dms.aws_dms_replication_instance.this
module.dms.aws_dms_replication_subnet_group.this
module.dms.aws_dms_replication_task.this
module.ecr.aws_ecr_repository.flaskapp
module.iam.aws_iam_role.dms_cloudwatch[0]
module.iam.aws_iam_role.dms_vpc[0]
module.iam.aws_iam_role_policy_attachment.dms_cloudwatch[0]
module.iam.aws_iam_role_policy_attachment.dms_vpc[0]
```

판정: backend 변경 후에도 remote state 접근이 정상이다.

Command:

```bash
AWS_PROFILE=project-admin aws dynamodb scan \
  --table-name terraform-state-lock \
  --region ap-northeast-2 \
  --select COUNT \
  --output json
```

Result:

```json
{
    "Count": 0,
    "ScannedCount": 0,
    "ConsumedCapacity": null
}
```

판정: backend 재초기화 후에도 잔여 lock item은 없다.

Command:

```bash
AWS_PROFILE=project-admin terraform -chdir=terraform/envs/dr validate
```

Result:

```text
Success! The configuration is valid.
```

## 남은 작업 목록

### 1. SNS email confirmation

상태:

- SNS topic `flaskapp-dr-alarms` 생성 완료.
- email subscription은 `PendingConfirmation`.

해야 할 일:

- `silveriver31@naver.com` 메일함에서 AWS SNS confirmation 메일 승인.
- 승인 후 아래 명령으로 `SubscriptionArn`이 실제 ARN으로 바뀌었는지 확인.

```bash
AWS_PROFILE=project-admin aws sns list-subscriptions-by-topic \
  --region ap-northeast-2 \
  --topic-arn arn:aws:sns:ap-northeast-2:080252689380:flaskapp-dr-alarms \
  --query 'Subscriptions[].{Protocol:Protocol,SubscriptionArn:SubscriptionArn}' \
  --output json
```

### 2. VPN tunnel 2 구성 또는 의도적 보류 결정

현재 상태:

- Tunnel 1: `UP`
- Tunnel 2: `DOWN`

해야 할 일:

- 고가용성이 필요하면 pfSense에서 두 번째 tunnel도 구성.
- MVP 리허설에서 tunnel 1만으로 충분하면 문서에 의도적 보류로 남김.

확인 명령:

```bash
AWS_PROFILE=project-admin aws ec2 describe-vpn-connections \
  --region ap-northeast-2 \
  --vpn-connection-ids vpn-02820049cf4de6764 \
  --query 'VpnConnections[0].VgwTelemetry[*].{OutsideIp:OutsideIpAddress,Status:Status,AcceptedRouteCount:AcceptedRouteCount,LastStatusChange:LastStatusChange}' \
  --output table
```

### 3. AWS ↔ 온프렘 라우팅/방화벽 검증

현재 상태:

- AWS route table에는 `172.16.0.0/16` 경로가 VGW로 잡혀 있다.
- VPN tunnel 1이 `UP`.

해야 할 일:

- 온프렘 쪽 route/firewall에서 AWS VPC `10.20.0.0/16` 왕복 경로 확인.
- 온프렘 MariaDB `172.16.43.160:3306`에 DMS SG에서 접근 가능한지 확인.
- pfSense/라우터 쪽 NAT 또는 policy가 source를 바꾸지 않는지 확인.

### 4. RDS parameter 실제 반영 확인

현재 상태:

- parameter group에 `binlog_format=ROW`, `binlog_row_image=FULL`, `binlog_checksum=NONE` 설정됨.
- apply method는 `pending-reboot`.

해야 할 일:

- RDS 재부팅 가능 시점에 재부팅.
- 재부팅 후 DB 내부에서 실제 값 확인.

예상 확인 SQL:

```sql
SHOW VARIABLES LIKE 'binlog_format';
SHOW VARIABLES LIKE 'binlog_row_image';
SHOW VARIABLES LIKE 'binlog_checksum';
```

주의:

- RDS 재부팅은 짧은 DB 중단을 만들 수 있으므로 리허설/검증 시간에 수행한다.

### 5. DMS endpoint connection test

현재 상태:

- DMS instance: `available`
- DMS endpoints: `active`
- DMS task: `ready`
- DMS task는 아직 시작하지 않음.

해야 할 일:

- source endpoint connection test.
- target endpoint connection test.
- 둘 다 successful 이후 DMS task 시작.

예시 흐름:

```bash
REPL_ARN="$(AWS_PROFILE=project-admin aws dms describe-replication-instances \
  --region ap-northeast-2 \
  --filters Name=replication-instance-id,Values=flaskapp-dr-dms \
  --query 'ReplicationInstances[0].ReplicationInstanceArn' \
  --output text)"

SOURCE_ARN="$(AWS_PROFILE=project-admin aws dms describe-endpoints \
  --region ap-northeast-2 \
  --filters Name=endpoint-id,Values=flaskapp-dr-source \
  --query 'Endpoints[0].EndpointArn' \
  --output text)"

TARGET_ARN="$(AWS_PROFILE=project-admin aws dms describe-endpoints \
  --region ap-northeast-2 \
  --filters Name=endpoint-id,Values=flaskapp-dr-target \
  --query 'Endpoints[0].EndpointArn' \
  --output text)"

AWS_PROFILE=project-admin aws dms test-connection \
  --region ap-northeast-2 \
  --replication-instance-arn "$REPL_ARN" \
  --endpoint-arn "$SOURCE_ARN"

AWS_PROFILE=project-admin aws dms test-connection \
  --region ap-northeast-2 \
  --replication-instance-arn "$REPL_ARN" \
  --endpoint-arn "$TARGET_ARN"
```

결과 확인:

```bash
AWS_PROFILE=project-admin aws dms describe-connections \
  --region ap-northeast-2 \
  --filters Name=replication-instance-arn,Values="$REPL_ARN" \
  --query 'Connections[].{EndpointArn:EndpointArn,Status:Status,LastFailureMessage:LastFailureMessage}' \
  --output table
```

### 6. DMS task 시작

조건:

- VPN 최소 1개 tunnel `UP`.
- source/target endpoint connection test 성공.
- 온프렘 MariaDB replication user 권한 확인.
- RDS binlog parameter 반영 확인.

시작 명령:

```bash
AWS_PROFILE=project-admin aws dms start-replication-task \
  --region ap-northeast-2 \
  --replication-task-arn arn:aws:dms:ap-northeast-2:080252689380:task:ZKSWKBLADZEDXFD2WTJU2VTP6E \
  --start-replication-task-type start-replication
```

상태 확인:

```bash
AWS_PROFILE=project-admin aws dms describe-replication-tasks \
  --region ap-northeast-2 \
  --filters Name=replication-task-id,Values=flaskapp-dr-full-load-cdc \
  --query 'ReplicationTasks[0].{Status:Status,StopReason:StopReason,LastFailureMessage:LastFailureMessage,Stats:ReplicationTaskStats}' \
  --output json
```

### 7. DMS lag monitoring

해야 할 일:

- full load 이후 CDC 상태 확인.
- CloudWatch alarm `flaskapp-dr-dms-cdc-lag`가 데이터 수집을 시작하는지 확인.

확인 명령:

```bash
AWS_PROFILE=project-admin aws cloudwatch get-metric-statistics \
  --region ap-northeast-2 \
  --namespace AWS/DMS \
  --metric-name CDCLatencyTarget \
  --dimensions Name=ReplicationInstanceIdentifier,Value=flaskapp-dr-dms Name=ReplicationTaskIdentifier,Value=flaskapp-dr-full-load-cdc \
  --start-time "$(date -u -d '30 min ago' +%FT%TZ)" \
  --end-time "$(date -u +%FT%TZ)" \
  --period 60 \
  --statistics Maximum \
  --output json
```

### 8. DR 리허설 시 EKS/ALB 생성

현재 상태:

- `dr_active=false`
- EKS cluster 없음.
- ALB 없음.
- EC2 worker node 없음.

해야 할 일:

- DR 리허설 또는 전환 시점에만 `dr_active=true`로 apply.

명령:

```bash
cd /mnt/c/Users/ChlWoGur/Desktop/클라우드/프젝/terraform_pr/v4

AWS_PROFILE=project-admin terraform -chdir=terraform/envs/dr plan \
  -var="dr_active=true" \
  -out=dr-active.tfplan

AWS_PROFILE=project-admin terraform -chdir=terraform/envs/dr apply dr-active.tfplan
```

주의:

- EKS/ALB는 비용이 크게 증가한다.
- Helm provider가 EKS API에 접근해야 하므로 `admin_allowed_cidrs`가 현재 접속 IP를 포함해야 한다.

### 9. Application deployment and ALB DNS handoff

해야 할 일:

- EKS 생성 후 kubeconfig 업데이트.
- Flask app manifest/Helm/Kustomize 적용.
- AWS Load Balancer Controller가 ALB를 생성하는지 확인.
- ALB DNS를 온프렘 Bind9 담당자에게 전달.
- `flaskapp.team.snow.internal` CNAME을 ALB DNS로 반영.

현재 operator output:

```text
step_16_aws_prepare = /mnt/c/Users/ChlWoGur/Desktop/클라우드/프젝/terraform_pr/v4/scripts/dr-step-16-aws-prepare.sh
step_17_verify_dns = dig @172.16.40.10 flaskapp.team.snow.internal +noall +answer
```

### 10. 비용 관리 결정

현재 월 예상:

```text
약 $174.76 / 30 days
```

비용이 부담되면 우선순위:

1. DMS replication instance 정리 또는 일시 중단 대안 검토.
2. NAT Gateway 정리 가능 여부 검토.
3. VPN 유지 필요 여부 검토.
4. RDS 유지 필요 여부 검토.

단, 현재는 VPN/DMS/RDS 검증 단계이므로 무작정 destroy하면 진행 흐름이 끊긴다.

### 11. 최종 문서화

해야 할 일:

- pfSense tunnel 설정 결과 문서화.
- DMS endpoint test 결과 문서화.
- DMS task start 결과와 lag 문서화.
- EKS/ALB를 켠 경우 ALB DNS, Bind9 handoff, 서비스 접속 검증 문서화.
- 실제 리허설 순서와 rollback/cleanup 절차를 하나의 운영 runbook으로 압축.

### 12. GitHub 연결과 커밋 기반 변경 관리

현재 상태:

- v4 코드와 적용 기록 Markdown은 로컬 workspace에 존재한다.
- 이 폴더 자체는 현재 Codex 기준으로 Git repository로 인식되지 않는다.
- 이후 팀 협업과 변경 이력 관리를 위해 GitHub repository에 연결하는 절차가 필요하다.

해야 할 일:

- GitHub에 팀 repository 또는 개인 작업 branch를 준비한다.
- 현재 v4 코드를 Git repository로 초기화하거나 기존 repository의 올바른 위치로 옮긴다.
- `.gitignore`가 민감 파일을 제외하는지 확인한다.
- `terraform.tfvars`, `backend.hcl`, `.terraform/`, `*.tfstate`, `*.tfplan`, VPN XML, 실행 log는 commit하지 않는다.
- `terraform.tfvars.example`, `backend.hcl.example`, Terraform module/root 코드, `record/*.md`는 commit 대상으로 둔다.
- 변경은 앞으로 직접 콘솔에서 흩어지게 수정하지 말고 Git commit 단위로 남긴다.

예시 흐름:

```bash
git init
git status
git add README.md .gitignore terraform scripts record
git status
git commit -m "Add AWS DR v4 Terraform stack and apply records"
git branch -M main
git remote add origin <GITHUB_REPOSITORY_URL>
git push -u origin main
```

주의:

- commit 전 `git status`에서 `terraform.tfvars`, `backend.hcl`, `.terraform/`, VPN XML, log 파일이 staged 되지 않았는지 반드시 확인한다.
- 이미 팀 repository가 있다면 새 repository를 만들기보다 팀 repository의 branch/PR 흐름에 맞춘다.
- AWS 적용 변경은 가능하면 `plan` 결과 확인 후 commit/PR에 요약을 남긴다.

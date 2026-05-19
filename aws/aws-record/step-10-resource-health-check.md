# Step 10 - Resource Health Check

기준일: 2026-05-19

목표: 지금까지 Terraform으로 생성 또는 관리 대상으로 편입한 AWS 자원을 AWS CLI로 직접 조회하고, 활성화 여부를 기록한다.

주의:

- VPN customer gateway configuration XML은 pre-shared key가 포함되므로 조회하지 않았다.
- SNS email endpoint는 개인정보라 결과에서 제외하고 protocol/subscription status만 조회했다.
- 모든 명령은 WSL에서 `AWS_PROFILE=project-admin`, `ap-northeast-2` 기준으로 실행했다.

## 종합 판정

| 영역 | 상태 | 판정 |
|---|---|---|
| AWS 인증 | `project-admin` profile 성공 | 정상 |
| VPC/Subnet/Route | `available`, route `active` | 정상 |
| NAT Gateway | `available` | 정상 |
| S3 Gateway Endpoint | `available` | 정상 |
| Site-to-Site VPN | VPN connection `available`, tunnel 1 `UP`, tunnel 2 `DOWN` | 부분 정상. 최소 1개 tunnel 연결됨 |
| RDS | `available` | 정상 |
| RDS parameter group | user parameter 존재, `pending-reboot` | 정상. 재부팅 전까지 일부 값은 pending |
| DMS instance | `available` | 정상 |
| DMS endpoints | `active` | 정상 |
| DMS task | `ready` | 정상. 아직 복제 시작하지 않음 |
| S3 bucket | 존재, versioning/encryption/public block 정상 | 정상 |
| ECR repository | 존재, immutable, scan on push | 정상 |
| ECR image | `latest`, `d457c7e` image 존재 | 정상 |
| DMS IAM roles | role/policy attachment 존재 | 정상 |
| Security Groups | 생성됨 | 정상 |
| Observability | SNS/alarms/dashboard 생성 | 정상. SNS email은 confirmation 대기 |
| EC2/EKS/ALB/ASG | 없음 | 정상. `dr_active=false` 상태 |

## 1. AWS 인증 확인

Command:

```bash
AWS_PROFILE=project-admin aws sts get-caller-identity --output json
```

Result:

```json
{
    "UserId": "AIDARFL3O3PSGFKFRBSRQ",
    "Account": "080252689380",
    "Arn": "arn:aws:iam::080252689380:user/project-admin"
}
```

판정: `project-admin` IAM user로 정상 인증됨.

## 2. Terraform 관리 대상 확인

Command:

```bash
AWS_PROFILE=project-admin terraform -chdir=terraform/envs/dr state list | sort
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
module.network.aws_customer_gateway.this
module.network.aws_eip.nat[0]
module.network.aws_internet_gateway.this
module.network.aws_nat_gateway.this[0]
module.network.aws_route_table.app[0]
module.network.aws_route_table.app[1]
module.network.aws_route_table.data[0]
module.network.aws_route_table.data[1]
module.network.aws_route_table.public
module.network.aws_route_table_association.app[0]
module.network.aws_route_table_association.app[1]
module.network.aws_route_table_association.data[0]
module.network.aws_route_table_association.data[1]
module.network.aws_route_table_association.public[0]
module.network.aws_route_table_association.public[1]
module.network.aws_subnet.app[0]
module.network.aws_subnet.app[1]
module.network.aws_subnet.data[0]
module.network.aws_subnet.data[1]
module.network.aws_subnet.public[0]
module.network.aws_subnet.public[1]
module.network.aws_vpc.this
module.network.aws_vpc_endpoint.s3
module.network.aws_vpn_connection.this
module.network.aws_vpn_connection_route.onprem
module.network.aws_vpn_gateway.this
module.observability.aws_cloudwatch_dashboard.dr_readiness
module.observability.aws_cloudwatch_metric_alarm.dms_cdc_lag
module.observability.aws_cloudwatch_metric_alarm.rds_storage_low
module.observability.aws_sns_topic.dr_alarms
module.observability.aws_sns_topic_subscription.email
module.rds.aws_db_instance.this
module.rds.aws_db_parameter_group.mariadb
module.rds.aws_db_subnet_group.this
module.s3.aws_s3_bucket.proddata
module.s3.aws_s3_bucket_ownership_controls.proddata
module.s3.aws_s3_bucket_policy.proddata
module.s3.aws_s3_bucket_public_access_block.proddata
module.s3.aws_s3_bucket_server_side_encryption_configuration.proddata
module.s3.aws_s3_bucket_versioning.proddata
module.sg.aws_security_group.alb
module.sg.aws_security_group.cluster
module.sg.aws_security_group.dms
module.sg.aws_security_group.eks_node
module.sg.aws_security_group.rds
```

판정: EKS/ALB Controller를 제외한 기본 AWS DR 자원이 Terraform state에 들어와 있다.

## 3. VPC

Command:

```bash
AWS_PROFILE=project-admin aws ec2 describe-vpcs \
  --region ap-northeast-2 \
  --vpc-ids vpc-012db63a21da35d74 \
  --query 'Vpcs[].{VpcId:VpcId,State:State,Cidr:CidrBlock,IsDefault:IsDefault}' \
  --output table
```

Result:

```text
---------------------------------------------------------------------
|                           DescribeVpcs                            |
+--------------+------------+------------+--------------------------+
|     Cidr     | IsDefault  |   State    |          VpcId           |
+--------------+------------+------------+--------------------------+
|  10.20.0.0/16|  False     |  available |  vpc-012db63a21da35d74   |
+--------------+------------+------------+--------------------------+
```

판정: VPC는 `available`.

## 4. Subnets

Command:

```bash
AWS_PROFILE=project-admin aws ec2 describe-subnets \
  --region ap-northeast-2 \
  --filters Name=vpc-id,Values=vpc-012db63a21da35d74 \
  --query 'Subnets[].{SubnetId:SubnetId,AZ:AvailabilityZone,Cidr:CidrBlock,State:State,MapPublicIp:MapPublicIpOnLaunch,AvailableIps:AvailableIpAddressCount}' \
  --output table
```

Result:

```text
--------------------------------------------------------------------------------------------------------------
|                                               DescribeSubnets                                              |
+-----------------+---------------+----------------+--------------+------------+-----------------------------+
|       AZ        | AvailableIps  |     Cidr       | MapPublicIp  |   State    |          SubnetId           |
+-----------------+---------------+----------------+--------------+------------+-----------------------------+
|  ap-northeast-2a|  250          |  10.20.20.0/24 |  False       |  available |  subnet-01c68e65229dd6807   |
|  ap-northeast-2a|  250          |  10.20.0.0/24  |  True        |  available |  subnet-01f13816430dd3192   |
|  ap-northeast-2c|  251          |  10.20.11.0/24 |  False       |  available |  subnet-0640713279709de9b   |
|  ap-northeast-2c|  250          |  10.20.21.0/24 |  False       |  available |  subnet-0ad5f15aadf00631e   |
|  ap-northeast-2a|  251          |  10.20.10.0/24 |  False       |  available |  subnet-0dcab10445c4f20fe   |
|  ap-northeast-2c|  251          |  10.20.1.0/24  |  True        |  available |  subnet-0ac9f62e6a065059a   |
+-----------------+---------------+----------------+--------------+------------+-----------------------------+
```

판정: public/app/data subnet 6개 모두 `available`.

## 5. Internet Gateway

Command:

```bash
AWS_PROFILE=project-admin aws ec2 describe-internet-gateways \
  --region ap-northeast-2 \
  --filters Name=attachment.vpc-id,Values=vpc-012db63a21da35d74 \
  --query 'InternetGateways[].{InternetGatewayId:InternetGatewayId,Attachments:Attachments}' \
  --output json
```

Result:

```json
[
    {
        "InternetGatewayId": "igw-095790de21e23ee0d",
        "Attachments": [
            {
                "State": "available",
                "VpcId": "vpc-012db63a21da35d74"
            }
        ]
    }
]
```

판정: IGW는 VPC에 정상 attach.

## 6. NAT Gateway

Command:

```bash
AWS_PROFILE=project-admin aws ec2 describe-nat-gateways \
  --region ap-northeast-2 \
  --filter Name=vpc-id,Values=vpc-012db63a21da35d74 \
  --query 'NatGateways[].{NatGatewayId:NatGatewayId,State:State,SubnetId:SubnetId,PublicIp:NatGatewayAddresses[0].PublicIp,PrivateIp:NatGatewayAddresses[0].PrivateIp,AllocationId:NatGatewayAddresses[0].AllocationId}' \
  --output table
```

Result:

```text
------------------------------------------------
|              DescribeNatGateways             |
+---------------+------------------------------+
|  AllocationId |  eipalloc-04388099f32aaf102  |
|  NatGatewayId |  nat-08537bd5c6f96afee       |
|  PrivateIp    |  10.20.0.42                  |
|  PublicIp     |  15.164.83.144               |
|  State        |  available                   |
|  SubnetId     |  subnet-01f13816430dd3192    |
+---------------+------------------------------+
```

판정: NAT Gateway는 `available`.

## 7. S3 Gateway VPC Endpoint

Command:

```bash
AWS_PROFILE=project-admin aws ec2 describe-vpc-endpoints \
  --region ap-northeast-2 \
  --filters Name=vpc-id,Values=vpc-012db63a21da35d74 \
  --query 'VpcEndpoints[].{EndpointId:VpcEndpointId,Service:ServiceName,Type:VpcEndpointType,State:State,RouteTableIds:RouteTableIds}' \
  --output json
```

Result:

```json
[
    {
        "EndpointId": "vpce-06bd35008ec7064f6",
        "Service": "com.amazonaws.ap-northeast-2.s3",
        "Type": "Gateway",
        "State": "available",
        "RouteTableIds": [
            "rtb-0e29de398b13dc69c",
            "rtb-06488358fab9c5357",
            "rtb-07f9a5184c6431c05",
            "rtb-0218741ad34f22c2b"
        ]
    }
]
```

판정: S3 Gateway Endpoint는 `available`.

## 8. Route Tables

Command:

```bash
AWS_PROFILE=project-admin aws ec2 describe-route-tables \
  --region ap-northeast-2 \
  --filters Name=vpc-id,Values=vpc-012db63a21da35d74 \
  --query 'RouteTables[].{RouteTableId:RouteTableId,Associations:Associations[].SubnetId,Routes:Routes[].{Destination:DestinationCidrBlock,Gateway:GatewayId,Nat:NatGatewayId,Vgw:GatewayId,VpcEndpoint:VpcEndpointId,State:State}}' \
  --output json
```

Result 요약:

```text
rtb-0163d643665e4be21: public subnets -> 0.0.0.0/0 via igw-095790de21e23ee0d, active
rtb-07f9a5184c6431c05: app subnet 10.20.10.0/24 -> 0.0.0.0/0 via nat-08537bd5c6f96afee, 172.16.0.0/16 via vgw-0bf302f0f7bdfdec6, active
rtb-0e29de398b13dc69c: app subnet 10.20.11.0/24 -> 0.0.0.0/0 via nat-08537bd5c6f96afee, 172.16.0.0/16 via vgw-0bf302f0f7bdfdec6, active
rtb-0218741ad34f22c2b: data subnet 10.20.20.0/24 -> 172.16.0.0/16 via vgw-0bf302f0f7bdfdec6, active
rtb-06488358fab9c5357: data subnet 10.20.21.0/24 -> 172.16.0.0/16 via vgw-0bf302f0f7bdfdec6, active
S3 Gateway Endpoint route: app/data route tables에 active
```

판정: public/app/data 라우팅은 의도와 일치한다.

## 9. VPN Gateways and Tunnel

Command:

```bash
AWS_PROFILE=project-admin aws ec2 describe-customer-gateways \
  --region ap-northeast-2 \
  --customer-gateway-ids cgw-0305ac365b3e3f2f1 \
  --query 'CustomerGateways[].{CustomerGatewayId:CustomerGatewayId,State:State,IpAddress:IpAddress,BgpAsn:BgpAsn,Type:Type}' \
  --output table
```

Result:

```text
--------------------------------------------------------------------------------
|                           DescribeCustomerGateways                           |
+--------+-------------------------+------------------+------------+-----------+
| BgpAsn |    CustomerGatewayId    |    IpAddress     |   State    |   Type    |
+--------+-------------------------+------------------+------------+-----------+
|  65000 |  cgw-0305ac365b3e3f2f1  |  125.131.208.229 |  available |  ipsec.1  |
+--------+-------------------------+------------------+------------+-----------+
```

Command:

```bash
AWS_PROFILE=project-admin aws ec2 describe-vpn-gateways \
  --region ap-northeast-2 \
  --vpn-gateway-ids vgw-0bf302f0f7bdfdec6 \
  --query 'VpnGateways[].{VpnGatewayId:VpnGatewayId,State:State,Type:Type,VpcAttachments:VpcAttachments}' \
  --output json
```

Result:

```json
[
    {
        "VpnGatewayId": "vgw-0bf302f0f7bdfdec6",
        "State": "available",
        "Type": "ipsec.1",
        "VpcAttachments": [
            {
                "VpcId": "vpc-012db63a21da35d74",
                "State": "attached"
            }
        ]
    }
]
```

Command:

```bash
AWS_PROFILE=project-admin aws ec2 describe-vpn-connections \
  --region ap-northeast-2 \
  --vpn-connection-ids vpn-02820049cf4de6764 \
  --query 'VpnConnections[].{VpnConnectionId:VpnConnectionId,State:State,Type:Type,CustomerGatewayId:CustomerGatewayId,VpnGatewayId:VpnGatewayId,Routes:Routes,VgwTelemetry:VgwTelemetry[].{OutsideIp:OutsideIpAddress,Status:Status,Message:StatusMessage,AcceptedRouteCount:AcceptedRouteCount,LastStatusChange:LastStatusChange}}' \
  --output json
```

Result:

```json
[
    {
        "VpnConnectionId": "vpn-02820049cf4de6764",
        "State": "available",
        "Type": "ipsec.1",
        "CustomerGatewayId": "cgw-0305ac365b3e3f2f1",
        "VpnGatewayId": "vgw-0bf302f0f7bdfdec6",
        "Routes": [
            {
                "DestinationCidrBlock": "172.16.0.0/16",
                "State": "available"
            }
        ],
        "VgwTelemetry": [
            {
                "OutsideIp": "3.38.81.120",
                "Status": "UP",
                "Message": "",
                "AcceptedRouteCount": 1,
                "LastStatusChange": "2026-05-19T10:07:42+00:00"
            },
            {
                "OutsideIp": "43.203.75.154",
                "Status": "DOWN",
                "Message": "",
                "AcceptedRouteCount": 1,
                "LastStatusChange": "2026-05-19T08:10:36+00:00"
            }
        ]
    }
]
```

판정: VPN connection은 `available`, tunnel 1은 `UP`, tunnel 2는 `DOWN`. 최소 1개 tunnel이 올라와 있으므로 기본 VPN 연결은 성립했다.

## 10. RDS

Command:

```bash
AWS_PROFILE=project-admin aws rds describe-db-instances \
  --region ap-northeast-2 \
  --db-instance-identifier flaskapp-dr-mariadb \
  --query 'DBInstances[].{DBInstanceIdentifier:DBInstanceIdentifier,DBInstanceStatus:DBInstanceStatus,Engine:Engine,EngineVersion:EngineVersion,DBInstanceClass:DBInstanceClass,Endpoint:Endpoint.Address,Port:Endpoint.Port,MultiAZ:MultiAZ,PubliclyAccessible:PubliclyAccessible,StorageType:StorageType,AllocatedStorage:AllocatedStorage,BackupRetentionPeriod:BackupRetentionPeriod,PendingModifiedValues:PendingModifiedValues}' \
  --output json
```

Result:

```json
[
    {
        "DBInstanceIdentifier": "flaskapp-dr-mariadb",
        "DBInstanceStatus": "available",
        "Engine": "mariadb",
        "EngineVersion": "10.11.16",
        "DBInstanceClass": "db.t4g.small",
        "Endpoint": "flaskapp-dr-mariadb.cfy4qsq4kamy.ap-northeast-2.rds.amazonaws.com",
        "Port": 3306,
        "MultiAZ": false,
        "PubliclyAccessible": false,
        "StorageType": "gp3",
        "AllocatedStorage": 20,
        "BackupRetentionPeriod": 1,
        "PendingModifiedValues": {}
    }
]
```

판정: RDS MariaDB는 `available`, private only, Single-AZ.

## 11. RDS Parameter Group

Command:

```bash
AWS_PROFILE=project-admin aws rds describe-db-parameters \
  --region ap-northeast-2 \
  --db-parameter-group-name flaskapp-dr-mariadb-params \
  --source user \
  --query 'Parameters[].{Name:ParameterName,Value:ParameterValue,ApplyMethod:ApplyMethod,ApplyType:ApplyType,Source:Source}' \
  --output table
```

Result:

```text
------------------------------------------------------------------------
|                         DescribeDBParameters                         |
+-----------------+------------+--------------------+---------+--------+
|   ApplyMethod   | ApplyType  |       Name         | Source  | Value  |
+-----------------+------------+--------------------+---------+--------+
|  pending-reboot |  dynamic   |  binlog_checksum   |  user   |  NONE  |
|  pending-reboot |  dynamic   |  binlog_format     |  user   |  ROW   |
|  pending-reboot |  static    |  binlog_row_image  |  user   |  FULL  |
+-----------------+------------+--------------------+---------+--------+
```

판정: DMS CDC에 필요한 binlog parameter가 설정됐다. `pending-reboot`이므로 RDS 재부팅 전까지 실제 적용 여부는 별도 확인이 필요하다.

## 12. DMS Replication Instance

Command:

```bash
AWS_PROFILE=project-admin aws dms describe-replication-instances \
  --region ap-northeast-2 \
  --filters Name=replication-instance-id,Values=flaskapp-dr-dms \
  --query 'ReplicationInstances[].{Id:ReplicationInstanceIdentifier,Status:ReplicationInstanceStatus,Class:ReplicationInstanceClass,EngineVersion:EngineVersion,AllocatedStorage:AllocatedStorage,MultiAZ:MultiAZ,PubliclyAccessible:PubliclyAccessible,VpcSecurityGroups:VpcSecurityGroups,PendingModifiedValues:PendingModifiedValues}' \
  --output json
```

Result:

```json
[
    {
        "Id": "flaskapp-dr-dms",
        "Status": "available",
        "Class": "dms.t3.small",
        "EngineVersion": "3.5.4",
        "AllocatedStorage": 20,
        "MultiAZ": false,
        "PubliclyAccessible": false,
        "VpcSecurityGroups": [
            {
                "VpcSecurityGroupId": "sg-0872688d610c62dd6",
                "Status": "active"
            }
        ],
        "PendingModifiedValues": {}
    }
]
```

판정: DMS replication instance는 `available`.

## 13. DMS Endpoints

Command:

```bash
AWS_PROFILE=project-admin aws dms describe-endpoints \
  --region ap-northeast-2 \
  --filters Name=endpoint-id,Values=flaskapp-dr-source,flaskapp-dr-target \
  --query 'Endpoints[].{Id:EndpointIdentifier,Type:EndpointType,Status:Status,Engine:EngineName,Server:ServerName,Port:Port,Database:DatabaseName}' \
  --output table
```

Result:

```text
------------------------------------------------------------------------------------------------------------------------------------------------
|                                                               DescribeEndpoints                                                              |
+----------+----------+---------------------+-------+---------------------------------------------------------------------+---------+----------+
| Database | Engine   |         Id          | Port  |                               Server                                | Status  |  Type    |
+----------+----------+---------------------+-------+---------------------------------------------------------------------+---------+----------+
|  flaskapp|  mariadb |  flaskapp-dr-source |  3306 |  172.16.43.160                                                      |  active |  SOURCE  |
|  flaskapp|  mariadb |  flaskapp-dr-target |  3306 |  flaskapp-dr-mariadb.cfy4qsq4kamy.ap-northeast-2.rds.amazonaws.com  |  active |  TARGET  |
+----------+----------+---------------------+-------+---------------------------------------------------------------------+---------+----------+
```

판정: source/target endpoint 모두 `active`.

## 14. DMS Replication Task

Command:

```bash
AWS_PROFILE=project-admin aws dms describe-replication-tasks \
  --region ap-northeast-2 \
  --filters Name=replication-task-id,Values=flaskapp-dr-full-load-cdc \
  --query 'ReplicationTasks[].{Id:ReplicationTaskIdentifier,Status:Status,MigrationType:MigrationType,ReplicationInstanceArn:ReplicationInstanceArn,StopReason:StopReason,LastFailureMessage:LastFailureMessage,Stats:ReplicationTaskStats}' \
  --output json
```

Result:

```json
[
    {
        "Id": "flaskapp-dr-full-load-cdc",
        "Status": "ready",
        "MigrationType": "full-load-and-cdc",
        "ReplicationInstanceArn": "arn:aws:dms:ap-northeast-2:080252689380:rep:UIIXREKDJFBTLAPCBCY2WXQLQI",
        "StopReason": "Stop Reason NORMAL",
        "LastFailureMessage": null,
        "Stats": {
            "FullLoadProgressPercent": 0,
            "ElapsedTimeMillis": 0,
            "TablesLoaded": 0,
            "TablesLoading": 0,
            "TablesQueued": 0,
            "TablesErrored": 0
        }
    }
]
```

판정: DMS task는 의도대로 `ready`. 아직 replication은 시작하지 않았다.

## 15. S3 Bucket

Command:

```bash
AWS_PROFILE=project-admin aws s3api head-bucket \
  --bucket flaskapp-proddata-kosa-project-team3-snow-lai9z \
  --region ap-northeast-2 \
  --output json
```

Result:

```json
{
    "BucketArn": "arn:aws:s3:::flaskapp-proddata-kosa-project-team3-snow-lai9z",
    "BucketRegion": "ap-northeast-2",
    "AccessPointAlias": false
}
```

Command:

```bash
AWS_PROFILE=project-admin aws s3api get-bucket-versioning \
  --bucket flaskapp-proddata-kosa-project-team3-snow-lai9z \
  --region ap-northeast-2 \
  --output json
```

Result:

```json
{
    "Status": "Enabled"
}
```

Command:

```bash
AWS_PROFILE=project-admin aws s3api get-bucket-encryption \
  --bucket flaskapp-proddata-kosa-project-team3-snow-lai9z \
  --region ap-northeast-2 \
  --query 'ServerSideEncryptionConfiguration.Rules[].ApplyServerSideEncryptionByDefault' \
  --output json
```

Result:

```json
[
    {
        "SSEAlgorithm": "AES256"
    }
]
```

Command:

```bash
AWS_PROFILE=project-admin aws s3api get-public-access-block \
  --bucket flaskapp-proddata-kosa-project-team3-snow-lai9z \
  --region ap-northeast-2 \
  --query 'PublicAccessBlockConfiguration' \
  --output json
```

Result:

```json
{
    "BlockPublicAcls": true,
    "IgnorePublicAcls": true,
    "BlockPublicPolicy": true,
    "RestrictPublicBuckets": true
}
```

판정: S3 bucket은 존재하며 versioning, SSE-S3, public access block이 활성화되어 있다.

## 16. ECR

Command:

```bash
AWS_PROFILE=project-admin aws ecr describe-repositories \
  --region ap-northeast-2 \
  --repository-names flaskapp \
  --query 'repositories[].{Name:repositoryName,Uri:repositoryUri,CreatedAt:createdAt,ImageTagMutability:imageTagMutability,ScanOnPush:imageScanningConfiguration.scanOnPush}' \
  --output table
```

Result:

```text
--------------------------------------------------------------------------------------
|                                DescribeRepositories                                |
+---------------------+--------------------------------------------------------------+
|  CreatedAt          |  2026-05-08T16:17:31.404000+09:00                            |
|  ImageTagMutability |  IMMUTABLE                                                   |
|  Name               |  flaskapp                                                    |
|  ScanOnPush         |  True                                                        |
|  Uri                |  080252689380.dkr.ecr.ap-northeast-2.amazonaws.com/flaskapp  |
+---------------------+--------------------------------------------------------------+
```

Command:

```bash
AWS_PROFILE=project-admin aws ecr describe-images \
  --region ap-northeast-2 \
  --repository-name flaskapp \
  --query 'imageDetails[].{Digest:imageDigest,Tags:imageTags,PushedAt:imagePushedAt,Size:imageSizeInBytes}' \
  --output json
```

Result:

```json
[
    {
        "Digest": "sha256:dfdf05e61001a736077050791b3f2f33cd5074cded0244bebc2a6d811b7242ec",
        "Tags": null,
        "PushedAt": "2026-05-18T09:18:35.931000+09:00",
        "Size": 1375
    },
    {
        "Digest": "sha256:02143705ddee192529d4f94fa3c8cddbe0161b2150ed3e2c79acec1b84ec5bb5",
        "Tags": null,
        "PushedAt": "2026-05-18T09:18:35.940000+09:00",
        "Size": 98758221
    },
    {
        "Digest": "sha256:90346afc397b73424d4a07a7fb1579085bfcf9d281bc1d9e0a6bd83449b6648c",
        "Tags": [
            "latest",
            "d457c7e"
        ],
        "PushedAt": "2026-05-18T09:18:36.234000+09:00",
        "Size": 98758221
    }
]
```

판정: ECR repository는 존재하고 `latest`, `d457c7e` image가 있다.

## 17. IAM Roles for DMS

Command:

```bash
AWS_PROFILE=project-admin aws iam get-role \
  --role-name dms-vpc-role \
  --query 'Role.{RoleName:RoleName,Arn:Arn,CreateDate:CreateDate,Path:Path}' \
  --output json

AWS_PROFILE=project-admin aws iam list-attached-role-policies \
  --role-name dms-vpc-role \
  --query 'AttachedPolicies[].PolicyArn' \
  --output json
```

Result:

```json
{
    "RoleName": "dms-vpc-role",
    "Arn": "arn:aws:iam::080252689380:role/dms-vpc-role",
    "CreateDate": "2026-05-19T09:12:41+00:00",
    "Path": "/"
}
[
    "arn:aws:iam::aws:policy/service-role/AmazonDMSVPCManagementRole"
]
```

Command:

```bash
AWS_PROFILE=project-admin aws iam get-role \
  --role-name dms-cloudwatch-logs-role \
  --query 'Role.{RoleName:RoleName,Arn:Arn,CreateDate:CreateDate,Path:Path}' \
  --output json

AWS_PROFILE=project-admin aws iam list-attached-role-policies \
  --role-name dms-cloudwatch-logs-role \
  --query 'AttachedPolicies[].PolicyArn' \
  --output json
```

Result:

```json
{
    "RoleName": "dms-cloudwatch-logs-role",
    "Arn": "arn:aws:iam::080252689380:role/dms-cloudwatch-logs-role",
    "CreateDate": "2026-05-19T09:12:41+00:00",
    "Path": "/"
}
[
    "arn:aws:iam::aws:policy/service-role/AmazonDMSCloudWatchLogsRole"
]
```

판정: DMS 필수 IAM role 2개와 managed policy attachment가 정상 생성됐다.

## 18. Security Groups

Command:

```bash
AWS_PROFILE=project-admin aws ec2 describe-security-groups \
  --region ap-northeast-2 \
  --filters Name=vpc-id,Values=vpc-012db63a21da35d74 \
  --query 'SecurityGroups[].{GroupId:GroupId,GroupName:GroupName,Description:Description,IngressCount:length(IpPermissions),EgressCount:length(IpPermissionsEgress)}' \
  --output table
```

Result:

```text
------------------------------------------------------------------------------------------------------------------
|                                             DescribeSecurityGroups                                             |
+-----------------------------+--------------+-----------------------+--------------------------+----------------+
|         Description         | EgressCount  |        GroupId        |        GroupName         | IngressCount   |
+-----------------------------+--------------+-----------------------+--------------------------+----------------+
|  default VPC security group |  1           |  sg-0cdd59fdc8bd4e94c |  default                 |  1             |
|  DMS replication instance   |  1           |  sg-0872688d610c62dd6 |  flaskapp-dr-dms-sg      |  0             |
|  EKS control plane          |  1           |  sg-045382999d0717688 |  flaskapp-dr-cluster-sg  |  0             |
|  EKS worker nodes           |  1           |  sg-0fdf5164e5d218c1a |  flaskapp-dr-eks-node-sg |  2             |
|  ALB ingress                |  1           |  sg-0087fc4ea188f745e |  flaskapp-dr-alb-sg      |  2             |
|  RDS MariaDB                |  1           |  sg-0daadef872b663133 |  flaskapp-dr-rds-sg      |  1             |
+-----------------------------+--------------+-----------------------+--------------------------+----------------+
```

판정: 필요한 SG가 모두 생성됐다. EKS/ALB SG는 `dr_active=false`여도 향후 활성화 대비용으로 존재한다.

## 19. Observability

Command:

```bash
AWS_PROFILE=project-admin aws sns list-topics \
  --region ap-northeast-2 \
  --query 'Topics[].TopicArn' \
  --output text | tr '\t' '\n' | grep 'flaskapp-dr-alarms'
```

Result:

```text
arn:aws:sns:ap-northeast-2:080252689380:flaskapp-dr-alarms
```

Command:

```bash
AWS_PROFILE=project-admin aws sns list-subscriptions-by-topic \
  --region ap-northeast-2 \
  --topic-arn arn:aws:sns:ap-northeast-2:080252689380:flaskapp-dr-alarms \
  --query 'Subscriptions[].{Protocol:Protocol,SubscriptionArn:SubscriptionArn}' \
  --output json
```

Result:

```json
[
    {
        "Protocol": "email",
        "SubscriptionArn": "PendingConfirmation"
    }
]
```

판정: SNS topic은 생성됨. Email subscription은 아직 confirm 대기.

Command:

```bash
AWS_PROFILE=project-admin aws cloudwatch describe-alarms \
  --region ap-northeast-2 \
  --alarm-name-prefix flaskapp-dr \
  --query 'MetricAlarms[].{AlarmName:AlarmName,StateValue:StateValue,MetricName:MetricName,Namespace:Namespace,Threshold:Threshold,EvaluationPeriods:EvaluationPeriods}' \
  --output table
```

Result:

```text
----------------------------------------------------------------------------------------------------------------------------
|                                                      DescribeAlarms                                                      |
+------------------------------+--------------------+-------------------+------------+--------------------+----------------+
|           AlarmName          | EvaluationPeriods  |    MetricName     | Namespace  |    StateValue      |   Threshold    |
+------------------------------+--------------------+-------------------+------------+--------------------+----------------+
|  flaskapp-dr-dms-cdc-lag     |  5                 |  CDCLatencyTarget |  AWS/DMS   |  INSUFFICIENT_DATA |  300.0         |
|  flaskapp-dr-rds-storage-low |  2                 |  FreeStorageSpace |  AWS/RDS   |  OK                |  2147483648.0  |
+------------------------------+--------------------+-------------------+------------+--------------------+----------------+
```

판정: alarm 2개 생성. DMS는 아직 task 미실행이라 `INSUFFICIENT_DATA`가 정상 범위다.

Command:

```bash
AWS_PROFILE=project-admin aws cloudwatch list-dashboards \
  --region ap-northeast-2 \
  --dashboard-name-prefix flaskapp-dr \
  --query 'DashboardEntries[].{Name:DashboardName,LastModified:LastModified,Size:Size}' \
  --output table
```

Result:

```text
----------------------------------------------------------------
|                        ListDashboards                        |
+----------------------------+-------------------------+-------+
|        LastModified        |          Name           | Size  |
+----------------------------+-------------------------+-------+
|  2026-05-19T18:27:53+09:00 |  flaskapp-dr-readiness  |  1216 |
+----------------------------+-------------------------+-------+
```

판정: CloudWatch dashboard 생성 완료.

## 20. dr_active=false Verification

Command:

```bash
AWS_PROFILE=project-admin aws ec2 describe-instances \
  --region ap-northeast-2 \
  --filters Name=vpc-id,Values=vpc-012db63a21da35d74 \
  --query 'Reservations[].Instances[].{InstanceId:InstanceId,State:State.Name,Type:InstanceType,PrivateIp:PrivateIpAddress,PublicIp:PublicIpAddress}' \
  --output json
```

Result:

```json
[]
```

Command:

```bash
AWS_PROFILE=project-admin aws eks list-clusters \
  --region ap-northeast-2 \
  --query 'clusters' \
  --output json
```

Result:

```json
[]
```

Command:

```bash
AWS_PROFILE=project-admin aws elbv2 describe-load-balancers \
  --region ap-northeast-2 \
  --query 'LoadBalancers[].{Name:LoadBalancerName,VpcId:VpcId,State:State.Code,DNS:DNSName,Type:Type,Scheme:Scheme}' \
  --output json
```

Result:

```json
[]
```

Command:

```bash
AWS_PROFILE=project-admin aws autoscaling describe-auto-scaling-groups \
  --region ap-northeast-2 \
  --query 'AutoScalingGroups[].{Name:AutoScalingGroupName,Desired:DesiredCapacity,Min:MinSize,Max:MaxSize,Instances:Instances[].InstanceId}' \
  --output json
```

Result:

```json
[]
```

판정: EC2 instance, EKS cluster, Load Balancer, Auto Scaling Group 모두 없음. `dr_active=false` 운영 의도와 일치한다.

## 다음 조치

1. SNS confirmation email을 승인한다.
2. VPN tunnel 2도 필요하면 pfSense에서 두 번째 tunnel 설정을 마저 맞춘다.
3. RDS binlog parameter 실제 반영이 필요하면 재부팅 후 DB 내부 값 확인을 수행한다.
4. DMS endpoint connection test를 실행한다.
5. DMS task를 시작하기 전 온프렘 MariaDB 권한, binlog 설정, 방화벽을 확인한다.

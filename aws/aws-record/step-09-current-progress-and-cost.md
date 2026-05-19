# Step 09 - Current Progress and Cost

기준일: 2026-05-19

목표: `dr_active=false` 기준으로 현재까지 실제 생성된 AWS DR 자원과 예상 비용을 정리한다.

## 현재 적용 진도

`terraform output` 기준으로 EKS/ALB를 제외한 기본 AWS DR 기반은 생성됐다.

| 영역 | 상태 | 메모 |
|---|---|---|
| Network/VPC | 완료 | VPC, subnet, route table, IGW, NAT Gateway, S3 Gateway Endpoint 생성 |
| Site-to-Site VPN | 완료 | AWS 쪽 CGW/VGW/VPN 생성 완료. pfSense 설정 전이므로 tunnel은 `DOWN` 예상 |
| S3/ECR | 완료 | 기존 bucket/repository를 Terraform 관리 대상으로 사용 |
| IAM | 완료 | DMS service role 생성 |
| Security Group | 완료 | ALB/EKS/RDS/DMS용 SG 생성 |
| RDS | 완료 | MariaDB Single-AZ 생성 |
| DMS | 완료 | replication instance, endpoint, task 생성 |
| DMS task 실행 | 대기 | `ready` 확인. VPN 연결 전이므로 자동 시작하지 않음 |
| Observability | 완료 | SNS topic/subscription, CloudWatch alarms/dashboard 생성 |
| EKS/ALB | 미생성 | `dr_active=false`, `eks_cluster_name = ""` 정상 |

## 주요 출력값

| 항목 | 값 |
|---|---|
| VPC ID | `vpc-012db63a21da35d74` |
| VPN Connection ID | `vpn-02820049cf4de6764` |
| RDS endpoint | `flaskapp-dr-mariadb.cfy4qsq4kamy.ap-northeast-2.rds.amazonaws.com` |
| DMS instance ID | `flaskapp-dr-dms` |
| DMS task ID | `flaskapp-dr-full-load-cdc` |
| DMS task status | `ready` |
| EKS cluster name | empty string, 정상 |
| Service domain | `flaskapp.team.snow.internal` |

## 현재 예상 비용

가정:

- 리전: `ap-northeast-2` (Asia Pacific Seoul)
- 기간: 30일 = 720시간
- 트래픽은 거의 없다고 가정
- EKS/ALB는 아직 미생성이라 제외
- S3/ECR 저장 데이터 용량은 현재 알 수 없어 제외
- CloudWatch free tier 적용 여부는 계정 상황에 따라 달라질 수 있어 보수적으로 과금 기준에 포함

| 항목 | 수량 | 단가 | 시간당 | 1일 | 30일 |
|---|---:|---:|---:|---:|---:|
| NAT Gateway hourly | 1 | `$0.059/hour` | `$0.059` | `$1.416` | `$42.48` |
| Site-to-Site VPN connection | 1 | `$0.050/hour` | `$0.050` | `$1.200` | `$36.00` |
| Public IPv4 - NAT EIP | 1 | `$0.005/hour` | `$0.005` | `$0.120` | `$3.60` |
| Public IPv4 - VPN tunnel outside IPs | 2 | `$0.005/hour` | `$0.010` | `$0.240` | `$7.20` |
| RDS MariaDB `db.t4g.small` Single-AZ | 1 | `$0.051/hour` | `$0.051` | `$1.224` | `$36.72` |
| RDS gp3 storage | 20 GB | `$0.131/GB-month` | 약 `$0.004` | 약 `$0.087` | `$2.62` |
| DMS `dms.t3.small` Single-AZ | 1 | `$0.056/hour` | `$0.056` | `$1.344` | `$40.32` |
| DMS GP2 storage | 20 GB | `$0.131/GB-month` | 약 `$0.004` | 약 `$0.087` | `$2.62` |
| CloudWatch dashboard | 1 | `$3.00/month` | 약 `$0.004` | 약 `$0.100` | `$3.00` |
| CloudWatch alarms | 2 | `$0.10/alarm-month` | 약 `$0.0003` | 약 `$0.007` | `$0.20` |

고정성 비용 합계:

```text
약 $0.243/hour
약 $5.83/day
약 $174.76 / 30 days
```

주의:

- DMS task가 `ready`여도 DMS replication instance 자체는 켜져 있으므로 시간당 비용이 계속 발생한다.
- VPN tunnel이 `DOWN`이어도 Site-to-Site VPN connection은 provisioned 상태라 시간당 비용이 발생한다.
- RDS parameter group의 `pending-reboot` 값은 RDS 재부팅 전까지 실제 DB에 완전히 반영되지 않을 수 있다.
- DMS storage는 Terraform 설정값 20 GB 기준으로 계산했다. AWS 과금에서 최소 스토리지 단위가 더 크게 적용되면 월 몇 달러 정도 증가할 수 있다.
- NAT Gateway data processing은 별도다. 서울 리전 기준 약 `$0.059/GB`로 잡고, 인터넷 방향 data transfer out은 별도 과금될 수 있다.
- S3 bucket과 ECR repository는 자체 hourly 비용은 없지만 저장 용량, 요청, 데이터 송신량에 따라 비용이 붙는다.
- SNS email 알림은 지금 규모에서는 사실상 미미하나, 대량 발송 시 별도 비용이 붙을 수 있다.

## 다음 진행

1. pfSense에 AWS VPN XML을 안전하게 전달한다.
2. pfSense 쪽 tunnel을 구성한다.
3. AWS VPN telemetry에서 최소 1개 tunnel `UP`을 확인한다.
4. DMS source/target endpoint connection test를 수행한다.
5. DMS task를 `step_5_start_dms` 명령으로 시작한다.
6. DR 리허설 시점에만 `dr_active=true`로 EKS/ALB를 생성한다.

## 참고 가격 출처

- AWS Price List API: `AmazonRDS`, `AWSDatabaseMigrationSvc`, `AmazonVPC` Seoul region price files
- AWS VPN pricing: https://aws.amazon.com/vpn/pricing/
- Amazon VPC pricing: https://aws.amazon.com/vpc/pricing/
- NAT Gateway pricing note: https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-pricing.html
- Amazon RDS pricing: https://aws.amazon.com/rds/pricing/
- AWS DMS pricing: https://aws.amazon.com/dms/pricing/
- Amazon CloudWatch pricing: https://aws.amazon.com/cloudwatch/pricing/

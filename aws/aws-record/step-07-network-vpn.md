# Step 07 - Network and VPN

목표: 전체 DR stack 적용 전에 AWS 쪽 Site-to-Site VPN 기반을 먼저 생성하고, pfSense 담당자에게 넘길 설정 정보를 확보한다.

## 생성 범위

- VPC
- public/app/data subnets
- route tables
- Internet Gateway
- NAT Gateway 1개
- S3 Gateway VPC Endpoint
- Customer Gateway
- Virtual Private Gateway
- Site-to-Site VPN Connection
- VPN static route to `172.16.0.0/16`

## 변수 기준

| 변수 | 값 | 의미 |
|---|---|---|
| `vpc_cidr` | `10.20.0.0/16` | AWS DR VPC CIDR |
| `onprem_cidr` | `172.16.0.0/16` | 온프렘 사설망 CIDR |
| `pfsense_public_ip` | `125.131.208.229` | AWS Customer Gateway public IP |
| `customer_gateway_asn` | `65000` | Customer Gateway ASN |
| `nat_gateway_count` | `1` | 평시 NAT Gateway 1개 |

## Apply 결과

```text
Apply complete! Resources: 26 added, 0 changed, 0 destroyed.
```

| 자원 | ID |
|---|---|
| VPC | `vpc-012db63a21da35d74` |
| Customer Gateway | `cgw-0305ac365b3e3f2f1` |
| Virtual Private Gateway | `vgw-0bf302f0f7bdfdec6` |
| Site-to-Site VPN Connection | `vpn-02820049cf4de6764` |
| NAT Gateway | `nat-08537bd5c6f96afee` |
| S3 Gateway VPC Endpoint | `vpce-06bd35008ec7064f6` |

## pfSense 전달 파일

pfSense 설정용 XML은 다음 명령으로 다시 생성할 수 있다.

```bash
VPN_ID="vpn-02820049cf4de6764"

aws ec2 describe-vpn-connections \
  --region ap-northeast-2 \
  --vpn-connection-ids "$VPN_ID" \
  --query 'VpnConnections[0].CustomerGatewayConfiguration' \
  --output text > record/step-07-vpn-customer-gateway-configuration.xml
```

주의: 이 XML에는 VPN tunnel pre-shared key가 포함된다. Git commit, 단체 채팅방, 공개 문서에 올리지 않는다.

Tunnel 요약:

| Tunnel | Customer Gateway outside IP | AWS outside IP | Customer inside IP | AWS inside IP | Inside CIDR |
|---|---|---|---|---|---|
| 1 | `125.131.208.229` | `3.38.81.120` | `169.254.172.74` | `169.254.172.73` | `169.254.172.72/30` |
| 2 | `125.131.208.229` | `43.203.75.154` | `169.254.154.190` | `169.254.154.189` | `169.254.154.188/30` |

기본 파라미터:

| 항목 | 값 |
|---|---|
| VPN type | `ipsec.1` |
| Routing | Static route (`NoBGPVPNConnection`) |
| Remote/on-prem route | `172.16.0.0/16` |
| IKE auth | `sha1` |
| IKE encryption | `aes-128-cbc` |
| IKE lifetime | `28800` |
| IKE PFS | `group2` |
| IPsec protocol | `esp` |
| IPsec auth | `hmac-sha1-96` |
| IPsec encryption | `aes-128-cbc` |
| IPsec lifetime | `3600` |
| IPsec PFS | `group2` |
| DPD | interval `10`, retries `3` |

## 현재 상태 예상 비용

대상 리전은 `ap-northeast-2` (Asia Pacific Seoul)이다. 트래픽이 거의 없고 현재 생성된 network/VPN 자원만 유지한다고 가정한다.

| 항목 | 수량 | 단가 가정 | 시간당 | 1일 | 30일 |
|---|---:|---:|---:|---:|---:|
| NAT Gateway hourly | 1 | `$0.059/hour` | `$0.059` | `$1.416` | `$42.48` |
| Site-to-Site VPN connection | 1 | `$0.050/hour` | `$0.050` | `$1.200` | `$36.00` |
| Public IPv4 - NAT EIP | 1 | `$0.005/hour` | `$0.005` | `$0.120` | `$3.60` |
| Public IPv4 - VPN tunnel outside IPs | 2 | `$0.005/hour` | `$0.010` | `$0.240` | `$7.20` |
| S3 Gateway VPC Endpoint | 1 | 추가 요금 없음 | `$0.000` | `$0.000` | `$0.00` |
| VPC/subnet/route table/IGW/VGW/CGW | - | 보통 별도 hourly 없음 | `$0.000` | `$0.000` | `$0.00` |

고정성 비용 합계:

```text
약 $0.124/hour
약 $2.98/day
약 $89.28 / 30 days
```

변동 비용:

- NAT Gateway data processing: 서울 리전 기준 약 `$0.059/GB`로 잡는다.
- NAT Gateway를 통해 인터넷으로 나가는 트래픽은 별도 AWS data transfer out 요금이 붙을 수 있다.
- Site-to-Site VPN으로 AWS에서 온프렘 방향으로 나가는 트래픽도 EC2 data transfer out 기준의 데이터 송신 요금이 붙을 수 있다.
- 현재 RDS/DMS/EKS/ALB는 아직 미적용이므로 이 estimate에는 포함하지 않는다.

비용 메모:

- 지금 상태에서는 NAT Gateway와 VPN Connection이 비용의 대부분이다.
- pfSense 설정 대기 시간이 길어지면 하루 약 `$3`씩 누적된다고 보면 된다.
- 장기 대기해야 하면 `terraform destroy -target=module.network` 또는 별도 cleanup 절차를 검토한다. 단, VPN 설정을 계속 맞추는 중이면 유지한다.

## 상태 확인

pfSense 설정 전에는 tunnel status가 `DOWN`인 것이 정상이다. pfSense 설정 후 최소 하나 이상의 tunnel이 `UP`이어야 한다.

```bash
VPN_ID="vpn-02820049cf4de6764"

aws ec2 describe-vpn-connections \
  --region ap-northeast-2 \
  --vpn-connection-ids "$VPN_ID" \
  --query 'VpnConnections[0].VgwTelemetry[*].{OutsideIp:OutsideIpAddress,Status:Status,Message:StatusMessage}' \
  --output table
```

## 다음 진행

1. pfSense 설정용 XML을 다시 생성해 안전하게 전달한다.
2. pfSense에서 두 tunnel 중 최소 1개를 먼저 구성한다.
3. AWS telemetry에서 tunnel `UP` 확인.
4. AWS VPC `10.20.0.0/16` ↔ 온프렘 `172.16.0.0/16` 라우팅/방화벽 확인.
5. 이후 RDS/DMS 단계로 진행한다.

## 참고한 AWS 가격 문서

- AWS VPN pricing: https://aws.amazon.com/vpn/pricing/
- Amazon VPC pricing: https://aws.amazon.com/vpc/pricing/
- NAT Gateway pricing note: https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-pricing.html
- S3 Gateway Endpoint no additional charge: https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html

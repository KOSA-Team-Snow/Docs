# Step 12. pfSense 기반 AWS Site-to-Site VPN 터널 구성

작성일: 2026-05-20

## 1. 목적

이번 단계의 핵심 작업은 **온프렘 pfSense를 IPsec VPN 종단 장비로 설정하여 AWS DR VPC와 온프렘 내부망 사이의 Site-to-Site VPN 터널을 실제로 수립하는 것**이다.

즉, AWS 콘솔에서 VPN 리소스를 생성하는 것만이 아니라, 온프렘 쪽 pfSense에서 AWS VPN Tunnel Outside IP로 IPsec 연결을 시작하고, 다음 통신 경로를 열어주는 것이 목표다.

```text
AWS VPC 10.20.0.0/16
-> AWS VGW / Site-to-Site VPN
-> Internet
-> Omada / 온프렘 라우터
-> pfSense IPsec Tunnel
-> 온프렘 내부망 172.16.0.0/16
```

이번 작업의 성공 기준은 테스트 EC2에서 온프렘 주요 노드까지 실제 통신이 되는 것이다.

```text
AWS 테스트 EC2 -> 온프렘 라우터 ping 성공
AWS 테스트 EC2 -> pfSense ping 성공
AWS 테스트 EC2 -> MariaDB ping 성공
AWS 테스트 EC2 -> MariaDB 3306 TCP 성공
```

## 2. 구성 범위

| 항목                       | 값                      |
| -------------------------- | ----------------------- |
| AWS Region                 | `ap-northeast-2`        |
| AWS DR VPC                 | `10.20.0.0/16`          |
| 온프렘 CIDR                | `172.16.0.0/16`         |
| AWS VPN                    | `vpn-02820049cf4de6764` |
| VGW                        | `vgw-0bf302f0f7bdfdec6` |
| Customer Gateway Public IP | `125.131.208.229`       |
| pfSense WAN IP             | `172.16.30.3`           |
| 온프렘 라우터              | `172.16.30.1`           |
| MariaDB                    | `172.16.43.160:3306`    |
| 테스트 EC2                 | `10.20.10.194`          |

## 3. VPN 터널 정보

AWS VPN은 두 개의 tunnel을 제공한다. 이번 단계에서는 Tunnel 1을 먼저 구성하고 검증했다.

| Tunnel   | AWS Outside IP  | Customer Inside IP | AWS Inside IP     | Inside CIDR          |
| -------- | --------------- | ------------------ | ----------------- | -------------------- |
| Tunnel 1 | `3.38.81.120`   | `169.254.172.74`   | `169.254.172.73`  | `169.254.172.72/30`  |
| Tunnel 2 | `43.203.75.154` | `169.254.154.190`  | `169.254.154.189` | `169.254.154.188/30` |

> Pre-shared key는 AWS Customer Gateway Configuration XML에 포함되지만, 문서에는 기록하지 않는다.

## 4. pfSense IPsec Phase 1 구성

pfSense에서 Tunnel 1 Phase 1을 생성하여 AWS VPN endpoint `3.38.81.120`으로 IPsec 연결을 시작하도록 설정했다.

```text
VPN -> IPsec -> Tunnels -> Add P1
```

Tunnel 1 Phase 1 주요 설정:

```text
Description: vpn-db-tunnel-1
Key Exchange Version: IKEv2
Internet Protocol: IPv4
Interface: WAN
Remote Gateway: 3.38.81.120
Authentication Method: Mutual PSK
My Identifier: IP Address / 125.131.208.229
Peer Identifier: Peer IP Address / 3.38.81.120
NAT Traversal: Enable
Dead Peer Detection: Enable
DPD Delay: 10
```

IKE proposal은 AWS VPN XML 값과 일치시킨다.

```text
Encryption: AES-128-CBC
Authentication/Hash: SHA1
DH Group: 2
Lifetime: 28800
```

주의:

```text
AWS XML과 pfSense 설정값이 일치해야 한다.
초기 연결 단계에서는 임의로 AES-256, SHA-256, group14 등으로 바꾸지 않는다.
```

## 5. pfSense IPsec Phase 2 구성

Phase 2에서는 실제로 VPN 터널 안에서 통신할 사설망 대역을 지정했다.

```text
VPN -> IPsec -> Tunnels -> vpn-db-tunnel-1 -> Add P2
```

Tunnel 1 Phase 2 주요 설정:

```text
Description: vpn-db-tunnel-1-p2
Mode: Tunnel IPv4
Local Network: 172.16.0.0/16
Remote Network: 10.20.0.0/16
NAT/BINAT Translation: None
Protocol: ESP
Encryption: AES-128-CBC
Hash: SHA1
PFS Group: 2
Lifetime: 3600
```

의미:

```text
온프렘 내부망 172.16.0.0/16 과
AWS VPC 10.20.0.0/16 을
IPsec tunnel 안에서 서로 통신시킨다.
```

## 6. pfSense IPsec 터널 연결

Phase 1과 Phase 2 설정을 저장한 뒤, pfSense에서 Tunnel 1 연결을 시작했다.

```text
Status -> IPsec
-> vpn-db-tunnel-1
-> Connect P1 and P2s
```

성공 시 확인 기준:

```text
IKE_SA established
CHILD_SA established
Tunnel Status: Established / Connected
```

이번 작업에서는 pfSense가 AWS VPN endpoint와 UDP `500`, `4500`으로 정상 통신하는 것을 확인했고, Tunnel 1이 정상 수립되었다.

```text
pfSense 172.16.30.3 -> AWS Tunnel 1 3.38.81.120 UDP 500
pfSense 172.16.30.3 -> AWS Tunnel 1 3.38.81.120 UDP 4500
```

## 7. pfSense IPsec 방화벽 룰

터널이 올라와도 pfSense 방화벽에서 IPsec 트래픽을 막으면 실제 통신은 실패한다. 따라서 IPsec interface에 AWS VPC에서 온프렘 방향으로 들어오는 트래픽을 허용하는 rule을 추가했다.

```text
Firewall -> Rules -> IPsec
```

초기 검증용 rule:

```text
Action: Pass
Interface: IPsec
Protocol: Any
Source: 10.20.0.0/16
Destination: 172.16.0.0/16
Description: TEMP Allow AWS VPC to onprem over IPsec
```

운영 시에는 필요한 대상과 포트로 축소한다.

```text
ICMP:
10.20.0.0/16 -> 172.16.0.0/16

MariaDB:
DMS/Data subnet 또는 DMS replication instance IP -> 172.16.43.160/32 TCP 3306
```

## 8. 온프렘 라우팅

온프렘 내부망에서 AWS VPC로 응답하려면, AWS 대역 `10.20.0.0/16`으로 가는 트래픽이 pfSense로 전달되어야 한다.

온프렘 라우터에 필요한 static route:

```text
Destination IP: 10.20.0.0
Subnet Mask: 255.255.0.0
Next Hop: 172.16.30.3
Interface: LAN
```

의미:

```text
AWS VPC 10.20.0.0/16으로 가는 트래픽은
AWS VPN 터널을 가진 pfSense 172.16.30.3으로 보내라.
```

여기서 `Interface: LAN`은 목적지가 LAN이라는 뜻이 아니라, next hop인 pfSense `172.16.30.3`에 도달하기 위한 출구가 LAN이라는 뜻이다.

## 9. AWS 라우팅 확인

AWS app/data subnet route table에는 온프렘 대역으로 가는 경로가 필요하다.

```text
Destination: 172.16.0.0/16
Target: VGW
```

이번 테스트 EC2 `10.20.10.194`는 app subnet에 위치했고, 해당 subnet의 route table에는 다음 경로가 존재했다.

```text
10.20.0.0/16 -> local
172.16.0.0/16 -> vgw-0bf302f0f7bdfdec6
```

따라서 AWS에서 온프렘으로 나가는 경로는 정상으로 판단했다.

주의:

```text
public subnet과 main route table에는 172.16.0.0/16 -> VGW 경로가 없었다.
VPN 테스트용 EC2는 app/data subnet처럼 VGW route가 있는 subnet에 위치해야 한다.
```

## 10. 검증 결과

AWS VPC 내부 테스트 EC2 `10.20.10.194`에서 온프렘 주요 노드로 통신을 확인했다.

검증 명령:

```bash
ping -c 4 172.16.30.1
ping -c 4 172.16.30.3
ping -c 4 172.16.43.160
nc -zv 172.16.43.160 3306
```

검증 결과:

| Source             | Destination                  | Test | Result |
| ------------------ | ---------------------------- | ---- | ------ |
| EC2 `10.20.10.194` | 라우터 `172.16.30.1`         | ICMP | 성공   |
| EC2 `10.20.10.194` | pfSense `172.16.30.3`        | ICMP | 성공   |
| EC2 `10.20.10.194` | MariaDB `172.16.43.160`      | ICMP | 성공   |
| EC2 `10.20.10.194` | MariaDB `172.16.43.160:3306` | TCP  | 성공   |

이 결과를 기준으로, pfSense 기반 IPsec tunnel 구성은 성공으로 본다.

```text
AWS VPC 10.20.0.0/16
<-> pfSense IPsec Tunnel
<-> 온프렘 172.16.0.0/16

통신 성공
```

## 11. 후속 작업

Tunnel 1 통신 검증이 완료되었으므로, 이후 동일한 방식으로 Tunnel 2를 구성할 수 있다.

Tunnel 2 기본값:

```text
Remote Gateway: 43.203.75.154
Customer Inside IP: 169.254.154.190
AWS Inside IP: 169.254.154.189
Inside CIDR: 169.254.154.188/30
```

DMS Source Endpoint는 네트워크 timeout 단계는 통과했고, MariaDB 인증 단계까지 도달했다.

현재 남은 오류는 DB 계정/권한 문제다.

```text
Access denied for user 'dms_user'@'10.20.20.94'
```

후속 작업:

```text
1. MariaDB dms_user host 조건 확인
2. DMS replication instance source IP 확인
3. dms_user 비밀번호 확인
4. DMS에 필요한 최소 권한 부여
5. DMS Source Endpoint connection test 재수행
6. 검증 완료 후 테스트 EC2 10.20.10.194 삭제
```

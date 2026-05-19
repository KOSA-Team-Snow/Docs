# Bind9 내부 DNS 설계

## 목적

이 문서는 내부망에서 Route 53 역할을 모의하기 위한 Bind9 DNS 전용 VM 구성과 DR 전환 데모 기준을 정의한다.

ADR-003에서는 공인 IP를 받을 수 없는 실습 환경 제약 때문에 내부 Bind9 DNS를 사용하기로 결정했다. 이 문서는 그 결정에 따른 구체적인 VM, DNS zone, 방화벽, 검증 기준을 다룬다.

## Bind9 VM 상세 설계

내부 DNS 전환 데모를 위해 Bind9 전용 VM을 추가한다.

| 항목 | 값 |
|---|---|
| VM 이름 | `dns` |
| 역할 | 내부 DNS / Route 53 전환 모의 |
| OS | Ubuntu 24.04 |
| IP | `172.16.41.53/24` |
| Gateway | `172.16.41.1` |
| 배치 VLAN | VLAN 10 Public / 데모 PC 접근 구간 |
| DNS 서비스 포트 | UDP/TCP `53` |
| 관리 포트 | SSH `22` |
| 관리 도메인 | `team.snow.internal` |
| 서비스 도메인 | `flaskapp.team.snow.internal` |
| 정상 운영 응답 | `172.16.42.99` |
| DR 전환 응답 | AWS ALB DNS CNAME |

`dns` VM은 데모 PC가 직접 DNS 서버로 지정할 수 있도록 VLAN 10 Public 대역의 `172.16.41.53`을 사용한다.

정상 운영 시 `flaskapp.team.snow.internal`은 HAProxy VIP인 `172.16.42.99`로 응답한다.

```text
flaskapp.team.snow.internal -> 172.16.42.99
```

DR 전환 시 같은 이름을 AWS ALB DNS로 연결한다.

```text
flaskapp.team.snow.internal -> k8s-flaskapp-xxxx.ap-northeast-2.elb.amazonaws.com
```

AWS ALB는 고정 IP가 아니라 DNS 이름을 제공하므로, Bind9에서는 DR 전환 시 A 레코드 대신 CNAME 레코드를 사용한다.

## DNS 질의와 서비스 접속 흐름

`dns 172.16.41.53`은 서비스 트래픽을 HAProxy로 전달하는 프록시가 아니다. Bind9는 도메인 이름에 대한 DNS 응답만 제공한다.

따라서 `flaskapp.team.snow.internal` 접속 흐름은 DNS 질의와 HTTP/HTTPS 접속으로 분리된다.

```text
1. 데모 PC가 DNS 서버 172.16.41.53에 질의한다.

   flaskapp.team.snow.internal의 IP는 무엇인가?

2. Bind9가 HAProxy VIP를 응답한다.

   flaskapp.team.snow.internal = 172.16.42.99

3. 데모 PC가 응답받은 IP로 직접 HTTP/HTTPS 요청을 보낸다.

   데모 PC -> 172.16.42.99:80/443
```

전체 흐름은 다음과 같다.

```text
데모 PC
  ├─ DNS 질의 ──> Bind9 DNS 172.16.41.53:53
  │                └─ 응답: flaskapp.team.snow.internal = 172.16.42.99
  │
  └─ HTTP/HTTPS 요청 ─> HAProxy VIP 172.16.42.99:80/443
                         └─ Worker NodePort
                            └─ NGINX Ingress Controller
                               └─ FlaskApp Service
                                  └─ FlaskApp Pod
```

즉 `172.16.41.53 -> 172.16.42.99`로 웹 트래픽이 전달되는 구조가 아니다. 데모 PC가 먼저 `172.16.41.53`에 DNS를 질의하고, 이후에는 DNS 응답으로 받은 `172.16.42.99`에 직접 접속한다.

이 때문에 방화벽 정책도 DNS 질의와 서비스 접속을 각각 허용해야 한다.

```text
데모 PC / VLAN 10 -> dns 172.16.41.53:53
데모 PC / VLAN 10 -> HAProxy VIP 172.16.42.99:80/443
```

## Bind9 구성 계획

`dns` VM에는 다음 패키지를 설치한다.

```bash
sudo apt update
sudo apt install -y bind9 bind9utils dnsutils qemu-guest-agent chrony
sudo systemctl enable --now bind9
sudo systemctl enable --now qemu-guest-agent
sudo systemctl enable --now chrony
```

Bind9는 내부 실습망에서만 질의를 허용한다.

허용 대상 대역은 다음과 같다.

| 대역 | 용도 |
|---|---|
| `172.16.41.0/24` | VLAN 10 Public / 데모 PC 접근 구간 |
| `172.16.42.0/24` | VLAN 20 DMZ / HAProxy, Keepalived |
| `172.16.43.0/24` | VLAN 30 Internal / Kubernetes 노드 |
| `172.16.44.0/24` | VLAN 40 Admin / Bastion, 관리자 접근 |

외부 DNS 조회가 필요한 경우를 위해 forwarder를 설정한다.

```conf
forwarders {
    1.1.1.1;
    8.8.8.8;
};
```

이는 DR 전환 시 `flaskapp.team.snow.internal`이 AWS ALB DNS를 CNAME으로 가리키는 경우, Bind9가 ALB DNS 이름을 외부 DNS로 해석할 수 있어야 하기 때문이다.

## Zone 구성 기준

`team.snow.internal` zone을 Bind9 master zone으로 생성한다.

```conf
zone "team.snow.internal" {
    type master;
    file "/etc/bind/zones/db.team.snow.internal";
};
```

정상 운영용 zone 파일의 기본 레코드는 다음과 같다.

```dns
$TTL 60
@   IN SOA dns.team.snow.internal. admin.team.snow.internal. (
        2026051901
        60
        60
        604800
        60 )

    IN NS  dns.team.snow.internal.

dns        IN A  172.16.41.53
flaskapp   IN A  172.16.42.99
```

TTL은 DR 전환 데모 시 DNS 캐시 지연을 줄이기 위해 `60`초로 둔다.

DR 전환 시에는 `flaskapp`의 A 레코드를 제거하고 CNAME 레코드로 변경한다.

```dns
flaskapp   IN CNAME  k8s-flaskapp-xxxx.ap-northeast-2.elb.amazonaws.com.
```

동일한 이름에 A 레코드와 CNAME 레코드를 동시에 두지 않는다.

## Ingress Host 기준

DNS 레코드만 변경해서는 애플리케이션 라우팅이 완성되지 않는다.

클라이언트는 DNS 전환 이후에도 HTTP Host 헤더로 `flaskapp.team.snow.internal`을 보낸다. 따라서 On-premise NGINX Ingress와 AWS DR 환경의 ALB/EKS Ingress 모두 같은 Host 이름을 처리할 수 있어야 한다.

On-premise Helm values 또는 Ingress manifest의 host 값은 다음 기준으로 맞춘다.

```yaml
ingress:
  host: flaskapp.team.snow.internal
```

AWS DR 전환 후에도 같은 도메인을 유지하려면 AWS ALB/EKS Ingress rule에도 `flaskapp.team.snow.internal` Host rule이 포함되어야 한다.

## 방화벽 정책 기준

`dns` VM 추가에 따라 다음 트래픽을 허용한다.

| 출발지 | 목적지 | 포트 | 목적 |
|---|---|---:|---|
| 데모 PC / VLAN 10 | `dns 172.16.41.53` | UDP/TCP `53` | 내부 DNS 질의 |
| `dns 172.16.41.53` | 외부 DNS | UDP/TCP `53` | forwarder를 통한 외부 DNS 조회 |
| VLAN 40 Admin | `dns 172.16.41.53` | TCP `22` | 운영 관리 |
| 데모 PC / VLAN 10 | HAProxy VIP `172.16.42.99` | TCP `80`, `443` | On-premise FlaskApp 접근 |

`dns` VM은 내부 실습망의 DNS 전환 데모를 위한 노드이므로, 외부 인터넷에서 `dns` VM의 53번 포트로 직접 접근하는 구조는 만들지 않는다.

## 운영 및 데모 기준

정상 운영 데모 기준은 다음과 같다.

```text
1. 데모 PC의 DNS 서버가 dns VM 172.16.41.53을 바라본다.
2. Bind9는 flaskapp.team.snow.internal을 172.16.42.99로 응답한다.
3. HAProxy VIP 172.16.42.99는 worker node의 NGINX Ingress NodePort로 트래픽을 전달한다.
4. NGINX Ingress Controller는 Host 기준으로 FlaskApp Service에 요청을 전달한다.
5. 데모 PC에서 http://flaskapp.team.snow.internal/info 접속이 가능해야 한다.
```

DR 전환 데모 기준은 다음과 같다.

```text
1. On-premise 장애를 선언한다.
2. Bind9의 flaskapp.team.snow.internal 레코드를 AWS ALB DNS로 변경한다.
3. Bind9를 재시작하거나 zone reload를 수행한다.
4. 데모 PC에서 DNS 캐시를 갱신한다.
5. 같은 도메인으로 접속했을 때 AWS DR 환경의 FlaskApp으로 연결되어야 한다.
```

## 검증 명령

Bind9 VM에서 설정을 검증한다.

```bash
sudo named-checkconf
sudo named-checkzone team.snow.internal /etc/bind/zones/db.team.snow.internal
sudo systemctl reload bind9
sudo systemctl status bind9
```

데모 PC 또는 테스트 노드에서 DNS 응답을 확인한다.

```bash
dig @172.16.41.53 flaskapp.team.snow.internal
nslookup flaskapp.team.snow.internal 172.16.41.53
curl http://flaskapp.team.snow.internal/info
```

## 완료 기준

- `dns` VM이 `172.16.41.53/24`로 구성되어 있다.
- 데모 PC의 DNS 서버가 `172.16.41.53`으로 설정되어 있다.
- 정상 운영 시 `flaskapp.team.snow.internal`이 `172.16.42.99`로 응답한다.
- DR 전환 시 `flaskapp.team.snow.internal`이 AWS ALB DNS CNAME으로 응답한다.
- 같은 도메인으로 On-premise FlaskApp과 AWS DR FlaskApp 전환 흐름을 검증할 수 있다.

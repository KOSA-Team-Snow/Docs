# ADR-003: 내부 Bind9 기반 DNS 전환 데모 구조 선택

## 상태

승인됨

## 배경

프로젝트는 On-premise Kubernetes를 기본 운영 환경으로 사용하고, On-premise 장애 발생 시 AWS DR 환경으로 사용자 트래픽을 전환하는 구조를 목표로 한다.

ADR-001에서는 정상 운영 시 외부 사용자 진입점을 On-premise에 두고, 장애 시 AWS ALB로 전환하는 방향을 결정했다. 실제 운영 환경이라면 사용자는 Route 53에 등록된 서비스 도메인으로 접근하고, Route 53 레코드를 On-premise 진입점에서 AWS ALB로 변경하여 DR 전환을 수행한다.

하지만 현재 실습 환경에는 중요한 제약이 있다.

운영 환경 특성상 On-premise 구간에서 공인 IP를 직접 받을 수 없다. 강의실 네트워크는 상위 라우터에서 사설 IP를 발급받는 구조이고, 프로젝트 내부망도 별도 사설 IP 대역으로 구성되어 있다.

따라서 다음과 같은 실제 인터넷 공개 구조를 그대로 구현할 수 없다.

```text
사용자
  ↓
Route 53
  ↓
On-premise 공인 IP
  ↓
pfSense
  ↓
HAProxy VIP
  ↓
Kubernetes Ingress
  ↓
FlaskApp
```

현재 네트워크에서는 Route 53에 등록할 수 있는 On-premise 공인 IP가 없으므로, 공인 DNS를 통해 외부 인터넷에서 On-premise 서비스로 직접 접근하는 데모는 불가능하다.

이에 따라 실제 공인망 대신 내부망 안에 데모용 외부 사용자 구간을 만들고, 데모용 PC를 외부 사용자로 가정하여 DNS 기반 진입 및 DR 전환 흐름을 검증하는 방식이 필요하다.

## 검토한 대안

### 대안 1: 데모 PC의 `/etc/hosts`에 HAProxy VIP 직접 등록

이 대안은 데모 PC의 hosts 파일에 서비스 도메인과 HAProxy VIP를 직접 등록하는 방식이다.

```text
172.16.42.99 flaskapp.team.snow.internal
```

접속 흐름은 다음과 같다.

```text
데모 PC
  ↓
/etc/hosts
  ↓
HAProxy VIP 172.16.42.99
  ↓
HAProxy
  ↓
NGINX Ingress Controller
  ↓
FlaskApp
```

장점은 다음과 같다.

- 구성이 가장 단순하다.
- 별도 DNS 서버가 필요 없다.
- HAProxy VIP와 Kubernetes Ingress 연결을 빠르게 검증할 수 있다.

하지만 다음 이유로 최종 채택하지 않았다.

- 각 데모 PC마다 hosts 파일을 수정해야 한다.
- DNS 전환을 통한 DR 시나리오를 설명하기 어렵다.
- Route 53 레코드 전환을 모의하기에는 수동 hosts 파일 변경이 지나치게 단순하다.
- 여러 사용자가 같은 도메인 기준으로 테스트하는 환경을 만들기 어렵다.

### 대안 2: 내부 Bind9 DNS 서버를 사용

이 대안은 내부망에 Bind9 DNS VM을 설치하고, 서비스 도메인을 HAProxy VIP로 응답하게 하는 방식이다.

정상 운영 시 DNS 응답은 다음과 같다.

```text
flaskapp.team.snow.internal -> 172.16.42.99
```

접속 흐름은 다음과 같다.

```text
데모 PC
  ↓
Bind9 DNS
  ↓
HAProxy VIP 172.16.42.99
  ↓
HAProxy
  ↓
Worker NodePort
  ↓
NGINX Ingress Controller
  ↓
FlaskApp
```

DR 전환 시에는 같은 도메인의 DNS 응답을 AWS DR 진입점으로 변경한다.

```text
flaskapp.team.snow.internal -> AWS ALB DNS
```

AWS ALB는 고정 IP가 아니라 DNS 이름을 제공하므로, Bind9에서는 A 레코드 대신 CNAME 레코드로 AWS ALB DNS를 가리키는 방식으로 DR 전환을 모의할 수 있다.

```dns
flaskapp    IN    CNAME    k8s-flaskapp-xxxx.ap-northeast-2.elb.amazonaws.com.
```

장점은 다음과 같다.

- `/etc/hosts`보다 실제 DNS 운영 방식에 가깝다.
- Route 53 레코드 전환을 내부 실습망에서 모의할 수 있다.
- 데모 PC가 여러 대여도 DNS 서버만 바라보게 하면 동일한 도메인으로 접속할 수 있다.
- 정상 운영과 DR 운영 모두 같은 도메인을 유지할 수 있다.
- 발표 시 `Route 53 역할을 내부 Bind9로 대체했다`고 명확히 설명할 수 있다.

단점은 다음과 같다.

- Bind9 VM을 별도로 설치하고 운영해야 한다.
- 데모 PC의 DNS 서버를 Bind9로 지정해야 한다.
- AWS ALB DNS를 CNAME으로 사용할 경우 Bind9가 외부 DNS를 질의할 수 있도록 forwarder 설정이 필요하다.
- 실제 공인 인터넷 DNS 전환과 완전히 동일한 것은 아니므로, 데모용 모의 구조임을 명확히 설명해야 한다.

### 대안 3: 공인 IP 없이 Route 53에 사설 IP 등록

이 대안은 Route 53에 On-premise 사설 IP를 직접 등록하는 방식이다.

예를 들면 다음과 같다.

```text
flaskapp.example.com -> 172.16.42.99
```

하지만 이 방식은 외부 인터넷 사용자 관점에서 동작하지 않는다. `172.16.42.99`는 사설 IP이므로 인터넷에서 라우팅되지 않는다.

따라서 Route 53에 레코드를 만들 수는 있어도, 외부 사용자가 해당 IP로 접속할 수 없다.

이유는 다음과 같다.

- On-premise 공인 IP가 없다.
- 사설 IP는 인터넷에서 라우팅되지 않는다.
- 실제 외부 사용자는 `172.16.42.99`로 접근할 수 없다.
- 프로젝트 데모 목적에도 맞지 않는다.

## 결정 사항

내부망에 Bind9 DNS VM을 설치하고, 데모 PC는 Bind9를 DNS 서버로 사용한다.

On-premise 정상 운영 시 Bind9는 서비스 도메인을 HAProxy VIP로 응답한다.

```text
flaskapp.team.snow.internal -> 172.16.42.99
```

On-premise 장애가 발생하고 DR 전환을 선언한 경우, Bind9의 동일 도메인 레코드를 AWS DR 진입점으로 변경한다.

AWS ALB는 고정 IP를 제공하지 않으므로, DR 전환 시에는 AWS ALB DNS를 CNAME으로 연결한다.

```text
flaskapp.team.snow.internal -> AWS ALB DNS
```

최종 데모 구조는 다음과 같다.

```text
데모 PC
  ↓
Bind9 DNS
  ↓
flaskapp.team.snow.internal
  ↓
HAProxy VIP 172.16.42.99
  ↓
HAProxy
  ↓
Worker NodePort 30080/30443
  ↓
NGINX Ingress Controller
  ↓
FlaskApp Service
  ↓
FlaskApp Pod
```

DR 전환 후 구조는 다음과 같다.

```text
데모 PC
  ↓
Bind9 DNS
  ↓
flaskapp.team.snow.internal
  ↓
AWS ALB DNS
  ↓
EKS Ingress / Service
  ↓
FlaskApp Pod on EKS
  ↓
AWS RDS / S3
```

구성요소별 책임은 다음과 같다.

| 구성요소 | 책임 |
|---|---|
| 데모 PC | 외부 사용자 역할을 수행 |
| Bind9 DNS VM | Route 53 역할을 내부망에서 모의 |
| HAProxy VIP | On-premise 정상 운영 진입점 |
| NGINX Ingress Controller | Kubernetes 내부 Host/path 기반 라우팅 |
| AWS ALB | DR 전환 시 AWS 진입점 |
| Route 53 | 실제 운영 환경에서 DNS 전환을 담당할 서비스 |

## 결정 근거

### 1. On-premise 공인 IP를 받을 수 없는 환경 제약 반영

현재 강의실 네트워크와 프로젝트 내부망은 사설 IP 기반으로 구성되어 있다.

상위 라우터에서 공인 IP를 프로젝트 pfSense 또는 HAProxy 계층으로 직접 내려줄 수 없기 때문에, 실제 Route 53과 공인 IP 기반 외부 공개 구조를 그대로 구현할 수 없다.

따라서 이번 데모에서는 공인 인터넷 사용자 대신 내부망에 연결된 데모 PC를 외부 사용자로 가정한다.

### 2. Route 53 DNS 전환을 내부망에서 모의 가능

DR 시나리오의 핵심은 사용자가 같은 도메인으로 접속하되, 장애 시 DNS가 On-premise 진입점 대신 AWS DR 진입점을 응답하도록 바꾸는 것이다.

Bind9를 사용하면 Route 53의 역할을 내부망에서 모의할 수 있다.

정상 운영:

```text
flaskapp.team.snow.internal -> HAProxy VIP 172.16.42.99
```

DR 전환:

```text
flaskapp.team.snow.internal -> AWS ALB DNS
```

이 방식은 실제 운영의 Route 53 수동 전환 또는 failover 정책을 데모 환경에 맞게 축소한 구조이다.

### 3. `/etc/hosts`보다 운영 모델에 가깝다

`/etc/hosts` 방식은 단일 PC 테스트에는 충분하지만, DNS 기반 DR 전환을 설명하기에는 부족하다.

Bind9를 내부 DNS로 사용하면 데모 PC가 DNS 서버에 질의하고, DNS 응답에 따라 On-premise 또는 AWS DR 환경으로 이동하는 흐름을 보여줄 수 있다.

따라서 발표와 문서에서 다음 메시지를 명확히 전달할 수 있다.

```text
실제 운영에서는 Route 53이 담당하는 DNS 전환을,
실습 환경에서는 내부 Bind9가 대신 수행한다.
```

### 4. AWS ALB의 DNS 기반 진입 구조와 호환

AWS ALB는 고정 IP를 직접 제공하지 않고 DNS 이름을 제공한다.

실제 운영에서는 Route 53 Alias 레코드로 ALB를 가리킨다. 내부 Bind9 데모에서는 Alias 기능 대신 CNAME 레코드로 ALB DNS를 가리키는 방식으로 전환을 모의한다.

따라서 AWS DR 전환을 설명할 때도 다음 구조를 유지할 수 있다.

```text
서비스 도메인
  ↓
DNS 전환
  ↓
AWS ALB
  ↓
EKS FlaskApp
```

## 영향

### 긍정적 영향

- 공인 IP 없이도 DNS 기반 외부 진입 데모를 수행할 수 있다.
- Route 53 기반 DR 전환 개념을 내부망에서 설명할 수 있다.
- 데모 PC는 실제 사용자처럼 도메인으로 서비스에 접근할 수 있다.
- On-premise 정상 운영과 AWS DR 전환을 같은 도메인 기준으로 비교할 수 있다.
- HAProxy, NGINX Ingress Controller, FlaskApp까지 이어지는 현재 연결 구조를 유지할 수 있다.

### 부정적 영향

- 실제 인터넷 사용자가 접근하는 공인 서비스는 아니다.
- Bind9 VM 운영과 데모 PC DNS 설정이 추가된다.
- DNS 캐시와 TTL 때문에 전환 직후 일부 지연이 발생할 수 있다.
- 실제 Route 53 failover, health check, Alias record와 완전히 동일하지는 않다.
- AWS ALB 전환 시 Bind9의 외부 DNS forwarder 설정이 필요할 수 있다.

## 상세 설계 문서

Bind9 VM의 이름, IP, VLAN, zone 파일, forwarder, 방화벽 정책, 검증 명령은 별도 설계 문서인 `blueprint/bind9-design.md`에서 관리한다.

ADR-003은 내부 Bind9 DNS를 선택한 결정 배경과 근거를 기록하고, 구체적인 구현 설계는 `bind9-design.md`에 분리한다.

## 운영 및 데모 기준

정상 운영 데모 기준은 다음과 같다.

```text
1. 데모 PC의 DNS 서버가 Bind9 VM을 바라본다.
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

## 결정 사항 요약

운영 환경 특성상 On-premise 공인 IP를 받을 수 없으므로, Route 53과 공인 IP 기반 외부 공개 구조를 실제로 구현하지 않는다.

대신 내부망에 Bind9 DNS VM을 설치하고, 데모 PC를 외부 사용자로 가정한다.

Bind9는 정상 운영 시 서비스 도메인을 HAProxy VIP `172.16.42.99`로 응답한다. 장애 발생 후 DR 전환 시 동일 도메인을 AWS ALB DNS로 변경하여 AWS DR 환경으로 트래픽이 이동하는 과정을 모의한다.

이 결정은 공인 IP 제약이 있는 실습 환경에서도 DNS 기반 DR 전환 흐름을 일관되게 설명하고 검증하기 위한 것이다.

Bind9 VM의 구체적인 이름, IP, VLAN, zone 구성, 방화벽 정책, 검증 명령은 `blueprint/bind9-design.md`에서 관리한다.

# Architecture 문서 인덱스

이 디렉터리는 온프렘 + AWS DR 프로젝트의 아키텍처 결정, 네트워크 설계, DNS/진입점 설계, 과거 후보안을 관리한다.

## 먼저 읽을 문서

1. [current-architecture-summary.md](./current-architecture-summary.md)  
   현재 구현/실측 기준으로 정리한 최신 아키텍처 요약이다. 발표나 최종 문서 작성 시 이 문서를 기준으로 삼는다.

2. [시스템-아키텍쳐-설계-최종본.md](./시스템-아키텍쳐-설계-최종본.md)  
   온프렘 + AWS DR 전체 설계의 본문이다. 일부 IP 배치와 MetalLB 관련 설명은 과거 계획이 남아 있으므로 최신 요약과 대조한다.

3. [bind9-design.md](./bind9-design.md)  
   내부 Bind9 DNS로 Route 53 전환을 모의하는 설계 문서다.

4. [ADR](./ADR)  
   주요 아키텍처 결정을 기록한다.

## ADR 목록

| ADR | 결정 | 현재 상태 |
| --- | --- | --- |
| [ADR-001](./ADR/ADR-001-초기-아키텍쳐-설계-결정.md) | 정상 운영 진입점은 On-prem, 장애 시 AWS DR로 전환 | 채택 |
| [ADR-002](./ADR/ADR-002-일반-사용자-진입점-관련-결정.md) | MetalLB 대신 HAProxy + Keepalived + ingress-nginx NodePort | 채택 |
| [ADR-003](./ADR/ADR-003-bind9-사용-관련-결정.md) | 공인 Route 53 대신 내부 Bind9로 DNS 전환 데모 | 채택 |

## 문서 상태

| 문서 | 상태 | 설명 |
| --- | --- | --- |
| [current-architecture-summary.md](./current-architecture-summary.md) | 최신 기준 | 실측 결과를 반영한 현재 아키텍처 요약 |
| [시스템-아키텍쳐-설계-최종본.md](./시스템-아키텍쳐-설계-최종본.md) | 주요 설계 본문 | 전체 설계 설명용. 일부 과거 계획은 최신 요약과 대조 필요 |
| [bind9-design.md](./bind9-design.md) | 최신 기준에 가까움 | `flaskapp.team.snow.internal -> 172.16.42.99` 구조와 일치 |
| [하드웨어-구조-세팅-및-네트워크-설계.md](./하드웨어-구조-세팅-및-네트워크-설계.md) | 과거 계획 포함 | 초기 IP/VM 배치안. 실제 inventory와 일부 다름 |
| [시스템-아키텍처-설계안-후보-1.md](./시스템-아키텍처-설계안-후보-1.md) | 참고용 과거 후보 | AWS Front Door/EC2 HAProxy 중심 후보안 |
| [시스템-아키텍쳐-설계안-후보-2.md](./시스템-아키텍쳐-설계안-후보-2.md) | 참고용 과거 후보 | AWS Front Door 기반 Hybrid Pilot Light 후보안 |

## 최신 기준 핵심

```text
정상 운영:
사용자
  -> Bind9 DNS
  -> HAProxy/Keepalived VIP 172.16.42.99
  -> ingress-nginx NodePort 30080/30443
  -> NGINX Ingress
  -> FlaskApp Service
  -> FlaskApp Pod
  -> MariaDB VM 172.16.43.160

DR 기반:
On-prem MariaDB
  -> pfSense/VPN
  -> AWS DMS
  -> AWS RDS

장애 전환:
Terraform dr_active=true
  -> EKS/ALB 앱 실행 계층 활성화
  -> DNS를 AWS ALB로 전환
```

## 오래된 내용 주의

- `flaskapp.onprem.local`은 과거 테스트 host다. 최신 기준은 `flaskapp.team.snow.internal`이다.
- `grafana.onprem.local`은 과거 테스트 host다. 최신 기준은 `grafana.team.snow.internal`이다.
- MetalLB는 최종 사용자 진입점으로 채택하지 않았다.
- FlaskApp Service는 NodePort가 아니라 ClusterIP다.
- NodePort는 `ingress-nginx-controller`가 사용한다.
- AWS Front Door/EC2 HAProxy 상시 진입 구조는 최종 채택안이 아니다.
- `monitoring` VM은 이름과 달리 현재 실측 기준 별도 Grafana/Prometheus/Loki 서버가 아니라, 실제 관측 stack은 Kubernetes 내부 `monitoring`/`logging` namespace에서 동작한다.

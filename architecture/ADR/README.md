# ADR 문서 인덱스

이 디렉터리는 Architecture Decision Record를 보관한다.

| ADR | 결정 | 현재 상태 |
| --- | --- | --- |
| [ADR-001-초기-아키텍쳐-설계-결정.md](./ADR-001-초기-아키텍쳐-설계-결정.md) | 정상 운영은 On-prem, 장애 시 AWS DR 전환 | 채택 |
| [ADR-002-일반-사용자-진입점-관련-결정.md](./ADR-002-일반-사용자-진입점-관련-결정.md) | MetalLB 대신 HAProxy + Keepalived + ingress-nginx NodePort | 채택 |
| [ADR-003-bind9-사용-관련-결정.md](./ADR-003-bind9-사용-관련-결정.md) | 공인 Route 53 대신 내부 Bind9로 DNS 전환 데모 | 채택 |

최신 실측 기준은 [../current-architecture-summary.md](../current-architecture-summary.md)를 우선한다.

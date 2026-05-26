# AWS Old 문서 인덱스

이 디렉터리는 과거 AWS DR 설계/구현안을 보관한다.

최신 기준은 상위 디렉터리의 [../README.md](../README.md), [../aws-architecture-final.md](../aws-architecture-final.md), 그리고 [../../current/full-verification-assessment-2026-05-26.md](../../current/full-verification-assessment-2026-05-26.md)를 우선한다.

## 주의

- 일부 문서는 Route 53 public health check, `172.16.41.110` MetalLB 진입, 자동 failover 표현을 포함한다.
- 현재 실습/발표 기준은 내부 Bind9로 DNS 전환을 모의하고, On-prem 진입은 HAProxy/Keepalived VIP `172.16.42.99`를 사용한다.
- EKS/ALB는 Pilot Light 설계상 평시 미생성 상태다.

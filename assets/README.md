# Assets 문서 인덱스

이 디렉터리는 발표/문서에 사용할 Mermaid 구성도를 보관한다.

## 문서 상태

| 문서 | 상태 | 설명 |
| --- | --- | --- |
| [전체-아키텍쳐-설계도.md](./전체-아키텍쳐-설계도.md) | 과거 설계 포함 | Route 53 중심 표현이 남아 있어 최신 Bind9 기준과 대조 필요 |
| [네트워크-흐름도.md](./네트워크-흐름도.md) | 구버전 | MetalLB/`172.16.41.110` 직접 진입 표현이 있어 최신 발표에는 그대로 사용하지 않음 |
| [물리-네트워크-구조.md](./물리-네트워크-구조.md) | 참고 가능 | 물리/대역 구조 참고용. 실제 VM 목록은 최신 current 문서를 우선 |

## 최신 다이어그램 기준

최신 발표용 흐름은 [../presentation/onprem-system-architecture-mermaid.md](../presentation/onprem-system-architecture-mermaid.md)를 우선 사용한다.

최신 핵심 경로:

```text
사용자
  -> Bind9 DNS 172.16.41.53
  -> HAProxy/Keepalived VIP 172.16.42.99
  -> ingress-nginx NodePort 30080/30443
  -> NGINX Ingress
  -> FlaskApp Service
  -> FlaskApp Pod
  -> MariaDB VM 172.16.43.160
```

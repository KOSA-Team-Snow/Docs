# Monitoring Operation 문서 인덱스

이 디렉터리는 Kubernetes 내부 모니터링/AIOps 운영 문서를 관리한다.

| 문서 | 설명 |
| --- | --- |
| [monitoring-k8s-deployment-decision.md](./monitoring-k8s-deployment-decision.md) | Monitoring stack을 Kubernetes 내부에 배포한 결정 |
| [aiops-holmesgpt-alert-analysis-guide.md](./aiops-holmesgpt-alert-analysis-guide.md) | HolmesGPT 기반 alert 분석 가이드 |

## 최신 기준

- Monitoring stack은 별도 monitoring VM이 아니라 Kubernetes 내부 `monitoring` namespace 중심이다.
- Logging stack은 `logging` namespace의 Loki/Alloy 중심이다.
- AIOps는 `aiops` namespace의 HolmesGPT를 사용한다.
- Grafana host는 `grafana.team.snow.internal`이며 Bind9 기준 `172.16.42.99`로 해석된다.

# Operation 문서 인덱스

이 디렉터리는 운영/점검/배포/모니터링 Runbook을 관리한다.

## 하위 디렉터리

| 디렉터리 | 역할 |
| --- | --- |
| [k8s](./k8s) | Kubernetes, FlaskApp, ArgoCD, Ingress, ECR, Secret, S3 운영 |
| [monitoring](./monitoring) | Prometheus/Grafana/Loki/HolmesGPT 운영 |
| [proxmox](./proxmox) | Proxmox 운영 문서 자리. 현재 별도 문서 없음 |

## 최신 기준

- FlaskApp은 ArgoCD/Helm으로 `flaskapp-prod` namespace에 배포한다.
- 최신 image tag 확인 기준은 GitHub Actions/ECR/ArgoCD 상태다.
- On-prem 접속 host는 `flaskapp.team.snow.internal`, `grafana.team.snow.internal`이다.
- DB 접속은 현재 IP `172.16.43.160` 기준이다.

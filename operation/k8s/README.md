# Kubernetes Operation 문서 인덱스

Kubernetes와 FlaskApp 운영 Runbook 모음이다.

## FlaskApp / GitOps / CI-CD

| 문서 | 설명 |
| --- | --- |
| [github-actions-cicd-guide.md](./github-actions-cicd-guide.md) | GitHub Actions 기반 build/push/update 흐름 |
| [ecr-build-push-guide.md](./ecr-build-push-guide.md) | ECR build/push |
| [argocd-flaskapp-ingress-runbook.md](./argocd-flaskapp-ingress-runbook.md) | ArgoCD/Ingress 기준 FlaskApp 운영 |
| [argocd-ui-sync-check-guide.md](./argocd-ui-sync-check-guide.md) | ArgoCD UI sync 점검 |
| [flaskapp-probe-rollout-runbook.md](./flaskapp-probe-rollout-runbook.md) | Probe/Rollout 점검 |

## 연결/Secret/정책

| 문서 | 설명 |
| --- | --- |
| [db-secret-guide.md](./db-secret-guide.md) | DB Secret |
| [flaskapp-db-connect-runbook.md](./flaskapp-db-connect-runbook.md) | FlaskApp DB 연결 점검 |
| [flaskapp-s3-connect-runbook.md](./flaskapp-s3-connect-runbook.md) | FlaskApp S3 연결 점검 |
| [flaskapp-ecr-pull-secret-refresh-runbook.md](./flaskapp-ecr-pull-secret-refresh-runbook.md) | ECR Pull Secret 갱신 |
| [flaskapp-kubernetes-policy-summary.md](./flaskapp-kubernetes-policy-summary.md) | HPA/PDB/NetworkPolicy 등 운영 정책 |

## Kubernetes 구축/기초 문서

| 문서 | 설명 |
| --- | --- |
| [k8s-구성요소.md](./k8s-구성요소.md) | Kubernetes 구성요소 |
| [k8s-노드 계획.md](./k8s-노드%20계획.md) | 노드 계획 |
| [k8s-라벨링.md](./k8s-라벨링.md) | 노드 라벨링 |
| [k8s-설치검증.md](./k8s-설치검증.md) | 설치 검증 |
| [k8s-설치방법](./k8s-설치방법) | 설치 방법 기록 |
| [helm-ingress-guide.md](./helm-ingress-guide.md) | Helm/Ingress |
| [ingress-nginx-argocd-guide.md](./ingress-nginx-argocd-guide.md) | ingress-nginx/ArgoCD |

## 최신 기준

- Cluster: control plane 3대, worker 5대
- API VIP: `172.16.43.99:6443`
- CNI: Calico
- Ingress: ingress-nginx NodePort `30080/30443`
- FlaskApp Service: ClusterIP
- FlaskApp host: `flaskapp.team.snow.internal`

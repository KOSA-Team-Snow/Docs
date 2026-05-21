# FlaskApp Kubernetes 운영 정책 정리

## 목적

`flaskapp-prod` namespace에서 FlaskApp을 안정적으로 운영하기 위해 적용한 Kubernetes 정책을 정리한다.

ArgoCD는 FlaskApp 배포 기준으로 다음 Helm chart를 사용한다.

```text
helm/flaskapp
```

따라서 아래 정책들은 GitHub push 후 ArgoCD sync를 통해 클러스터에 반영된다.

## 완료된 정책 요약

| 정책 | 목적 | 주요 파일 |
| --- | --- | --- |
| ServiceAccount | 앱 Pod 실행 계정 분리 | `templates/serviceaccount.yaml`, `templates/deployment.yaml`, `values.yaml` |
| Probe | 앱 기동/준비/생존 상태 확인 | `templates/deployment.yaml`, `values.yaml` |
| PDB | 자발적 중단 시 최소 가용 Pod 유지 | `templates/pdb.yaml`, `values.yaml` |
| HPA | CPU 사용률 기준 자동 확장 | `templates/hpa.yaml`, `values.yaml` |
| Pod 분산 배치 | replica가 한 node에 몰리지 않도록 유도 | `templates/deployment.yaml`, `values.yaml` |
| RBAC | 조회자/배포자/Secret 관리자 권한 분리 | `templates/rbac.yaml`, `values.yaml` |
| NetworkPolicy | FlaskApp Pod 통신 범위 제한 | `templates/networkpolicy.yaml`, `values.yaml` |

## 1. ServiceAccount

FlaskApp Pod가 기본 `default` ServiceAccount 대신 앱 전용 `flaskapp-sa`로 실행되도록 설정했다.

```yaml
serviceAccount:
  create: true
  name: flaskapp-sa
  automountServiceAccountToken: false
```

의도:

- 앱 실행 계정을 명확히 분리
- Kubernetes API token 자동 마운트 차단
- 앱 Pod 권한 최소화

확인:

```bash
kubectl get sa -n flaskapp-prod flaskapp-sa
kubectl get deploy -n flaskapp-prod flaskapp -o yaml | grep -A4 serviceAccount
```

## 2. Probe

FlaskApp의 기동, 준비, 생존 상태를 확인하기 위해 `startupProbe`, `readinessProbe`, `livenessProbe`를 사용한다.

```yaml
probes:
  startup:
    path: /info
  readiness:
    path: /info
  liveness:
    path: /info
```

참고:

- ServiceAccount 적용 후 새 rollout 과정에서 기존 liveness 설정이 너무 빨리 실패해 `CrashLoopBackOff`가 발생했다.
- 원인은 ServiceAccount가 아니라 앱 초기 기동 시간보다 빠른 liveness check였다.
- `startupProbe` 추가와 probe 타이밍 완화 후 정상화했다.

확인:

```bash
kubectl get deploy -n flaskapp-prod flaskapp -o yaml | grep -A35 startupProbe
kubectl get pods -n flaskapp-prod
```

## 3. PDB

노드 drain, 점검, 재부팅 같은 자발적 중단 상황에서 FlaskApp Pod가 모두 동시에 내려가지 않게 설정했다.

```yaml
podDisruptionBudget:
  enabled: true
  minAvailable: 1
```

확인:

```bash
kubectl describe pdb -n flaskapp-prod flaskapp-pdb
```

정상 예시:

```text
Min available:  1
Allowed disruptions:  1
Current:              2
Desired:              1
Total:                2
```

## 4. HPA

CPU 사용률 기준으로 FlaskApp replica 수를 자동 조정하도록 설정했다.

```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 4
  targetCPUUtilizationPercentage: 70
```

주의:

- HPA 동작에는 metrics-server가 필요하다.
- CPU 사용률 계산에는 `resources.requests.cpu`가 필요하다.

확인:

```bash
kubectl get hpa -n flaskapp-prod
kubectl describe hpa -n flaskapp-prod flaskapp-hpa
kubectl top pods -n flaskapp-prod
```

## 5. Pod 분산 배치

FlaskApp Pod가 가능한 한 서로 다른 node에 배치되도록 `topologySpreadConstraints`를 설정했다.

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway
```

의도:

- 같은 node에 replica가 몰리는 상황 완화
- 노드 장애 또는 점검 시 서비스 영향 감소
- 분산 조건을 만족하지 못해도 Pod가 Pending으로 막히지 않도록 설정

확인:

```bash
kubectl get pods -n flaskapp-prod -o wide
```

## 6. RBAC

`flaskapp-prod` namespace의 운영 권한을 조회자, 배포자, Secret 관리자로 분리했다.

```yaml
rbac:
  enabled: true
  bindings:
    - role: viewer
      group: team3-flaskapp-viewers
    - role: deployer
      group: team3-flaskapp-deployers
    - role: secret-admin
      group: team3-flaskapp-secret-admins
```

역할:

| Role | 권한 |
| --- | --- |
| `flaskapp-viewer` | Pod, log, Service, Ingress, HPA, PDB 조회 |
| `flaskapp-deployer` | Deployment, Service, Ingress, ConfigMap, HPA, PDB 수정 |
| `flaskapp-secret-admin` | Secret 조회/생성/수정 |

중요:

```text
flaskapp-deployer에는 Secret 권한을 부여하지 않는다.
```

확인:

```bash
kubectl auth can-i get pods -n flaskapp-prod --as=test-viewer --as-group=team3-flaskapp-viewers
kubectl auth can-i patch deployment -n flaskapp-prod --as=test-deployer --as-group=team3-flaskapp-deployers
kubectl auth can-i get secret -n flaskapp-prod --as=test-deployer --as-group=team3-flaskapp-deployers
kubectl auth can-i update secret -n flaskapp-prod --as=test-secret-admin --as-group=team3-flaskapp-secret-admins
```

기대 결과:

```text
yes
yes
no
yes
```

## 7. NetworkPolicy

FlaskApp Pod에 대해 필요한 ingress/egress 통신만 허용하도록 NetworkPolicy를 설정했다.

```yaml
networkPolicy:
  enabled: true
  ingressController:
    namespace: ingress-nginx
  dns:
    namespace: kube-system
  database:
    cidr: 172.16.43.160/32
    port: 3306
  httpsEgress:
    cidr: 0.0.0.0/0
```

허용 통신:

```text
Ingress:
- ingress-nginx controller -> FlaskApp TCP 80

Egress:
- FlaskApp -> CoreDNS TCP/UDP 53
- FlaskApp -> MariaDB 172.16.43.160 TCP 3306
- FlaskApp -> 외부 HTTPS TCP 443
```

사전 확인:

- Calico Running 확인
- NetworkPolicy API 지원 확인
- Ingress Controller label 확인
- CoreDNS label 확인
- MariaDB `172.16.43.160:3306` 접근 확인
- S3 HTTPS endpoint 접근 확인

확인:

```bash
kubectl get networkpolicy -n flaskapp-prod
kubectl describe networkpolicy -n flaskapp-prod flaskapp-networkpolicy
curl -H "Host: flaskapp.team.snow.internal" http://172.16.43.110:30080/info
```

주의:

- Ingress Controller Service는 NodePort `30080`으로 노출된다.
- NetworkPolicy 적용 후 일반 테스트 Pod에서 `flaskapp-service:80`으로 직접 접근하는 것은 차단될 수 있다.

## 공통 확인 명령

```bash
kubectl get deploy,po,svc,hpa,pdb -n flaskapp-prod -o wide
kubectl get sa,role,rolebinding,networkpolicy -n flaskapp-prod
kubectl get events -n flaskapp-prod --sort-by=.lastTimestamp
```

## 남은 후보 작업

| 작업 | 비고 |
| --- | --- |
| Resource limit 조정 | 현재 memory limit `256Mi`, 사용량 보고 상향 검토 |
| nodeSelector / taints / tolerations | app/infra node 역할 분리가 확정되면 적용 |
| ArgoCD RBAC | ArgoCD UI/Sync 권한 분리가 필요할 때 적용 |
| Secret 관리 고도화 | ExternalSecret/SealedSecret 도입 시 별도 작업 |

## GitHub Issue 본문 초안

제목:

```text
[Task] FlaskApp Kubernetes 운영 정책 정리 및 적용
```

본문:

```md
## 작업 내용

FlaskApp Helm chart에 Kubernetes 운영 정책을 추가하고 적용 상태를 정리한다.

적용 정책:

- ServiceAccount
- startup/readiness/liveness Probe
- PodDisruptionBudget
- HorizontalPodAutoscaler
- topologySpreadConstraints
- RBAC Role/RoleBinding
- NetworkPolicy

## 변경 대상

```text
helm/flaskapp/templates/serviceaccount.yaml
helm/flaskapp/templates/deployment.yaml
helm/flaskapp/templates/pdb.yaml
helm/flaskapp/templates/hpa.yaml
helm/flaskapp/templates/rbac.yaml
helm/flaskapp/templates/networkpolicy.yaml
helm/flaskapp/values.yaml
Docs/operation/k8s/flaskapp-kubernetes-policy-summary.md
```

## 완료 기준

- [ ] FlaskApp Pod가 `flaskapp-sa` ServiceAccount로 실행된다.
- [ ] `automountServiceAccountToken: false`가 적용되어 있다.
- [ ] startup/readiness/liveness probe가 `/info` 기준으로 동작한다.
- [ ] PDB가 `minAvailable: 1`로 생성되어 있다.
- [ ] HPA가 `minReplicas: 2`, `maxReplicas: 4`, CPU target 70%로 생성되어 있다.
- [ ] FlaskApp Pod가 가능한 서로 다른 node에 분산된다.
- [ ] RBAC Role/RoleBinding이 viewer/deployer/secret-admin 기준으로 생성되어 있다.
- [ ] deployer는 Secret 조회 권한이 없다.
- [ ] NetworkPolicy가 생성되어 필요한 ingress/egress만 허용한다.
- [ ] Ingress NodePort 경로로 `/info` 접근이 정상이다.

## 검증 방법

```bash
kubectl get deploy,po,svc,hpa,pdb -n flaskapp-prod -o wide
kubectl get sa,role,rolebinding,networkpolicy -n flaskapp-prod
kubectl describe pdb -n flaskapp-prod flaskapp-pdb
kubectl describe hpa -n flaskapp-prod flaskapp-hpa
kubectl describe networkpolicy -n flaskapp-prod flaskapp-networkpolicy
curl -H "Host: flaskapp.team.snow.internal" http://172.16.43.110:30080/info
```

## 참고

NetworkPolicy 적용 후 일반 테스트 Pod에서 `flaskapp-service:80`으로 직접 접근하는 것은 차단될 수 있다.
FlaskApp 접근 검증은 Ingress NodePort 경로로 수행한다.
```

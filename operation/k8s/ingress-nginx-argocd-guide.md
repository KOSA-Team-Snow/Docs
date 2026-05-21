# Nginx Ingress Controller ArgoCD GitOps 전환 가이드

> 담당: 팀원 C (@ireneminhee)  
> 관련 이슈: infra #55, #56, #58  
> 상태: ArgoCD 관리 전환 완료 / NodePort 30080 고정

---

## 1. 배경

초기 설치(`argocd-flaskapp-ingress-runbook.md` 참고)에서는 Nginx Ingress Controller를 아래 방식으로 수동 설치했다.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.3/deploy/static/provider/baremetal/deploy.yaml
```

이 방식은 동작에는 문제가 없지만 GitOps 원칙과 맞지 않는다.  
다른 ArgoCD Application(flaskapp, monitoring)과 달리 ingress-nginx만 클러스터에 직접 설치된 상태로 남아 있었다.

infra #55에서 `argocd/apps/ingress.yaml`을 추가해 ArgoCD가 Helm chart로 ingress-nginx를 관리하도록 전환했다.

---

## 2. ArgoCD Application 파일

`infra/argocd/apps/ingress.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ingress-nginx
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://kubernetes.github.io/ingress-nginx
    chart: ingress-nginx
    targetRevision: 4.11.3
    helm:
      values: |
        controller:
          replicaCount: 1
          ingressClass: nginx
          ingressClassResource:
            enabled: true
            name: nginx
            controllerValue: k8s.io/ingress-nginx
          service:
            type: NodePort
            nodePorts:
              http: 30080
              https: 30443
          admissionWebhooks:
            enabled: false
          metrics:
            enabled: false
  destination:
    server: https://kubernetes.default.svc
    namespace: ingress-nginx
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

---

## 3. 핵심 설정 이유

### NodePort 고정 (30080 / 30443)

기존 수동 설치에서는 NodePort가 랜덤 할당되었다 (예: `32109`).  
랜덤 포트는 HAProxy backend 설정을 변경할 때마다 Ansible playbook을 수정해야 하는 문제가 있다.

Helm values에 NodePort를 명시적으로 고정해서 HAProxy 설정과 일관성을 유지한다.

```
HAProxy VIP:80 → Worker Node:30080 → Nginx Ingress Controller → FlaskApp Pod
```

### admissionWebhooks 비활성화

ArgoCD는 sync 시 Helm hook 리소스를 prune할 수 있다.  
Nginx Ingress Controller의 admission webhook Job이 완료되기 전에 ArgoCD가 관련 리소스를 삭제하면 아래 오류가 발생한다.

```text
Error: UPGRADE FAILED: pre-upgrade hooks failed: ... admission webhook ... timed out
```

이 문제를 피하기 위해 `admissionWebhooks.enabled: false`를 설정했다.  
On-prem 데모 환경에서 admission webhook 없이도 정상 동작한다.

### ServerSideApply

ArgoCD가 관리하는 리소스 중 어노테이션이 큰 CRD가 포함될 경우 client-side apply 방식은 실패할 수 있다.  
`ServerSideApply=true` syncOption을 추가해 이를 방지한다.

---

## 4. root-app과의 연동

`infra/argocd/root-app.yaml`이 `argocd/apps/` 경로를 감시하고 있어서  
`ingress.yaml` 파일이 Git에 머지되면 ArgoCD가 자동으로 ingress-nginx Application을 생성하고 sync한다.

```
root-app (argocd/apps/ 감시)
  ├── flaskapp       → helm/flaskapp
  ├── monitoring     → helm/monitoring
  └── ingress-nginx  → ingress-nginx Helm chart (kubernetes.github.io)
```

별도로 `kubectl apply`를 실행할 필요가 없다.

---

## 5. 검증

ArgoCD sync 후 아래 명령어로 확인한다.

```bash
# ingress-nginx Pod 상태
kubectl get pods -n ingress-nginx

# NodePort 확인 (30080, 30443이어야 함)
kubectl get svc ingress-nginx-controller -n ingress-nginx

# IngressClass 확인
kubectl get ingressclass

# FlaskApp Ingress 연결 확인
kubectl get ingress -n flaskapp-prod
```

정상 상태 예시:

```text
NAME                       READY   STATUS    RESTARTS
ingress-nginx-controller   1/1     Running   0

NAME                       TYPE       CLUSTER-IP   PORT(S)
ingress-nginx-controller   NodePort   10.x.x.x     80:30080/TCP,443:30443/TCP

NAME    CONTROLLER
nginx   k8s.io/ingress-nginx

NAME               CLASS   HOSTS                          ADDRESS         PORTS
flaskapp-ingress   nginx   flaskapp.team.snow.internal    172.16.43.114   80
```

NodePort 경유 직접 접속 테스트:

```bash
curl http://172.16.43.114:30080/info -H "Host: flaskapp.team.snow.internal"
```

HAProxy VIP 경유 접속 테스트 (HAProxy 연동 완료 후):

```bash
curl http://172.16.42.99/info
```

---

## 6. 트러블슈팅

### ArgoCD sync 후 ingress-nginx Pod가 뜨지 않는 경우

```bash
kubectl describe pod -n ingress-nginx
kubectl get events -n ingress-nginx --sort-by='.lastTimestamp'
```

### admissionWebhooks 관련 오류가 발생하는 경우

```bash
kubectl get validatingwebhookconfiguration | grep ingress
kubectl delete validatingwebhookconfiguration ingress-nginx-admission
```

기존 수동 설치 당시 생성된 admission webhook 설정이 남아있을 수 있다.  
위 명령으로 삭제 후 ArgoCD sync를 재시도한다.

### 기존 수동 설치 리소스와 충돌하는 경우

수동 설치(`kubectl apply -f baremetal manifest`)로 생성된 리소스와 ArgoCD Helm 리소스가 겹치면 충돌이 발생한다.

ArgoCD Application이 `Degraded` 상태라면 아래 순서로 정리한다.

```bash
# 기존 수동 설치 리소스 확인
helm list -n ingress-nginx
kubectl get all -n ingress-nginx

# 수동 설치 리소스 제거 (ArgoCD sync가 재생성)
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.3/deploy/static/provider/baremetal/deploy.yaml

# ArgoCD에서 ingress-nginx Application sync
kubectl get application ingress-nginx -n argocd
```

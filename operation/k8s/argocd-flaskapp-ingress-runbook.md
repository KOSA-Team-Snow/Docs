> 문서 상태: 2026-05-18 당시 작업 기록이다. 최신 기준은 `flaskapp.team.snow.internal` + HAProxy/Keepalived VIP `172.16.42.99` + ingress-nginx NodePort `30080/30443`이다.
>
> 이 문서의 `flaskapp.onprem.local` 및 worker NodePort 직접 접속 예시는 과거 검증 기록으로 본다.

# ArgoCD FlaskApp 배포 및 Ingress 연결 기록

> 담당: 팀원 C (@ireneminhee)  
> 기준일: 2026-05-18  
> 실행 위치: `bastion` 및 로컬 Mac  
> 목적: ECR 이미지 push 이후 ArgoCD로 FlaskApp을 배포하고, Nginx Ingress Controller를 통해 `/info` 응답을 확인한다.

---

## 1. 현재 완료 상태

아래 작업까지 완료했다.

- FlaskApp Docker image를 ECR에 push
- ArgoCD 설치
- `root-app.yaml` 적용
- `flaskapp` ArgoCD Application 생성
- ECR private image pull을 위한 `imagePullSecret` 생성
- FlaskApp Pod `Running` 확인
- Nginx Ingress Controller 설치
- `flaskapp-ingress`가 `nginx` IngressClass로 연결됨 확인
- NodePort 경유 `/info` 응답 확인

테스트 성공 명령:

```bash
curl http://172.16.43.114:32109/info -H "Host: flaskapp.onprem.local"
```

응답으로 FlaskApp의 `Employee Directory` HTML이 출력됨을 확인했다.

---

## 2. 주요 리소스 값

| 항목 | 값 |
| --- | --- |
| ECR repository | `080252689380.dkr.ecr.ap-northeast-2.amazonaws.com/flaskapp` |
| Push된 SHA 태그 | `d457c7e` |
| Push된 latest 태그 | `latest` |
| ECR digest | `sha256:90346afc397b73424d4a07a7fb1579085bfcf9d281bc1d9e0a6bd83449b6648c` |
| FlaskApp namespace | `flaskapp-prod` |
| ArgoCD namespace | `argocd` |
| Ingress namespace | `ingress-nginx` |
| IngressClass | `nginx` |
| Nginx Ingress HTTP NodePort | `32109` |
| Nginx Ingress HTTPS NodePort | `31232` |
| FlaskApp host | `flaskapp.onprem.local` |
| 확인된 Ingress Address | `172.16.43.114` |

---

## 3. ECR 이미지 Push

로컬 Mac에서 AWS CLI 인증 후 FlaskApp image를 push했다.

```bash
cd /Users/ireneminhee/Documents/GitHub/KOSA/Flaskapp
./build-push.sh
```

결과:

```text
080252689380.dkr.ecr.ap-northeast-2.amazonaws.com/flaskapp:d457c7e
080252689380.dkr.ecr.ap-northeast-2.amazonaws.com/flaskapp:latest
```

`d457c7e`는 Git commit SHA 기반 고정 태그이고, `latest`는 최신 이미지 별칭이다. 운영 추적성과 ArgoCD 변경 감지를 위해 Helm values에는 SHA 태그를 쓰는 것이 좋다.

```yaml
image:
  repository: 080252689380.dkr.ecr.ap-northeast-2.amazonaws.com/flaskapp
  tag: "d457c7e"
```

주의:

- AWS Access Key, Secret Key는 문서, GitHub issue, commit에 기록하지 않는다.
- `ecr-registry-secret.yaml` 같은 Secret manifest도 Git에 commit하지 않는다.

---

## 4. ArgoCD 설치

`bastion`에서 Kubernetes cluster에 ArgoCD를 설치했다.

```bash
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -

kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

확인:

```bash
kubectl get pods -n argocd
```

정상 기준:

```text
argocd-application-controller      1/1 Running
argocd-applicationset-controller   1/1 Running
argocd-dex-server                  1/1 Running
argocd-notifications-controller    1/1 Running
argocd-redis                       1/1 Running
argocd-repo-server                 1/1 Running
argocd-server                      1/1 Running
```

설치 중 `etcdserver: request timed out` 또는 `etcdserver: leader changed`가 발생할 수 있었다. 대부분 리소스가 적용된 상태라면 잠시 후 재확인하거나 동일 `kubectl apply`를 다시 실행한다.

---

## 5. root-app 적용

github infra 의 `infra/argocd/root-app.yaml`을 bastion으로 복사했다.

로컬에서:

```bash
scp /Users/ireneminhee/Documents/GitHub/KOSA/infra/argocd/root-app.yaml \
  kosa@172.16.44.100:~/root-app.yaml
```

`bastion`에서:

```bash
kubectl apply -f ~/root-app.yaml -n argocd
kubectl get applications.argoproj.io -n argocd
```

확인 결과:

```text
NAME       SYNC STATUS   HEALTH STATUS
flaskapp   OutOfSync     Progressing
root-app   Synced        Healthy
```

`root-app`은 정상 적용되었고, `flaskapp` Application이 생성되었다.

---

## 6. ECR Pull Secret 생성

> 현재 권장 방식은 `ecr-regcred`를 Deployment의 `imagePullSecrets`로 참조하고, `flaskapp-ecr-secret-refresh` CronJob으로 6시간마다 갱신하는 방식이다. 자세한 운영/복구 절차는 `flaskapp-ecr-pull-secret-refresh-runbook.md`를 참고한다.
>
> 아래 내용은 초기 배포 당시 사용한 임시 수동 방식이다.

처음 FlaskApp Pod는 아래 오류로 `ImagePullBackOff`가 발생했다.

```text
authorization failed: no basic auth credentials
```

원인:

- On-prem Kubernetes node는 EKS NodeRole이 없으므로 private ECR에서 이미지를 pull할 인증 정보가 필요하다.
- `flaskapp-prod` namespace에 Docker registry Secret을 만들고 ServiceAccount에 연결해야 한다.

로컬 Mac에서 AWS 인증이 되어 있었기 때문에 Secret YAML을 생성했다.

```bash
ECR_PASSWORD=$(aws ecr get-login-password --region ap-northeast-2)

kubectl create secret docker-registry ecr-registry-secret \
  --docker-server=080252689380.dkr.ecr.ap-northeast-2.amazonaws.com \
  --docker-username=AWS \
  --docker-password="$ECR_PASSWORD" \
  -n flaskapp-prod \
  --dry-run=client -o yaml > ecr-registry-secret.yaml
```

생성된 파일을 bastion으로 복사했다.

```bash
scp ecr-registry-secret.yaml kosa@172.16.44.100:~/ecr-registry-secret.yaml
```

`bastion`에서 적용했다.

```bash
kubectl apply -f ~/ecr-registry-secret.yaml

kubectl patch serviceaccount default -n flaskapp-prod \
  -p '{"imagePullSecrets":[{"name":"ecr-registry-secret"}]}'
```

Pod를 재시작했다.

```bash
kubectl rollout restart deployment flaskapp -n flaskapp-prod
kubectl get pods -n flaskapp-prod -w
```

새 Pod가 `Running` 상태가 되었다.

```text
flaskapp-59ffbc6d57-xwbvt   1/1   Running
```

주의:

- `ecr-registry-secret.yaml`에는 ECR 인증 토큰이 포함되므로 Git에 commit하지 않는다.
- 작업 후 로컬과 bastion의 YAML 파일은 삭제해도 된다.
- Kubernetes Secret 자체는 남겨두어야 image pull이 계속 가능하다.

---

## 7. Nginx Ingress Controller 설치

FlaskApp Ingress 리소스는 생성되었지만, 이를 처리할 Nginx Ingress Controller가 필요했다.

`bastion`에서 baremetal manifest를 적용했다.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.3/deploy/static/provider/baremetal/deploy.yaml
```

생성 리소스:

```text
namespace/ingress-nginx
service/ingress-nginx-controller
deployment/ingress-nginx-controller
ingressclass.networking.k8s.io/nginx
```

확인:

```bash
kubectl get ingressclass
kubectl get svc -n ingress-nginx
kubectl get pods -n ingress-nginx
```

확인 결과:

```text
NAME    CONTROLLER
nginx   k8s.io/ingress-nginx

ingress-nginx-controller   NodePort   80:32109/TCP,443:31232/TCP
ingress-nginx-controller   1/1 Running
```

설치 직후 controller Pod가 `ContainerCreating` 상태에서 멈춘 것처럼 보였으나, 원인은 admission Secret 생성 전 mount 실패였다.

```text
MountVolume.SetUp failed for volume "webhook-cert" : secret "ingress-nginx-admission" not found
```

`ingress-nginx-admission-create`와 `ingress-nginx-admission-patch` Job이 완료되면서 Secret이 생성되었고, 이후 controller가 정상 기동했다.

```bash
kubectl get secret ingress-nginx-admission -n ingress-nginx
kubectl get pods -n ingress-nginx
```

### 7.1 ADR-002 기준 확인

ADR-002의 핵심 요구사항은 **FlaskApp Service를 NodePort로 바꾸는 것**이 아니라, **NGINX Ingress Controller Service를 NodePort로 노출하는 것**이다.

현재 확인된 상태:

```text
ingress-nginx-controller   NodePort   80:32109/TCP,443:31232/TCP
```

따라서 현재 ingress-nginx 설치 상태는 ADR-002의 `HAProxy VIP -> Worker Node NodePort -> NGINX Ingress Controller -> FlaskApp Service(ClusterIP)` 구조와 맞다.

다만 설치 방식은 `ingress-nginx` Helm chart가 아니라 upstream baremetal manifest 적용 방식이다.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.3/deploy/static/provider/baremetal/deploy.yaml
```

운영상 선택지는 두 가지다.

| 선택지 | 의미 | 현재 판단 |
| --- | --- | --- |
| 현재 manifest 설치 유지 | 이미 NodePort로 동작 중인 ingress-nginx를 그대로 사용 | 당장 HAProxy 연동을 진행하기에 충분 |
| Helm chart로 재설치/전환 | ingress-nginx 자체도 Helm/ArgoCD 관리 대상으로 정리 | GitOps 일관성이 필요할 때 별도 작업으로 진행 |

현재 Day 8 목표가 FlaskApp ArgoCD 배포와 Ingress 연결 확인이라면, ingress-nginx를 Helm으로 즉시 재설치할 필요는 없다. 다음 우선순위는 HAProxy가 현재 확인된 NodePort `32109`, `31232`로 트래픽을 전달하도록 설정하는 것이다.

---

## 8. FlaskApp Ingress 연결 확인

Ingress 리소스 확인:

```bash
kubectl get ingress -n flaskapp-prod
kubectl describe ingress flaskapp-ingress -n flaskapp-prod
```

확인 결과:

```text
NAME               CLASS   HOSTS                   ADDRESS         PORTS
flaskapp-ingress   nginx   flaskapp.onprem.local   172.16.43.114   80
```

Backend도 정상 연결되었다.

```text
/   flaskapp-service:80 (10.244.69.196:80,10.244.79.69:80)
```

NodePort를 통해 FlaskApp 응답을 확인했다.

```bash
curl http://172.16.43.114:32109/info -H "Host: flaskapp.onprem.local"
```

응답:

```text
Employee Directory
인스턴스 아이디: i-fakeabc
가용 영역: us-fake-1a
```

---

## 9. 브라우저 접속 방법

현재는 테스트용으로 NodePort와 Host header를 사용한다.

`bastion`에서는 아래처럼 확인한다.

```bash
curl http://172.16.43.114:32109/info -H "Host: flaskapp.onprem.local"
```

로컬 브라우저에서 임시 확인하려면 로컬의 `/etc/hosts`에 아래를 추가한다.

```text
172.16.43.114 flaskapp.onprem.local
```

그 다음 브라우저에서 접속한다.

```text
http://flaskapp.onprem.local:32109/info
```

이 방식은 임시 테스트용이다. 최종적으로는 HAProxy가 80/443 요청을 Ingress Controller NodePort로 전달해야 한다.

---

## 10. 남은 작업

### 10.1 HAProxy 연동

현재 최종 사용자용 주소는 아직 완성되지 않았다. HAProxy backend가 Nginx Ingress Controller의 NodePort로 트래픽을 전달해야 한다.

현재 HTTP NodePort:

```text
32109
```

예상 backend 후보:

```text
172.16.43.110:32109
172.16.43.111:32109
172.16.43.114:32109
```

HAProxy 연동 후 목표 접속:

```text
http://flaskapp.onprem.local/info
```

### 10.2 Helm chart에 imagePullSecrets 반영

초기에는 ECR pull secret을 직접 생성하고 default ServiceAccount에 patch했다.

```bash
kubectl patch serviceaccount default -n flaskapp-prod \
  -p '{"imagePullSecrets":[{"name":"ecr-registry-secret"}]}'
```

이 방식은 동작하지만 GitOps 관점에서는 클러스터에 직접 넣은 변경이다. 현재는 Helm chart에 `imagePullSecrets` 설정을 추가하고, `ecr-regcred` Secret을 CronJob으로 주기 갱신하는 방식으로 정리했다.

현재 권장 문서:

```text
flaskapp-ecr-pull-secret-refresh-runbook.md
```

### 10.3 ingress-nginx Helm 전환 여부 결정

현재 ingress-nginx는 Helm release가 아니라 baremetal manifest로 설치되어 있다. ADR-002 관점에서는 이미 `NodePort`이므로 정상이다.

다만 ingress-nginx도 GitOps/Helm 관리 대상으로 맞추려면, 별도 전환 작업이 필요하다. 이때는 기존 manifest 기반 리소스와 Helm이 생성하는 리소스가 충돌할 수 있으므로, 바로 `helm install`을 덮어쓰기보다 아래 순서로 진행한다.

```bash
kubectl get all,ingressclass -n ingress-nginx
kubectl get validatingwebhookconfiguration | grep ingress
helm list -n ingress-nginx
```

Helm 전환 시 권장 values 예시:

```yaml
controller:
  service:
    type: NodePort
    nodePorts:
      http: 32109
      https: 31232
```

NodePort를 고정하면 HAProxy backend 설정이 임의 포트 변경에 영향을 받지 않는다.

### 10.4 image.tag를 SHA 태그로 고정

처음 Deployment는 `latest`를 pull했지만, 현재 infra repo의 `infra/helm/flaskapp/values.yaml`은 아래처럼 SHA 태그로 정리되어 있다.

```text
080252689380.dkr.ecr.ap-northeast-2.amazonaws.com/flaskapp:d457c7e
```

PR/commit에 반영되었는지 확인하면 된다.

### 10.5 ArgoCD 상태 재확인

```bash
kubectl get applications.argoproj.io -n argocd
kubectl get pods -n flaskapp-prod
kubectl get svc,ingress -n flaskapp-prod
kubectl get pods -n ingress-nginx
```

---

## 11. 문제 해결 기록

### 11.1 ArgoCD 설치 중 etcd 오류

증상:

```text
Error from server: etcdserver: request timed out
Error from server: etcdserver: leader changed
```

대응:

- 잠시 후 API VIP와 etcd readiness 확인
- 동일 `kubectl apply` 재실행 가능
- 네트워크/VIP가 불안정하면 팀원 B에게 control-plane/etcd 상태 확인 요청

확인 명령:

```bash
kubectl get nodes
ssh kosa@172.16.43.100 "curl -k https://172.16.43.99:6443/readyz"
```

### 11.2 FlaskApp ImagePullBackOff

증상:

```text
authorization failed: no basic auth credentials
403 Forbidden
```

원인:

- private ECR pull 인증 정보 없음
- ECR pull secret 만료 또는 미생성

해결:

- `ecr-refresh-aws-credentials` Secret 생성
- `flaskapp-ecr-secret-refresh` CronJob으로 `ecr-regcred` 생성/갱신
- Deployment가 `imagePullSecrets: ecr-regcred`를 사용하도록 Helm chart 반영
- 실패한 Pod 삭제 또는 Deployment 재시작

상세 절차:

```text
flaskapp-ecr-pull-secret-refresh-runbook.md
```

### 11.3 Ingress Controller ContainerCreating

증상:

```text
MountVolume.SetUp failed for volume "webhook-cert" : secret "ingress-nginx-admission" not found
```

원인:

- admission create/patch Job 완료 전 controller가 Secret을 mount하려고 시도

해결:

- admission Job 완료 확인
- Secret 생성 확인
- controller Pod가 `1/1 Running` 되는지 확인

```bash
kubectl get jobs -n ingress-nginx
kubectl get secret ingress-nginx-admission -n ingress-nginx
kubectl get pods -n ingress-nginx
```

---

## 12. 팀 공유용 요약

```text
FlaskApp ArgoCD 배포 및 Ingress 연결 확인 완료.

완료:
- ECR push 완료: flaskapp:d457c7e, latest
- ArgoCD 설치 완료
- root-app 적용 완료
- flaskapp Application 생성
- ECR pull secret 생성 및 ServiceAccount 연결
- FlaskApp Pod Running 확인
- Nginx Ingress Controller 설치 완료
- flaskapp-ingress nginx class 연결 확인
- NodePort 32109로 /info 응답 확인

테스트:
curl http://172.16.43.114:32109/info -H "Host: flaskapp.onprem.local"

남은 작업:
- HAProxy -> worker NodePort 32109 라우팅 설정
- flaskapp.onprem.local을 HAProxy VIP로 연결
- ingress-nginx Helm 전환 여부 결정
- values.yaml image.tag=d457c7e PR/commit 반영 여부 확인
- imagePullSecrets Helm chart 반영 여부 검토
```

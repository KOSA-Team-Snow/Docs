# ArgoCD UI 접속 및 GitHub 반영 확인 가이드

작성일: 2026-05-19

## 목적

ArgoCD UI에 접속하는 방법과, GitHub에 push한 변경사항이 ArgoCD에 실제로 반영되었는지 commit SHA 기준으로 확인하는 절차를 정리한다.

이 문서는 FlaskApp 배포 확인을 기준으로 작성한다.

## 현재 GitOps 구조

ArgoCD는 FlaskApp 소스 repo가 아니라 `infra` repo를 바라본다.

```text
GitHub infra repo
  ↓
infra/argocd/apps/flaskapp.yaml
  ↓
path: helm/flaskapp
  ↓
ArgoCD Application: flaskapp
  ↓
Kubernetes namespace: flaskapp-prod
```

따라서 ArgoCD의 sync revision은 기본적으로 `infra` repo의 commit SHA이다.

FlaskApp 소스코드를 수정한 경우에는 보통 다음 흐름을 거친다.

```text
Flaskapp repo 변경
  ↓
Docker image build/push
  ↓
ECR image tag 생성, 보통 Flaskapp git SHA 7자리
  ↓
infra/helm/flaskapp/values.yaml image.tag 수정
  ↓
infra repo commit/push
  ↓
ArgoCD가 infra repo 변경을 감지하고 배포
```

즉, ArgoCD에서 보는 Git SHA와 앱 이미지 태그 SHA는 다를 수 있다.

- ArgoCD sync revision: `infra` repo commit SHA
- FlaskApp image tag: `Flaskapp` repo commit SHA 기반 태그

## ArgoCD UI 접속: 임시 방식

ArgoCD UI를 바로 외부에 노출하지 않은 상태에서는 `kubectl port-forward`와 SSH 터널을 사용한다.

### 1. Bastion에서 ArgoCD Server Service 확인

```bash
kubectl get svc -n argocd
```

보통 다음 Service를 사용한다.

```text
argocd-server
```

### 2. Bastion에서 port-forward 실행

```bash
kubectl port-forward -n argocd svc/argocd-server 8081:443
```

이 터미널은 계속 켜둔다.

### 3. 로컬 PC에서 SSH 터널 실행

로컬 PC 터미널에서 Bastion으로 터널을 연다.

```bash
ssh -L 8081:127.0.0.1:8081 kosa@172.16.44.100
```

### 4. 로컬 브라우저에서 접속

```text
https://127.0.0.1:8081
```

인증서 경고가 뜨면 내부 테스트용 접속이므로 고급 설정에서 계속 진행한다.

## ArgoCD admin 비밀번호 확인

초기 admin secret이 남아 있는 경우:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

로그인 정보:

```text
Username: admin
Password: 위 명령어 출력값
```

`argocd-initial-admin-secret`이 없으면 이미 초기 비밀번호가 변경된 상태일 수 있다. 이 경우 팀에서 정한 admin 비밀번호를 사용한다.

주의: 비밀번호 출력 결과를 문서, issue, PR comment, 채팅에 남기지 않는다.

## ArgoCD UI 접속을 조금 편하게 하기

매번 긴 SSH 명령을 치기 싫으면 로컬 PC의 `~/.ssh/config`에 터널 설정을 저장할 수 있다.

```sshconfig
Host kosa-bastion-argocd
    HostName <BASTION_IP>
    User kosa
    IdentityFile <KEY_PATH>
    LocalForward 8081 127.0.0.1:8081
```

이후 로컬 PC에서는 다음만 실행한다.

```bash
ssh kosa-bastion-argocd
```

Bastion에서는 port-forward 명령을 alias로 등록할 수 있다.

```bash
echo "alias argocd-ui='kubectl port-forward -n argocd svc/argocd-server 8081:443'" >> ~/.bashrc
source ~/.bashrc
```

이후 Bastion에서는 다음만 실행한다.

```bash
argocd-ui
```

장기적으로는 `argocd.onprem.local` Ingress/HAProxy 경로를 구성하면 터널 없이 접속할 수 있다. 다만 이 경우 인증, TLS, 접근 제어를 함께 정리해야 한다.

## GitHub push가 ArgoCD에 반영됐는지 확인

### 1. infra repo 최신 commit 확인

ArgoCD가 바라보는 repo는 `infra`이다.

로컬 또는 GitHub에서 최신 commit SHA를 확인한다.

```bash
cd /Users/ireneminhee/Documents/GitHub/KOSA/infra
git rev-parse HEAD
```

짧은 SHA만 확인하려면:

```bash
git rev-parse --short HEAD
```

원격 기준 최신 commit을 확인하려면:

```bash
git ls-remote origin HEAD
```

### 2. ArgoCD가 보고 있는 revision 확인

Bastion에서 실행한다.

```bash
kubectl -n argocd get app flaskapp \
  -o jsonpath='{.status.sync.revision}{"\n"}'
```

이 값이 GitHub `infra` repo의 최신 commit SHA와 같으면 ArgoCD가 최신 Git revision을 보고 있는 것이다.

상태를 같이 보려면:

```bash
kubectl -n argocd get app flaskapp \
  -o jsonpath='sync={.status.sync.status} health={.status.health.status} revision={.status.sync.revision}{"\n"}'
```

정상 예시:

```text
sync=Synced health=Healthy revision=<infra-commit-sha>
```

### 3. UI에서 확인하는 방법

ArgoCD UI에서 `flaskapp` Application을 연다.

확인할 값:

- `SYNC STATUS`: `Synced`
- `HEALTH STATUS`: `Healthy`
- `LAST SYNC RESULT`: 성공 여부
- `REVISION`: GitHub infra repo의 commit SHA와 일치하는지

GitHub에 push한 뒤 UI의 revision이 아직 예전 값이면 ArgoCD가 아직 refresh하지 않았거나 sync가 진행 중인 상태일 수 있다.

## ArgoCD refresh / sync 강제 확인

자동 sync가 켜져 있어도 GitHub 변경 감지까지 시간이 걸릴 수 있다. 강제로 refresh하려면:

```bash
kubectl -n argocd annotate app flaskapp \
  argocd.argoproj.io/refresh=hard \
  --overwrite
```

이후 상태 확인:

```bash
kubectl -n argocd get app flaskapp
```

자세한 상태:

```bash
kubectl -n argocd describe app flaskapp
```

UI에서는 `REFRESH` 버튼을 눌러 Git 상태를 다시 읽게 할 수 있다. 필요하면 `SYNC` 버튼으로 수동 sync를 실행한다.

## values.yaml image.tag 반영 확인

FlaskApp 이미지 태그를 바꾼 경우에는 ArgoCD revision뿐 아니라 실제 Deployment image도 확인해야 한다.

infra repo에서 기대하는 태그 확인:

```bash
cd /Users/ireneminhee/Documents/GitHub/KOSA/infra
rg -n "tag:" helm/flaskapp/values.yaml
```

클러스터에 적용된 Deployment image 확인:

```bash
kubectl -n flaskapp-prod get deploy flaskapp \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

예시:

```text
080252689380.dkr.ecr.ap-northeast-2.amazonaws.com/flaskapp:d457c7e
```

여기서 태그가 `values.yaml`의 `image.tag`와 같아야 한다.

Pod가 실제 새 이미지로 떠 있는지 확인:

```bash
kubectl -n flaskapp-prod get pods -l app=flaskapp
kubectl -n flaskapp-prod rollout status deployment/flaskapp
```

## 배포 반영 전체 확인 순서

GitHub에 infra 변경사항을 push한 뒤 다음 순서로 확인한다.

```bash
# 1. ArgoCD Application 상태
kubectl -n argocd get app flaskapp

# 2. ArgoCD가 보고 있는 Git revision
kubectl -n argocd get app flaskapp \
  -o jsonpath='{.status.sync.revision}{"\n"}'

# 3. 필요 시 hard refresh
kubectl -n argocd annotate app flaskapp \
  argocd.argoproj.io/refresh=hard \
  --overwrite

# 4. Deployment rollout 확인
kubectl -n flaskapp-prod rollout status deployment/flaskapp

# 5. Pod 상태 확인
kubectl -n flaskapp-prod get pods

# 6. 적용된 image 확인
kubectl -n flaskapp-prod get deploy flaskapp \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

## 자주 헷갈리는 점

### GitHub에 push했는데 ArgoCD가 안 바뀐 것처럼 보임

확인할 것:

```bash
kubectl -n argocd get app flaskapp \
  -o jsonpath='{.status.sync.status} {.status.health.status} {.status.sync.revision}{"\n"}'
```

- revision이 예전 SHA이면 ArgoCD가 아직 Git 변경을 읽지 못한 상태이다.
- revision은 최신인데 OutOfSync이면 sync가 아직 안 된 상태이다.
- Synced인데 Pod image가 예전이면 `values.yaml image.tag`가 실제로 바뀌었는지 확인한다.

### FlaskApp repo commit SHA와 ArgoCD revision이 다름

정상일 수 있다.

ArgoCD는 `infra` repo를 보고, FlaskApp image tag는 `Flaskapp` repo commit SHA를 사용할 수 있다. 따라서 두 SHA는 역할이 다르다.

### UI 접속이 매번 귀찮음

임시로는 SSH config와 alias를 사용한다.

장기적으로는 ArgoCD용 Ingress와 HAProxy 라우팅을 구성한다.

예상 목표:

```text
https://argocd.onprem.local
```

## 관련 파일

```text
infra/argocd/apps/flaskapp.yaml
infra/argocd/root-app.yaml
infra/helm/flaskapp/values.yaml
infra/helm/flaskapp/templates/deployment.yaml
```

관련 문서:

```text
operation/k8s/argocd-flaskapp-ingress-runbook.md
operation/k8s/ecr-build-push-guide.md
operation/k8s/helm-ingress-guide.md
```

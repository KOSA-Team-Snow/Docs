# FlaskApp ECR Pull Secret 자동 갱신 및 ImagePullBackOff 복구 기록

작성일: 2026-05-19

## 목적

`flaskapp` Deployment가 private ECR 이미지를 pull하지 못해 `ImagePullBackOff` 상태가 되었을 때 원인을 확인하고, ECR pull secret 자동 갱신 구조를 적용해 롤아웃을 복구한다.

이 문서는 DB 연결 절차를 다루지 않는다. DB 연결과 앱 테스트는 `flaskapp-db-connect-runbook.md`를 참고한다.

## 배경

On-prem Kubernetes worker node는 EKS node role을 사용하지 않는다. 따라서 private ECR 이미지를 pull하려면 Kubernetes docker-registry Secret이 필요하다.

ECR 로그인 토큰은 장기 비밀번호가 아니라 임시 토큰이다. `aws ecr get-login-password`로 받은 값은 약 12시간 후 만료되므로, 수동으로 만든 pull secret만으로는 새 Pod 생성 시 다시 `ImagePullBackOff`가 발생할 수 있다.

## 발생 증상

`flaskapp` rollout이 다음 상태에서 멈췄다.

```text
Waiting for deployment "flaskapp" rollout to finish: 1 out of 2 new replicas have been updated...
```

Pod 확인 결과 새 Pod 하나가 `ImagePullBackOff`였다.

```bash
kubectl get pods -n flaskapp-prod -l app=flaskapp -o wide
```

예시:

```text
flaskapp-699c699d8f-xg2dz   0/1   ImagePullBackOff
```

event 확인 결과 ECR에서 이미지 manifest를 조회할 때 `403 Forbidden`이 발생했다.

```bash
kubectl get events -n flaskapp-prod --sort-by=.lastTimestamp | tail -30
```

예시:

```text
Failed to pull image "080252689380.dkr.ecr.ap-northeast-2.amazonaws.com/flaskapp:d457c7e"
failed to resolve image: unexpected status from HEAD request ... 403 Forbidden
```

## 원인

Private ECR 이미지를 pull하기 위한 인증 정보가 worker node 또는 Pod에 안정적으로 제공되지 않았다.

일부 Pod가 `Running`이었던 이유는 해당 worker node에 이미지가 이미 캐시되어 있었기 때문이다. 이미지가 캐시되지 않은 node에서는 ECR 인증 실패로 새 Pod가 뜨지 못했다.

## 적용한 GitOps 변경

Infra repo의 Helm chart에 다음 변경을 적용했다.

대상 repo:

```text
infra
```

대상 파일:

```text
helm/flaskapp/Chart.yaml
helm/flaskapp/values.yaml
helm/flaskapp/templates/deployment.yaml
helm/flaskapp/templates/ecr-secret-refresh.yaml
```

### Deployment imagePullSecrets 추가

`flaskapp` Deployment가 ECR pull secret을 명시적으로 사용하도록 했다.

```yaml
imagePullSecrets:
  - name: ecr-regcred
```

### ECR secret refresh CronJob 추가

6시간마다 `ecr-regcred` docker-registry Secret을 갱신하는 CronJob을 추가했다.

최종 설정:

```yaml
ecrSecretRefresh:
  enabled: true
  secretName: ecr-regcred
  registry: 080252689380.dkr.ecr.ap-northeast-2.amazonaws.com
  region: ap-northeast-2
  schedule: "0 */6 * * *"
  awsCredentialsSecretName: ecr-refresh-aws-credentials
  serviceAccountName: ecr-secret-refresher
  image:
    repository: alpine/k8s
    tag: "1.30.14"        # 1.30.1은 존재하지 않아 ImagePullBackOff 발생 → 1.30.14로 수정
    pullPolicy: IfNotPresent
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 3
  backoffLimit: 3          # 초기값 1 → 3으로 강화 (issue #85)
  activeDeadlineSeconds: 600
  retryAttempts: 3
  retryInitialDelaySeconds: 5
```

운영 메모: Kubernetes API, etcd, AWS ECR API가 순간적으로 실패해도 바로 ArgoCD `Degraded`로 이어지지 않도록 CronJob은 `backoffLimit: 3`을 사용하고, 스크립트 내부에서 ECR password 조회, Secret apply, Secret annotate 단계를 각각 3회 재시도한다. 재시도 간격은 5초부터 시작해 2배씩 늘린다. Job 전체 실행 시간은 10분으로 제한해 긴 hang이 다음 CronJob 실행을 막지 않게 한다.

## 사전 Secret 생성

CronJob은 AWS ECR 로그인 토큰을 발급받기 위해 AWS credential이 필요하다. 이 값은 Git에 저장하지 않고 클러스터에 직접 Secret으로 생성한다.

```bash
kubectl -n flaskapp-prod create secret generic ecr-refresh-aws-credentials \
  --from-literal=AWS_ACCESS_KEY_ID='<ACCESS_KEY_ID>' \
  --from-literal=AWS_SECRET_ACCESS_KEY='<SECRET_ACCESS_KEY>' \
  --dry-run=client -o yaml | kubectl apply -f -
```

확인:

```bash
kubectl -n flaskapp-prod get secret ecr-refresh-aws-credentials
```

주의:

- AWS access key와 secret key는 문서, Git, issue, PR comment에 남기지 않는다.
- 화면 공유 중 Secret 값을 출력하지 않는다.

## 필요한 IAM 권한

AWS credential에는 최소한 ECR read 권한이 필요하다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ecr:GetAuthorizationToken",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer"
      ],
      "Resource": "arn:aws:ecr:ap-northeast-2:080252689380:repository/flaskapp"
    }
  ]
}
```

## ArgoCD 반영 확인

GitHub에 변경사항을 push한 뒤 ArgoCD가 최신 commit을 보고 있는지 확인한다.

```bash
kubectl -n argocd get app flaskapp -o jsonpath='{.status.sync.revision}{"\n"}'
```

GitHub 최신 commit SHA와 위 값이 같아야 한다. 다르면 ArgoCD가 아직 예전 commit을 보고 있는 것이다.

강제 refresh:

```bash
kubectl -n argocd annotate app flaskapp argocd.argoproj.io/refresh=hard --overwrite --request-timeout=60s
```

확인:

```bash
kubectl -n argocd get app flaskapp --request-timeout=60s
kubectl -n flaskapp-prod get cronjob --request-timeout=60s
```

정상적으로 반영되면 다음 CronJob이 보여야 한다.

```text
flaskapp-ecr-secret-refresh
```

## CronJob 수동 실행

6시간 주기를 기다리지 않고 한 번 즉시 실행한다.

```bash
JOB_NAME=ecr-secret-refresh-now-$(date +%s)
kubectl -n flaskapp-prod create job "$JOB_NAME" --from=cronjob/flaskapp-ecr-secret-refresh --request-timeout=60s
kubectl -n flaskapp-prod get pod -l job-name="$JOB_NAME" -w
```

정상 흐름:

```text
Pending
ContainerCreating
Running
Completed
```

`Completed`가 되면 로그를 확인한다.

```bash
kubectl -n flaskapp-prod logs job/"$JOB_NAME"
```

성공 로그:

```text
secret/ecr-regcred serverside-applied
secret/ecr-regcred annotated
```

Secret 생성 확인:

```bash
kubectl -n flaskapp-prod get secret ecr-regcred
```

## 실패한 Pod 복구

`ecr-regcred`가 생긴 뒤 `ImagePullBackOff` 상태의 Pod를 삭제하면 Deployment가 새 Pod를 만든다.

```bash
kubectl get pods -n flaskapp-prod -l app=flaskapp
kubectl delete pod <ImagePullBackOff-Pod-이름> -n flaskapp-prod
```

이미 실패 Pod가 사라졌다면 `NotFound`가 나올 수 있다. 이 경우 현재 Pod 상태가 `Running`이면 문제 없다.

롤아웃 확인:

```bash
kubectl rollout status deployment/flaskapp -n flaskapp-prod
```

성공 기준:

```text
deployment "flaskapp" successfully rolled out
```

ArgoCD 상태 확인:

```bash
kubectl -n argocd get app flaskapp
```

성공 기준:

```text
flaskapp   Synced   Healthy
```

## CronJob image tag 문제 대응

작업 중 CronJob Pod도 `ImagePullBackOff`가 되었다.

확인:

```bash
kubectl -n flaskapp-prod describe pod <ecr-secret-refresh-pod>
```

원인:

```text
docker.io/alpine/k8s:1.30.1: not found
```

즉 Helm values에 지정한 `alpine/k8s:1.30.1` 태그가 존재하지 않았다.

임시 live patch:

```bash
kubectl -n flaskapp-prod patch cronjob flaskapp-ecr-secret-refresh \
  --type='json' \
  -p='[{"op":"replace","path":"/spec/jobTemplate/spec/template/spec/containers/0/image","value":"alpine/k8s:1.30.14"}]' \
  --request-timeout=60s
```

확인:

```bash
kubectl -n flaskapp-prod get cronjob flaskapp-ecr-secret-refresh \
  -o jsonpath='{.spec.jobTemplate.spec.template.spec.containers[0].image}{"\n"}'
```

기대값:

```text
alpine/k8s:1.30.14
```

주의: live patch는 클러스터에만 적용된다. 반드시 `infra/helm/flaskapp/values.yaml`도 같은 값으로 수정하고 commit/push해야 ArgoCD가 다시 `1.30.1`로 되돌리지 않는다.

## etcd / Kubernetes API 일시 오류

작업 중 다음 오류가 발생했다.

```text
etcdserver: request timed out
etcdserver: leader changed
Unable to connect to the server: dial tcp 172.16.43.99:6443: connect: no route to host
```

대응:

- 같은 명령을 `--request-timeout=60s`와 함께 재시도한다.
- `etcdserver: leader changed`는 etcd leader 변경 순간에 발생할 수 있으며, 일회성이면 재시도로 넘어간다.
- `no route to host`가 반복되면 Kubernetes API VIP 또는 control-plane 상태를 먼저 확인한다.

확인 명령:

```bash
kubectl get nodes --request-timeout=60s
kubectl get pods -n kube-system -o wide --request-timeout=60s
kubectl get pods -n argocd -o wide --request-timeout=60s
```

## 최종 확인된 상태

최종적으로 다음 상태를 확인했다.

```bash
kubectl rollout status deployment/flaskapp -n flaskapp-prod
kubectl -n argocd get app flaskapp
```

결과:

```text
deployment "flaskapp" successfully rolled out
flaskapp   Synced   Healthy
```

`ecr-regcred`도 정상 생성되었다.

```bash
kubectl -n flaskapp-prod get secret ecr-regcred
```

## 자주 쓰는 명령 모음

ArgoCD가 보고 있는 commit 확인:

```bash
kubectl -n argocd get app flaskapp -o jsonpath='{.status.sync.revision}{"\n"}'
```

ArgoCD hard refresh:

```bash
kubectl -n argocd annotate app flaskapp argocd.argoproj.io/refresh=hard --overwrite --request-timeout=60s
```

CronJob 확인:

```bash
kubectl -n flaskapp-prod get cronjob flaskapp-ecr-secret-refresh
```

CronJob 이미지 확인:

```bash
kubectl -n flaskapp-prod get cronjob flaskapp-ecr-secret-refresh \
  -o jsonpath='{.spec.jobTemplate.spec.template.spec.containers[0].image}{"\n"}'
```

ECR refresh Job 즉시 실행:

```bash
JOB_NAME=ecr-secret-refresh-now-$(date +%s)
kubectl -n flaskapp-prod create job "$JOB_NAME" --from=cronjob/flaskapp-ecr-secret-refresh --request-timeout=60s
kubectl -n flaskapp-prod get pod -l job-name="$JOB_NAME" -w
kubectl -n flaskapp-prod logs job/"$JOB_NAME"
```

ECR pull secret 확인:

```bash
kubectl -n flaskapp-prod get secret ecr-regcred
```

Flaskapp Pod 확인:

```bash
kubectl get pods -n flaskapp-prod -l app=flaskapp -o wide
```

Rollout 확인:

```bash
kubectl rollout status deployment/flaskapp -n flaskapp-prod
```

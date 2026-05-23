# FlaskApp ECR 이미지 빌드 및 Push 가이드

> 담당: 팀원 C (@ireneminhee)  
> 관련 이슈: #18

> **2026-05-23 업데이트**: ECR push 및 values.yaml 업데이트는 GitHub Actions CD로 자동화되었다.
> Flaskapp main에 merge하면 자동으로 처리된다. 아래 수동 방법은 긴급 상황 또는 로컬 테스트용으로만 사용한다.
> 자동화 파이프라인 전체 구조는 `github-actions-cicd-guide.md` 참고.

---

## 개요

FlaskApp 소스코드를 Docker 이미지로 빌드하고 AWS ECR에 push한 뒤, infra 레포의 `values.yaml`을 업데이트하면 ArgoCD가 자동으로 새 이미지를 감지해 배포한다.

```
[자동] Flaskapp main push
  ↓ GitHub Actions CD 실행
  ↓ ECR push (git SHA 태그)
  ↓ infra 레포 values.yaml image.tag 자동 업데이트
  ↓ ArgoCD 감지 → FlaskApp Pod 롤링 업데이트

[수동] build-push.sh 실행 (Flaskapp 레포)
  ↓ AWS 자격증명 확인
  ↓ ECR 로그인
  ↓ docker build --platform linux/amd64
  ↓ ECR push (git SHA 태그)
  ↓ 스크립트가 SHA 값 출력
infra 레포 values.yaml image.tag를 해당 SHA로 수동 변경 후 git push
  ↓
ArgoCD 감지 (automated sync)
  ↓ helm upgrade 자동 실행
FlaskApp Pod 롤링 업데이트
```

---

## ECR 레포지토리

| 항목 | 값 |
|------|-----|
| 레포지토리 URL | `080252689380.dkr.ecr.ap-northeast-2.amazonaws.com/flaskapp` |
| 리전 | `ap-northeast-2` |
| 이미지 이름 | `flaskapp` |

---

## 이미지 태그 전략

git commit SHA 앞 7자리를 이미지 태그로 사용한다. `latest` 태그도 함께 push한다.

| 태그 | 예시 | 용도 |
|------|------|------|
| git SHA | `a1b2c3d` | ArgoCD 배포 추적용 (변경 감지) |

SHA 태그를 사용하면 어떤 커밋이 클러스터에 배포되어 있는지 정확히 알 수 있다. `latest`만 사용하면 이미지가 교체되어도 ArgoCD가 변경을 감지하지 못한다.

> ECR 태그 불변성(immutable tag) 정책이 적용되어 있어 `latest` 태그는 push하지 않는다.

---

## 사전 요건

### AWS 자격증명 설정

`build-push.sh`는 실행 시 자격증명을 자동으로 검증한다. 실패하면 스크립트가 즉시 종료된다.

```bash
export AWS_ACCESS_KEY_ID=<액세스키>
export AWS_SECRET_ACCESS_KEY=<시크릿키>
export AWS_DEFAULT_REGION=ap-northeast-2

# 설정 확인
aws sts get-caller-identity
```

IAM Role이 부여된 EC2(예: bastion)에서 실행하는 경우 `export` 없이도 자동 인증된다.

---

## build-push.sh 사용 방법

스크립트는 Flaskapp 레포 루트에 위치한다.

```bash
# Flaskapp 레포 루트에서 실행
./build-push.sh
```

### 실행 흐름

1. AWS 자격증명 확인 (`aws sts get-caller-identity`)
2. ECR 로그인 (`aws ecr get-login-password | docker login`)
3. Docker 빌드 (`--platform linux/amd64`)
4. ECR push (SHA 태그 + latest 태그)
5. 다음 단계 안내 출력

### 출력 예시

```
[INFO] 빌드 시작
  이미지: 080252689380.dkr.ecr.ap-northeast-2.amazonaws.com/flaskapp:a1b2c3d

[INFO] ECR 로그인
[INFO] Docker 빌드
[INFO] ECR Push

[DONE] 080252689380.dkr.ecr.ap-northeast-2.amazonaws.com/flaskapp:a1b2c3d

다음 단계: infra 레포에서 아래 값으로 image.tag를 업데이트하세요
  파일: helm/flaskapp/values.yaml
  tag: "a1b2c3d"
```

---

## --platform linux/amd64 이유

Mac M칩(arm64 아키텍처)에서 빌드하면 K8s 워커 노드(x86_64 아키텍처)와 아키텍처가 맞지 않아 Pod가 실행되지 않는다. `--platform linux/amd64`를 지정하면 플랫폼에 관계없이 x86_64 이미지를 빌드한다.

---

## infra 레포 values.yaml 업데이트

`build-push.sh` 실행 후 출력된 SHA 태그를 `infra` 레포의 `values.yaml`에 반영한다.

```yaml
# infra/helm/flaskapp/values.yaml
image:
  repository: 080252689380.dkr.ecr.ap-northeast-2.amazonaws.com/flaskapp
  tag: "a1b2c3d"   # build-push.sh 출력 SHA로 교체
  pullPolicy: IfNotPresent
```

변경 후 커밋 + push하면 ArgoCD가 자동으로 감지해 배포한다.

```bash
# infra 레포에서
git add helm/flaskapp/values.yaml
git commit -m "feat(helm): update flaskapp image tag to a1b2c3d"
git push
```

---

## ArgoCD 배포 확인

infra 레포에 push 후 약 3분 내에 ArgoCD가 자동 sync를 수행한다.

```bash
# ArgoCD CLI로 상태 확인
argocd app get flaskapp

# 즉시 sync 원할 때
argocd app sync flaskapp

# Pod 롤링 업데이트 확인
kubectl get pods -n flaskapp-prod -w
```

ArgoCD UI에서도 `flaskapp` Application의 상태가 `Synced` → `Healthy`로 바뀌는 것을 확인할 수 있다.

---

## 문제 해결

| 증상 | 원인 | 해결 |
|------|------|------|
| `AWS 자격증명이 설정되지 않았습니다` | `AWS_ACCESS_KEY_ID` 미설정 | `export AWS_ACCESS_KEY_ID=...` 후 재실행 |
| `exec format error` (Pod CrashLoop) | `--platform` 없이 arm64로 빌드 | `build-push.sh` 사용 (자동으로 `linux/amd64` 적용) |
| ArgoCD가 새 이미지를 감지 못함 | `values.yaml` 태그 미변경 | `image.tag`를 새 SHA로 업데이트 후 push |
| ECR 로그인 실패 | 자격증명 만료 또는 권한 없음 | IAM 권한 확인 (`ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`) |
| `ImagePullBackOff` / ECR `403 Forbidden` | 클러스터의 ECR pull secret 없음 또는 만료 | `flaskapp-ecr-pull-secret-refresh-runbook.md` 참고 |

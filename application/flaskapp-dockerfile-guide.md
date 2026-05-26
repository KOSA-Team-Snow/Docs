# FlaskApp Dockerfile 작성 및 이미지 빌드 가이드

> 담당: 정 @ireneminhee / 부 @realhjung  
> 관련 이슈: #13  
> 관련 레포: [KOSA-Team-Snow/Flaskapp](https://github.com/KOSA-Team-Snow/Flaskapp)

---

## 개요

FlaskApp 소스코드를 컨테이너 이미지로 패키징하기 위한 Dockerfile 구조, 로컬 빌드, ECR push 흐름을 정리한다.

## 현재 운영 이미지

| 항목 | 값 |
|---|---|
| ECR repository | `080252689380.dkr.ecr.ap-northeast-2.amazonaws.com/flaskapp` |
| 현재 운영 tag | `18b68fe` |
| 운영 namespace | `flaskapp-prod` |
| 운영 배포 방식 | ArgoCD + Helm chart `infra/helm/flaskapp` |

---

## Dockerfile 전체 내용

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV FLASK_APP=application.py

EXPOSE 80

CMD ["flask", "run", "--host=0.0.0.0", "--port=80"]
```

---

## 각 라인 설명

| 명령어 | 내용 |
|--------|------|
| `FROM python:3.11-slim` | 베이스 이미지. slim은 불필요한 패키지를 제외한 경량 버전 |
| `WORKDIR /app` | 컨테이너 안에서 작업할 디렉터리 설정 |
| `COPY requirements.txt .` | 패키지 목록만 먼저 복사 (캐시 활용을 위해 코드보다 먼저) |
| `RUN pip install --no-cache-dir -r requirements.txt` | 패키지 설치. `--no-cache-dir`로 이미지 크기 최소화 |
| `COPY . .` | 나머지 소스코드 전체 복사 |
| `ENV FLASK_APP=application.py` | Flask 진입점 지정. 없으면 `flask run` 실패 |
| `EXPOSE 80` | 컨테이너가 사용할 포트 명시 (문서 역할) |
| `CMD [...]` | 컨테이너 실행 시 자동으로 실행될 명령어 |

### 베이스 이미지로 `python:3.11-slim`을 선택한 이유

- `python:3.11` 대비 이미지 크기가 약 3배 작음
- FlaskApp 실행에 필요한 패키지는 `requirements.txt`로 직접 설치하므로 불필요한 도구 불필요
- `slim` 버전은 `gcc` 등 빌드 도구가 없어 일부 패키지 설치 실패 가능성 있으나, FlaskApp의 의존성(`Flask`, `boto3`, `pillow` 등)은 모두 정상 설치 확인

---

## .dockerignore

빌드 시 이미지에 포함하지 않을 파일 목록.

```
__pycache__/
*.pyc
.env
.git/
CLAUDE.md
```

`.env`와 시크릿 파일이 이미지에 포함되지 않도록 반드시 작성해야 한다.

---

## 빌드 방법

```bash
# Flaskapp 레포 루트에서 실행
cd /Users/snowkwon/Desktop/KOSA/FlaskApp

docker build -t flaskapp:latest .
```

Kubernetes 노드가 x86_64이므로 배포용 이미지는 `linux/amd64`로 빌드하는 것이 안전하다.

```bash
docker build --platform linux/amd64 -t flaskapp:<git-sha> .
```

### 빌드 성공 확인

```
[+] Building 27.8s (11/11) FINISHED
```

`[11/11] FINISHED` 메시지가 나오면 빌드 성공.

빌드된 이미지 확인:

```bash
docker images | grep flaskapp
```

---

## 로컬 실행 방법

```bash
docker run -p 8080:80 \
  -e DATABASE_HOST=172.16.43.160 \
  -e DATABASE_USER=flaskapp \
  -e DATABASE_PASSWORD=<DB비밀번호> \
  -e DATABASE_DB_NAME=flaskapp \
  -e AWS_DEFAULT_REGION=ap-northeast-2 \
  -e PHOTOS_BUCKET=<S3버킷명> \
  flaskapp:latest
```

브라우저에서 `http://localhost:8080` 접속하여 확인.

### 환경변수 주입 방법 (`-e` 옵션)

컨테이너는 외부 환경(DB, S3)에 접근하기 위해 환경변수가 필요하다.  
로컬 테스트 시 `docker run -e KEY=VALUE` 형태로 주입한다.  
K8s 배포 시에는 ConfigMap과 Secret으로 주입한다. (Day 5 참고)

| 환경변수 | 종류 | 값 |
|----------|------|-----|
| `DATABASE_HOST` | ConfigMap | `172.16.43.160` |
| `DATABASE_USER` | ConfigMap | `flaskapp` |
| `DATABASE_DB_NAME` | ConfigMap | `flaskapp` |
| `AWS_DEFAULT_REGION` | ConfigMap | `ap-northeast-2` |
| `PHOTOS_BUCKET` | ConfigMap | S3 버킷명 |
| `DATABASE_PASSWORD` | Secret | DB 비밀번호 |
| `FLASK_SECRET` | Secret | 임의 문자열 |

> `DYNAMO_MODE`는 설정하지 않는다. 설정 시 MariaDB 대신 DynamoDB로 전환됨.

## ECR Push 흐름

`FlaskApp/build-push.sh`는 다음 흐름으로 이미지를 빌드하고 ECR에 push한다.

```text
AWS 자격증명 확인
-> git commit SHA 7자리로 image tag 생성
-> ECR login
-> docker build --platform linux/amd64
-> ECR에 <sha> tag push
-> latest tag push
-> infra repo의 helm/flaskapp/values.yaml image.tag 업데이트 안내
```

실행:

```bash
cd /Users/snowkwon/Desktop/KOSA/FlaskApp
bash build-push.sh
```

현재 운영 이미지 tag는 infra repo의 `infra/helm/flaskapp/values.yaml`에서 관리한다.

```yaml
image:
  repository: 080252689380.dkr.ecr.ap-northeast-2.amazonaws.com/flaskapp
  tag: '18b68fe'
```

---

## 로그 확인

```bash
# 실행 중인 컨테이너 확인
docker ps

# 로그 보기
docker logs <컨테이너ID>

# 실시간 로그
docker logs -f <컨테이너ID>
```

---

## 참고

- 환경변수 전체 목록: [flaskapp-env-analysis.md](./flaskapp-env-analysis.md)
- 컨테이너 실행 가이드: [flaskapp-container-run-guide.md](./flaskapp-container-run-guide.md)
- K8s 배포 시 ConfigMap/Secret 연동: `infra/helm/flaskapp/templates/configmap.yaml`, `infra/helm/flaskapp/templates/deployment.yaml`
- ECR pull secret 갱신: `infra/helm/flaskapp/templates/ecr-secret-refresh.yaml`

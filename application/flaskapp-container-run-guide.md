# FlaskApp 컨테이너 실행 가이드

> 로컬에서 FlaskApp Docker 컨테이너를 실행하고 최소 동작을 확인하는 방법

---

## 사전 준비

- Docker Desktop 실행 중
- `flaskapp:latest` 또는 Git SHA 태그 이미지 빌드 완료 ([flaskapp-dockerfile-guide.md](./flaskapp-dockerfile-guide.md) 참고)

---

## 필수 환경변수

`config.py`에서 아래 환경변수를 읽는다.

| 변수명 | 필수 여부 | 설명 |
|--------|-----------|------|
| `PHOTOS_BUCKET` | **필수** | S3 버킷명. 없으면 앱 자체가 뜨지 않음 |
| `DATABASE_HOST` | 사실상 필수 | MariaDB 주소. `/info`는 가능하지만 `/` 등 DB 기능은 실패 |
| `DATABASE_USER` | 사실상 필수 | DB 사용자명 |
| `DATABASE_PASSWORD` | 사실상 필수 | DB 비밀번호 |
| `DATABASE_DB_NAME` | 사실상 필수 | DB 이름 |
| `AWS_DEFAULT_REGION` | 권장 | AWS 리전 |
| `FLASK_SECRET` | 개선 필요 | 현재 config.py에 하드코딩 → 추후 Secret으로 교체 예정 |

`.env.example` 파일에서 전체 목록 확인 가능.

---

## 로컬 실행 방법

### 방법 1 — docker-run.sh 스크립트 사용 (권장)

```bash
bash docker-run.sh
```

### 방법 2 — 직접 실행

```bash
docker run -d \
  -p 8080:80 \
  --name flaskapp-test \
  -e PHOTOS_BUCKET=flaskapp-proddata-kosa-project-team3-snow-lai9z \
  -e DATABASE_HOST=172.16.43.160 \
  -e DATABASE_USER=flaskapp \
  -e DATABASE_DB_NAME=flaskapp \
  -e DATABASE_PASSWORD='<DB 비밀번호>' \
  -e AWS_DEFAULT_REGION=ap-northeast-2 \
  flaskapp:latest
```

> DB/S3 미연결 상태에서 컨테이너 기동만 확인하려면 더미 값으로 실행할 수 있다.  
> 단, 더미 DB 값이면 `/info`만 확인하고 `/`는 500이 날 수 있다.

---

## 동작 확인

```bash
# /info 라우트로 접속 확인 (DB 없어도 응답)
curl http://localhost:8080/info

# DB까지 연결된 경우 홈 화면 확인
curl -I http://localhost:8080/

# 로그 확인
docker logs flaskapp-test
```

> `/` 홈 라우트는 DB 연결이 없으면 500 에러 발생 — 정상.  
> DB 없이 테스트할 때는 `/info` 라우트 사용.

---

## 컨테이너 정리

```bash
docker stop flaskapp-test && docker rm flaskapp-test
```

---

## 주의사항

- `PHOTOS_BUCKET`이 없으면 앱이 시작 단계에서 `KeyError`로 종료됨
- `.env` 파일은 절대 커밋하지 않음 (`.gitignore`로 차단)
- 실제 운영 환경에서는 K8s ConfigMap/Secret으로 환경변수를 주입함
- 현재 운영 배포는 로컬 `docker run`이 아니라 ArgoCD/Helm을 통해 `flaskapp-prod` namespace에 배포됨

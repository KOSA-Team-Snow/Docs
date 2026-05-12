# FlaskApp 컨테이너 실행 가이드

> 로컬에서 FlaskApp Docker 컨테이너를 실행하고 동작을 확인하는 방법

---

## 사전 준비

- Docker Desktop 실행 중
- `flaskapp:latest` 이미지 빌드 완료 (`flaskapp-dockerfile-guide.md` 참고)

---

## 필수 환경변수

`config.py`에서 아래 환경변수를 읽는다.

| 변수명 | 필수 여부 | 설명 |
|--------|-----------|------|
| `PHOTOS_BUCKET` | **필수** | S3 버킷명. 없으면 앱 자체가 뜨지 않음 |
| `DATABASE_HOST` | 선택 | MariaDB 주소. 없으면 DB 기능만 비활성 |
| `DATABASE_USER` | 선택 | DB 사용자명 |
| `DATABASE_PASSWORD` | 선택 | DB 비밀번호 |
| `DATABASE_DB_NAME` | 선택 | DB 이름 |
| `AWS_DEFAULT_REGION` | 선택 | AWS 리전 (기본값: ap-northeast-2) |
| `FLASK_SECRET` | - | 현재 config.py에 하드코딩 → 추후 Secret으로 교체 예정 |

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
  -e PHOTOS_BUCKET=dummy \
  -e DATABASE_HOST=dummy \
  -e DATABASE_USER=dummy \
  -e DATABASE_DB_NAME=dummy \
  -e DATABASE_PASSWORD=dummy \
  -e AWS_DEFAULT_REGION=ap-northeast-2 \
  flaskapp:latest
```

> DB/S3 미연결 상태에서 테스트할 경우 더미 값으로 실행 가능.  
> 실제 값은 `.env.example`을 복사해 `.env`로 만들어 사용.

---

## 동작 확인

```bash
# /info 라우트로 접속 확인 (DB 없어도 응답)
curl http://localhost:8080/info

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
- 실제 운영 환경에서는 K8s ConfigMap/Secret으로 환경변수를 주입함 (Day 5 참고)

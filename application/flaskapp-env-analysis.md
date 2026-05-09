# FlaskApp 환경변수 및 실행 조건 분석

## 문서 목적

FlaskApp 소스코드를 분석하여 컨테이너 실행에 필요한 환경변수 목록과 외부 의존성을 정리한다.

## 담당

| 구분 | 담당 |
|---|---|
| 정(Lead) | 팀원 C / @ireneminhee |
| 부(Partner) | 팀원 B / @realhjung |

## 환경변수 목록

| 변수명 | 필수 여부 | 설명 | 예시값 |
|---|---|---|---|
| `PHOTOS_BUCKET` | 필수 | AWS S3 버킷 이름 (이미지 저장용) | my-flaskapp-bucket |
| `DATABASE_HOST` | 필수 | MariaDB 서버 주소 | 172.16.0.160 |
| `DATABASE_USER` | 필수 | DB 접속 계정 | kosa |
| `DATABASE_PASSWORD` | 필수 | DB 접속 비밀번호 | kosa1004 |
| `DATABASE_DB_NAME` | 필수 | 사용할 DB 이름 | employees |
| `AWS_DEFAULT_REGION` | 필수 | AWS 리전 | ap-northeast-2 |
| `DYNAMO_MODE` | 선택 | 설정 시 MariaDB 대신 DynamoDB 사용 | 1 |

> `DATABASE_*` 변수는 코드상 None 기본값이지만, 없으면 DB 연결 시 앱이 죽음.  
> `FLASK_SECRET`은 현재 `config.py`에 하드코딩(`"something-random"`) — 프로덕션에서는 환경변수로 교체 필요.

## 외부 의존성

| 항목 | 설명 |
|---|---|
| MariaDB / MySQL | 직원 정보 저장 (employee 테이블) |
| AWS S3 | 직원 사진 이미지 저장 및 presigned URL 생성 |
| AWS 자격증명 | boto3가 S3 접근 시 필요 (IAM Role 또는 환경변수) |
| EC2 Instance Metadata | 인스턴스 ID, 가용영역 조회 (없으면 fake 값으로 fallback) |

## 앱이 하는 일

- 직원 정보(이름, 직책, 위치, 뱃지) 조회/추가/수정/삭제
- 직원 사진을 AWS S3에 업로드하고 presigned URL로 표시
- `/info` 페이지에서 EC2 인스턴스 ID, 가용영역 표시

## 필요한 패키지 (requirements.txt)

| 패키지 | 역할 |
|---|---|
| Flask, Flask-WTF | 웹 프레임워크 |
| mysql_connector_python | MariaDB 연결 |
| boto3 | AWS S3 연결 |
| pillow | 직원 사진 이미지 리사이즈 (120x160) |
| requests | EC2 메타데이터 조회 |

## 로컬 실행 방법

```bash
# 1. 패키지 설치
pip3 install -r requirements.txt

# 2. 환경변수 설정
export PHOTOS_BUCKET=<S3 버킷 이름>
export DATABASE_HOST=<MariaDB 호스트>
export DATABASE_USER=<DB 유저>
export DATABASE_PASSWORD=<DB 비밀번호>
export DATABASE_DB_NAME=employees
export AWS_DEFAULT_REGION=ap-northeast-2

# 3. DB 테이블 생성 (최초 1회)
mysql -h $DATABASE_HOST -u $DATABASE_USER -p$DATABASE_PASSWORD < database_create_tables.sql

# 4. 앱 실행
FLASK_APP=application.py flask run --host=0.0.0.0 --port=80
```

## 확인 필요 항목

- [ ] AWS S3 버킷 이름 팀 내 확정
- [ ] MariaDB 서버 IP 확정 (현재 예상: 172.16.0.160)
- [ ] DB 계정명 및 비밀번호 팀원 D와 협의
- [ ] AWS 자격증명 방식 확정 (IAM Role vs 환경변수)
- [ ] `FLASK_SECRET` 환경변수 처리 방식 결정

## 후속 문서

| 후속 작업 | 연결 이슈 |
|---|---|
| Dockerfile 작성 및 이미지 빌드 가이드 | Docs #13 |
| Kubernetes manifest 작성 가이드 | Docs #14 |

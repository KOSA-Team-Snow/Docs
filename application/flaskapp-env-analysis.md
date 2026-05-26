# FlaskApp 환경변수 및 실행 조건 분석

## 문서 목적

FlaskApp 소스코드를 기준으로 컨테이너 실행에 필요한 환경변수, 외부 의존성, 운영 시 주의할 점을 정리한다.

## 현재 운영 기준

| 항목 | 현재 값 |
|---|---|
| 운영 namespace | `flaskapp-prod` |
| 운영 이미지 | `080252689380.dkr.ecr.ap-northeast-2.amazonaws.com/flaskapp:18b68fe` |
| 운영 DB | MariaDB VM `172.16.43.160` |
| 운영 DB 이름 | `flaskapp` |
| S3 bucket | `flaskapp-proddata-kosa-project-team3-snow-lai9z` |
| Probe 경로 | `/info` |

## 담당

| 구분 | 담당 |
|---|---|
| 정(Lead) | 팀원 C / @ireneminhee |
| 부(Partner) | 팀원 B / @realhjung |

## 환경변수 목록

| 변수명 | 필수 여부 | 설명 | 예시값 |
|---|---|---|---|
| `PHOTOS_BUCKET` | 필수 | AWS S3 버킷 이름. 코드에서 `os.environ['PHOTOS_BUCKET']`로 직접 읽기 때문에 없으면 앱 시작 실패 | `flaskapp-proddata-kosa-project-team3-snow-lai9z` |
| `DATABASE_HOST` | 필수 | MariaDB 서버 주소. 없으면 앱 기동은 가능하나 DB 기능 사용 시 실패 | `172.16.43.160` |
| `DATABASE_USER` | 필수 | DB 접속 계정 | flaskapp |
| `DATABASE_PASSWORD` | 필수 | DB 접속 비밀번호 | K8s Secret에 저장 |
| `DATABASE_DB_NAME` | 필수 | 사용할 DB 이름 | flaskapp |
| `AWS_DEFAULT_REGION` | 권장 | boto3가 사용할 AWS 리전 | ap-northeast-2 |
| `DYNAMO_MODE` | 선택 | 설정 시 MariaDB 대신 DynamoDB 사용 (우리 프로젝트는 미사용) | 1 |
| `AWS_ACCESS_KEY_ID` | 선택 | IAM Role 없을 때 S3 접근용 AWS 자격증명 | - |
| `AWS_SECRET_ACCESS_KEY` | 선택 | IAM Role 없을 때 S3 접근용 AWS 자격증명 | - |
| `FLASK_SECRET` | 선택 | 현재 `config.py`에 하드코딩됨, 프로덕션에서는 환경변수로 교체 필요 | some-secret |

> `DATABASE_*` 변수는 코드상 일부가 `None` 기본값을 가질 수 있지만, `/`, `/add`, `/edit`, `/save` 등 DB 접근 경로에서는 연결 실패가 발생한다.  
> `/info`는 DB 없이도 응답 가능하므로 Kubernetes startup/readiness/liveness probe로 사용한다.  
> `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`는 IAM Role 또는 Kubernetes Secret 기반 자격증명 사용 시 직접 환경변수로 두지 않을 수 있다.

## 외부 의존성

| 항목 | 설명 |
|---|---|
| MariaDB / MySQL | 직원 정보 저장 (`flaskapp.employee` 테이블) |
| AWS S3 | 직원 사진 이미지 저장 및 presigned URL 생성 |
| AWS 자격증명 | boto3가 S3 접근 시 필요 (IAM Role 또는 환경변수) |
| EC2 Instance Metadata | 인스턴스 ID, 가용영역 조회 (없으면 fake 값으로 fallback) |

## 앱이 하는 일

- 직원 정보(이름, 직책, 위치, 뱃지) 조회/추가/수정/삭제
- 직원 사진을 AWS S3에 업로드하고 presigned URL로 표시
- `/info` 페이지에서 EC2 인스턴스 ID, 가용영역, 버전 문자열 표시
- `/info/stress_cpu/<seconds>` 경로에서 CPU 부하 테스트 실행

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
export PHOTOS_BUCKET=flaskapp-proddata-kosa-project-team3-snow-lai9z
export DATABASE_HOST=172.16.43.160
export DATABASE_USER=flaskapp
export DATABASE_PASSWORD=<DB 비밀번호>
export DATABASE_DB_NAME=flaskapp
export AWS_DEFAULT_REGION=ap-northeast-2

# 3. DB 테이블 생성 (최초 1회)
mysql -h $DATABASE_HOST -u $DATABASE_USER -p$DATABASE_PASSWORD < database_create_tables.sql

# 4. 앱 실행
FLASK_APP=application.py flask run --host=0.0.0.0 --port=80
```

## 확인 필요 항목

- [x] AWS S3 버킷 이름 확정: `flaskapp-proddata-kosa-project-team3-snow-lai9z`
- [x] MariaDB 서버 IP 확정: 172.16.43.160
- [x] DB 계정명 확정: flaskapp
- [x] 운영 Kubernetes에서는 ConfigMap/Secret으로 환경변수 주입
- [ ] AWS 자격증명 방식 최종 문서화 (S3 접근 권한 주입 방식)
- [ ] `FLASK_SECRET` 환경변수 처리 방식 결정 및 코드 개선

## 후속 문서

| 후속 작업 | 연결 이슈 |
|---|---|
| Dockerfile 작성 및 이미지 빌드 가이드 | [flaskapp-dockerfile-guide.md](./flaskapp-dockerfile-guide.md) |
| 컨테이너 실행 가이드 | [flaskapp-container-run-guide.md](./flaskapp-container-run-guide.md) |
| Kubernetes 운영 정책 | `Docs/operation/k8s/flaskapp-kubernetes-policy-summary.md` |

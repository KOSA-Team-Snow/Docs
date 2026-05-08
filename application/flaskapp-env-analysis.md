# FlaskApp 환경변수 및 실행 조건 분석

## 문서 목적

FlaskApp 소스코드를 분석하여 컨테이너 실행에 필요한 환경변수 목록과 외부 의존성을 정리한다.

## 담당

| 구분 | 담당 |
|---|---|
| 정(Lead) | 팀원 C / @ireneminhee |

## 필수 환경변수

| 변수명 | 설명 | 예시값 |
|---|---|---|
| PHOTOS_BUCKET | AWS S3 버킷 이름 (이미지 저장용) | my-flaskapp-bucket |
| DATABASE_HOST | MariaDB 서버 주소 | 172.16.0.160 |
| DATABASE_USER | DB 접속 계정 | kosa |
| DATABASE_PASSWORD | DB 접속 비밀번호 | kosa1004 |
| DATABASE_DB_NAME | 사용할 DB 이름 | employees |

## 외부 의존성

| 항목 | 설명 |
|---|---|
| MariaDB / MySQL | 직원 정보 저장 (employee 테이블) |
| AWS S3 | 직원 사진 이미지 저장 |
| AWS 자격증명 | boto3가 S3 접근 시 필요 (IAM 또는 환경변수) |

## 앱이 하는 일

- 직원 정보(이름, 직책, 위치, 뱃지) 조회/추가/수정/삭제
- 직원 사진을 AWS S3에 업로드하고 presigned URL로 표시

## 필요한 패키지 (requirements.txt)

| 패키지 | 역할 |
|---|---|
| Flask, Flask-WTF | 웹 프레임워크 |
| mysql_connector_python | MariaDB 연결 |
| boto3 | AWS S3 연결 |
| pillow | 이미지 리사이즈 |
| requests | EC2 메타데이터 조회 |

## 확인 필요 항목

- [ ] AWS S3 버킷 이름 팀 내 확정
- [ ] MariaDB 서버 IP 확정 (현재 예상: 172.16.0.160)
- [ ] DB 계정명 및 비밀번호 팀원 D와 협의
- [ ] AWS 자격증명 방식 확정 (IAM Role vs 환경변수)

## 후속 문서

| 후속 작업 | 연결 이슈 |
|---|---|
| Dockerfile 작성 및 이미지 빌드 가이드 | Docs #13 |
| Kubernetes manifest 작성 가이드 | Docs #14 |

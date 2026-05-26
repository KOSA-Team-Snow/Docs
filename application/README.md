# Application 문서 인덱스

이 디렉터리는 FlaskApp 애플리케이션 자체를 이해하기 위한 문서 모음이다. Kubernetes, ArgoCD, AWS DR 전체 구조가 아니라 FlaskApp의 실행 조건, 컨테이너 이미지, 로컬 실행 방법을 다룬다.

## 현재 기준 요약

| 항목 | 현재 값 |
| --- | --- |
| 애플리케이션 | Flask 기반 Employee Directory |
| 주요 라우트 | `/`, `/info`, `/add`, `/edit/<id>`, `/employee/<id>`, `/delete/<id>` |
| 헬스/프로브 경로 | `/info` |
| 운영 DB | MariaDB VM `172.16.43.160` |
| 운영 DB 이름 | `flaskapp` |
| 이미지 저장소 | AWS S3 bucket `flaskapp-proddata-kosa-project-team3-snow-lai9z` |
| 컨테이너 이미지 | `080252689380.dkr.ecr.ap-northeast-2.amazonaws.com/flaskapp:18b68fe` |
| 운영 배포 | ArgoCD + Helm chart `infra/helm/flaskapp` |
| 운영 namespace | `flaskapp-prod` |

## 문서 읽는 순서

1. [flaskapp-env-analysis.md](./flaskapp-env-analysis.md)  
   FlaskApp이 어떤 환경변수와 외부 의존성을 필요로 하는지 정리한다.

2. [flaskapp-dockerfile-guide.md](./flaskapp-dockerfile-guide.md)  
   Dockerfile 구조, 이미지 빌드, ECR push 흐름을 정리한다.

3. [flaskapp-container-run-guide.md](./flaskapp-container-run-guide.md)  
   로컬 Docker 컨테이너 실행과 최소 동작 확인 방법을 정리한다.

## 최신 운영 기준

FlaskApp은 현재 Kubernetes에서 다음 방식으로 운영된다.

```text
HAProxy/Keepalived VIP 172.16.42.99
  -> NGINX Ingress
  -> flaskapp-service ClusterIP
  -> FlaskApp Pod 2개
  -> MariaDB VM 172.16.43.160
```

현재 운영 검증 기준:

- `http://172.16.42.99/info` + Host `flaskapp.team.snow.internal`: HTTP 200
- `http://172.16.42.99/` + Host `flaskapp.team.snow.internal`: HTTP 200
- FlaskApp Deployment: `2/2`
- HPA: min 2, max 4
- PDB: minAvailable 1
- NetworkPolicy: ingress-nginx, DNS, DB, HTTPS egress 허용

## 주의할 점

- `/info`는 DB 없이도 응답 가능한 경로라 probe와 기본 생존 확인에 적합하다.
- `/`는 MariaDB 연결이 필요하므로 DB 장애 시 500이 날 수 있다.
- `PHOTOS_BUCKET`은 코드상 필수 환경변수다. 없으면 앱 시작 시 실패한다.
- `DYNAMO_MODE`는 코드에 남아 있지만 현재 프로젝트에서는 사용하지 않는다.
- `FLASK_SECRET`은 현재 코드에 하드코딩되어 있어 운영 개선 포인트다.

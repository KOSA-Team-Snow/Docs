# Flaskapp DB 연결 및 앱 테스트 작업 기록

작성일: 2026-05-19

## 목적

Kubernetes에 배포된 `flaskapp` Pod가 MariaDB의 `flaskapp` 데이터베이스에 정상 연결되도록 설정하고, 웹 앱에서 직원 데이터를 추가해 DB insert까지 확인한다.

## 현재 구조

Flask 앱은 DB 접속 정보를 코드에 하드코딩하지 않고 환경변수로 읽는다.

- Flaskapp repo: `GitHub/KOSA/Flaskapp`
- Infra repo: `GitHub/KOSA/infra`
- Docs repo: `GitHub/KOSA/Docs`

Flaskapp 코드 흐름:

```text
Kubernetes ConfigMap / Secret
        ↓
Pod 환경변수
        ↓
config.py os.environ
        ↓
database.py mysql.connector.connect()
        ↓
MariaDB
```

관련 Flaskapp 파일:

- `config.py`: `DATABASE_HOST`, `DATABASE_USER`, `DATABASE_PASSWORD`, `DATABASE_DB_NAME`을 `os.environ`에서 읽음
- `database.py`: 위 환경변수 값으로 MariaDB 연결
- `database_create_tables.sql`: `employee` 테이블 생성 SQL 참고용

## 확인된 MariaDB 상태

MariaDB에서 확인한 내용:

```sql
SELECT User, Host FROM mysql.user;
```

확인된 앱 계정:

```text
User: flaskapp
Host: 172.16.43.%
```

데이터베이스 목록에는 `employees`, `flaskapp`이 모두 있었다.

```sql
SHOW DATABASES;
```

따라서 앱이 사용할 DB 설정은 다음과 같이 정리했다.

```text
DATABASE_HOST=172.16.43.160
DATABASE_USER=flaskapp
DATABASE_DB_NAME=flaskapp
DATABASE_PASSWORD=<K8s Secret에 저장>
```

주의: `flaskapp` 계정은 `172.16.43.%` 대역에서만 접속 가능하다. Bastion VM이 `172.16.44.100`인 경우, Bastion에서 `flaskapp` 계정으로 MariaDB에 직접 접속하면 `Access denied for 'flaskapp'@'172.16.44.100'`가 날 수 있다. 앱 Pod에서의 접속 성공 여부는 Pod 안에서 확인해야 한다.

## Infra 설정 위치

ArgoCD Application은 Helm 차트를 바라본다.

파일:

```text
infra/argocd/apps/flaskapp.yaml
```

핵심 설정:

```yaml
source:
  path: helm/flaskapp
```

따라서 실제 GitOps 기준 설정 파일은 다음이다.

```text
infra/helm/flaskapp/values.yaml
```

수정한 DB 설정:

```yaml
config:
  DATABASE_HOST: "172.16.43.160"
  DATABASE_USER: "flaskapp"
  DATABASE_DB_NAME: "flaskapp"
  AWS_DEFAULT_REGION: "ap-northeast-2"
  PHOTOS_BUCKET: "flaskapp-proddata-kosa-project-team3-snow-lai9z"
```

Helm Deployment 템플릿은 ConfigMap과 Secret을 Pod 환경변수로 주입한다.

파일:

```text
infra/helm/flaskapp/templates/deployment.yaml
```

핵심 설정:

```yaml
envFrom:
  - configMapRef:
      name: {{ .Release.Name }}-config
  - secretRef:
      name: {{ .Release.Name }}-secret
```

Release 이름이 `flaskapp`이면 실제 참조 이름은 다음과 같다.

```text
ConfigMap: flaskapp-config
Secret: flaskapp-secret
```

참고: `Release`는 Helm 배포 이름이다. 예를 들어 `helm install flaskapp ...`에서 `flaskapp`이 Release 이름이다.

## ConfigMap / Secret 역할

ConfigMap에는 공개 설정이 들어간다.

```text
DATABASE_HOST
DATABASE_USER
DATABASE_DB_NAME
AWS_DEFAULT_REGION
PHOTOS_BUCKET
```

Secret에는 민감 정보가 들어간다.

```text
DATABASE_PASSWORD
```

ConfigMap 확인:

```bash
kubectl describe configmap flaskapp-config -n flaskapp-prod
```

확인된 값:

```text
DATABASE_DB_NAME=flaskapp
DATABASE_HOST=172.16.43.160
DATABASE_USER=flaskapp
AWS_DEFAULT_REGION=ap-northeast-2
PHOTOS_BUCKET=flaskapp-proddata-kosa-project-team3-snow-lai9z
```

Secret 존재 및 키 확인:

```bash
kubectl get secret flaskapp-secret -n flaskapp-prod
kubectl describe secret flaskapp-secret -n flaskapp-prod
```

비밀번호 값을 직접 확인해야 할 때:

```bash
kubectl get secret flaskapp-secret -n flaskapp-prod -o jsonpath='{.data.DATABASE_PASSWORD}' | base64 -d; echo
```

주의: 위 명령은 비밀번호를 화면에 그대로 출력한다. 공유 채팅이나 문서에는 남기지 않는다.

Secret이 없을 때 생성 예시:

```bash
kubectl create secret generic flaskapp-secret \
  -n flaskapp-prod \
  --from-literal=DATABASE_PASSWORD='<실제 비밀번호>'
```

Secret을 바꾼 뒤에는 Pod를 재시작해야 한다.

```bash
kubectl rollout restart deployment/flaskapp -n flaskapp-prod
```

## 배포 반영

`values.yaml` 수정 후 ArgoCD sync 또는 Git push 이후 자동 sync가 필요하다. ConfigMap/Secret 값은 이미 떠 있는 Pod에는 자동으로 반영되지 않으므로 Pod 재시작을 수행했다.

```bash
kubectl rollout restart deployment/flaskapp -n flaskapp-prod
kubectl rollout status deployment/flaskapp -n flaskapp-prod
```

확인 결과:

```text
deployment.apps/flaskapp restarted
deployment "flaskapp" successfully rolled out
```

Pod에 환경변수가 들어갔는지 확인:

```bash
kubectl exec -it -n flaskapp-prod deploy/flaskapp -- env | grep DATABASE
```

확인된 값:

```text
DATABASE_DB_NAME=flaskapp
DATABASE_HOST=172.16.43.160
DATABASE_USER=flaskapp
DATABASE_PASSWORD=<Secret 값>
```

주의: `env | grep DATABASE`는 비밀번호도 출력할 수 있으므로 화면 공유나 문서화 시 주의한다.

## DB 접속 확인

Pod 안에서 DB 접속 및 테이블 목록을 확인했다.

```bash
kubectl exec -it -n flaskapp-prod deploy/flaskapp -- python3 -c 'import os, mysql.connector; c=mysql.connector.connect(host=os.environ["DATABASE_HOST"], user=os.environ["DATABASE_USER"], password=os.environ["DATABASE_PASSWORD"], database=os.environ["DATABASE_DB_NAME"], use_pure=True); cur=c.cursor(); cur.execute("SHOW TABLES"); print(cur.fetchall()); cur.close(); c.close()'
```

처음 결과:

```text
[]
```

의미:

- DB 접속은 성공했다.
- `flaskapp` 데이터베이스 안에 테이블이 없었다.

접속 실패라면 `Access denied`, `Can't connect`, `Unknown database` 같은 에러가 나야 한다.

## employee 테이블 생성

Flaskapp 코드는 `employee` 테이블을 사용한다.

`database.py`에서 사용하는 테이블:

```sql
FROM employee
```

MariaDB에서 `flaskapp` DB 선택 후 테이블을 생성했다.

```sql
USE flaskapp;

CREATE TABLE IF NOT EXISTS employee (
  id int not null auto_increment primary key,
  object_key nvarchar(80),
  full_name nvarchar(200) not null,
  location nvarchar(200) not null,
  job_title nvarchar(200) not null,
  badges nvarchar(200) not null,
  created_datetime DATETIME DEFAULT now()
);

SHOW TABLES;
```

확인 결과:

```text
Tables_in_flaskapp
employee
```

이로써 앱이 조회/insert할 기본 테이블이 준비되었다.

## 앱 Service 확인

Ingress 접속 전, Service로 앱이 정상 동작하는지 확인했다.

```bash
kubectl port-forward -n flaskapp-prod svc/flaskapp-service 8080:80
```

다른 Bastion 터미널에서 확인:

```bash
curl http://127.0.0.1:8080
```

확인 결과 `Employee Directory` HTML이 응답했다.

초기 화면에는 데이터가 없어서 다음 문구가 보였다.

```text
Empty Directory
```

이는 에러가 아니라 `employee` 테이블에 데이터가 아직 없다는 뜻이다.

## 로컬 브라우저로 접속하기: SSH 터널 포함

목표: 내 로컬 PC 브라우저에서 Kubernetes 내부 Service를 열고 앱에서 직접 insert한다.

필요한 터미널은 2개다.

### 1. Bastion 터미널

Bastion에서 Service port-forward를 켜둔다.

```bash
kubectl port-forward -n flaskapp-prod svc/flaskapp-service 8080:80
```

이 명령은 계속 실행 중이어야 한다.

### 2. 로컬 PC 터미널

내 로컬 PC에서 Bastion으로 SSH 터널을 연다.

```bash
ssh -L 8080:127.0.0.1:8080 kosa@<BASTION_IP>
```

평소 Bastion 접속에 SSH key가 필요하면 기존 접속 명령에 `-L 8080:127.0.0.1:8080`만 추가한다.

예시:

```bash
ssh -i <KEY_PATH> -L 8080:127.0.0.1:8080 kosa@<BASTION_IP>
```

### 3. 로컬 브라우저

브라우저에서 접속한다.

```text
http://127.0.0.1:8080
```

또는:

```text
http://localhost:8080
```

Employee Directory 화면에서 `Add` 버튼으로 데이터를 추가한다.

## 앱에서 DB insert 확인

브라우저에서 데이터를 추가한 뒤 화면에 다음이 표시되었다.

```text
Saved!
```

그리고 목록에 추가한 직원이 표시되었다.

이 의미:

```text
브라우저 → Flaskapp Service → Flask Pod → MariaDB insert → MariaDB select → 화면 표시
```

까지 성공했다.

Pod에서 DB 데이터를 직접 확인하는 명령:

```bash
kubectl exec -it -n flaskapp-prod deploy/flaskapp -- python3 -c 'import database; print(database.list_employees())'
```

SQL로 직접 확인하는 명령:

```bash
kubectl exec -it -n flaskapp-prod deploy/flaskapp -- python3 -c 'import os, mysql.connector; c=mysql.connector.connect(host=os.environ["DATABASE_HOST"], user=os.environ["DATABASE_USER"], password=os.environ["DATABASE_PASSWORD"], database=os.environ["DATABASE_DB_NAME"], use_pure=True); cur=c.cursor(); cur.execute("SELECT id, full_name, location, job_title, badges FROM employee"); print(cur.fetchall()); cur.close(); c.close()'
```

## Ingress 이슈

Ingress 리소스는 존재한다.

```bash
kubectl get ingress -n flaskapp-prod
```

확인된 Ingress:

```text
NAME               CLASS   HOSTS                   ADDRESS         PORTS
flaskapp-ingress   nginx   flaskapp.onprem.local   172.16.43.114   80
```

그러나 Bastion에서 도메인 조회 시 엉뚱한 외부 IP로 해석되었다.

```bash
getent hosts flaskapp.onprem.local
```

결과:

```text
182.162.73.77   gaja79.com flaskapp.onprem.local.co.kr
```

또한 Ingress IP 직접 접속도 실패했다.

```bash
curl -H 'Host: flaskapp.onprem.local' http://172.16.43.114
```

결과:

```text
curl: (7) Failed to connect to 172.16.43.114 port 80
```

따라서 현재 결론:

- 앱, Service, Pod, DB 연결은 정상이다.
- Ingress 또는 Ingress Controller/네트워크 접근은 별도 점검이 필요하다.
- 당장은 `kubectl port-forward`와 SSH 터널로 앱을 로컬 브라우저에서 사용할 수 있다.

Ingress 점검 명령:

```bash
kubectl get pods -A | grep -i ingress
kubectl get svc -A | grep -i ingress
kubectl describe ingress -n flaskapp-prod
```

DNS/hosts를 임시로 맞출 때는 `/etc/hosts`에 다음 줄을 넣을 수 있다.

```text
172.16.43.114 flaskapp.onprem.local
```

주의: 위 줄은 명령어가 아니라 `/etc/hosts` 파일에 넣는 내용이다.

## 최종 상태

완료된 것:

- MariaDB에 `flaskapp` DB 존재 확인
- MariaDB에 `flaskapp` 계정 존재 확인
- K8s ConfigMap DB 설정을 `flaskapp` 기준으로 반영
- K8s Secret의 `DATABASE_PASSWORD`가 Pod에 주입되는 것 확인
- Deployment rollout restart 완료
- Pod 환경변수 확인 완료
- Pod에서 MariaDB 접속 확인 완료
- `flaskapp.employee` 테이블 생성 완료
- Service port-forward로 Flask 앱 접속 확인 완료
- SSH 터널로 로컬 브라우저 접속 가능
- 앱에서 데이터 추가 후 `Saved!` 및 목록 표시 확인

남은 것:

- Ingress `172.16.43.114:80` 접근 실패 원인 확인
- `flaskapp.onprem.local` DNS 또는 hosts 설정 정리
- 노출된 DB 비밀번호가 있다면 Secret 비밀번호 교체 검토

## 자주 쓰는 명령 모음

Rollout 재시작:

```bash
kubectl rollout restart deployment/flaskapp -n flaskapp-prod
kubectl rollout status deployment/flaskapp -n flaskapp-prod
```

Pod 환경변수 확인:

```bash
kubectl exec -it -n flaskapp-prod deploy/flaskapp -- env | grep DATABASE
```

ConfigMap 확인:

```bash
kubectl describe configmap flaskapp-config -n flaskapp-prod
```

Secret 확인:

```bash
kubectl describe secret flaskapp-secret -n flaskapp-prod
```

테이블 확인:

```bash
kubectl exec -it -n flaskapp-prod deploy/flaskapp -- python3 -c 'import os, mysql.connector; c=mysql.connector.connect(host=os.environ["DATABASE_HOST"], user=os.environ["DATABASE_USER"], password=os.environ["DATABASE_PASSWORD"], database=os.environ["DATABASE_DB_NAME"], use_pure=True); cur=c.cursor(); cur.execute("SHOW TABLES"); print(cur.fetchall()); cur.close(); c.close()'
```

앱 DB 조회 확인:

```bash
kubectl exec -it -n flaskapp-prod deploy/flaskapp -- python3 -c 'import database; print(database.list_employees())'
```

Bastion port-forward:

```bash
kubectl port-forward -n flaskapp-prod svc/flaskapp-service 8080:80
```

로컬 SSH 터널:

```bash
ssh -L 8080:127.0.0.1:8080 kosa@<BASTION_IP>
```

로컬 브라우저:

```text
http://127.0.0.1:8080
```

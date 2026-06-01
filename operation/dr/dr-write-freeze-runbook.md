# DR Write Freeze Runbook

작성일: 2026-05-26

## 목적

DR 전환 선언 직후 On-premise write path를 차단하고, AWS RDS를 서비스 DB로 사용하기 전 DMS Task를 안전하게 정지하는 절차를 정리한다.

이 절차의 핵심 목적은 **split-write 방지**이다.



```text
On-prem 일부가 살아 있음
-> On-prem 앱/DB에 write 발생
-> DMS는 정지됨
-> AWS RDS에는 해당 write가 반영되지 않음
-> On-prem DB와 AWS RDS 데이터가 갈라짐
```

따라서 DR 전환 시에는 AWS RDS를 서비스 DB로 사용하기 전에 On-prem write 경로를 먼저 막고, DMS 지연을 확인한 뒤 DMS Task를 정지한다.

## 구현 위치

Infra repository:

```text
infra/scripts/dr-declare-write-freeze.sh
infra/aws/terraform/envs/dr/outputs.tf
```

Terraform output에는 다음 operator command가 추가된다.

```text
operator_runbook.step_3_5_write_freeze
```

## 수행 순서

스크립트는 다음 순서로 동작한다.

```text
1. On-prem FlaskApp Deployment replicas=0
2. 필요 시 HAProxy 정지
3. 가능한 경우 On-prem MariaDB read_only=ON
4. DMS CDC lag 확인
5. DMS Task 정지
6. DMS Task stopped 상태 확인
```

## 실행 전 조건

실행 위치는 bastion 또는 운영자가 kubectl, terraform, aws CLI를 사용할 수 있는 환경이다.

필요 도구:

```text
kubectl
terraform
aws
ssh
```

필요 권한:

```text
Kubernetes flaskapp-prod Deployment scale 권한
MariaDB VM SSH 및 sudo mysql 권한
AWS DMS describe/stop 권한
CloudWatch DMS metric 조회 권한
Terraform state/output 조회 권한
```

기본 경로:

```bash
cd infra
./scripts/dr-declare-write-freeze.sh
```

## 기본 실행

```bash
cd /path/to/infra
./scripts/dr-declare-write-freeze.sh
```

기본값:

| 항목 | 기본값 |
| --- | --- |
| Kubernetes namespace | `flaskapp-prod` |
| Deployment | `flaskapp` |
| freeze replica | `0` |
| AWS region | `ap-northeast-2` |
| 최대 허용 DMS lag | `300` seconds |
| MariaDB SSH target | `kosa@172.16.43.160` |
| MariaDB read_only | enabled |
| HAProxy stop | disabled |

## HAProxy까지 정지하는 경우

기본 실행은 FlaskApp Deployment를 scale 0으로 만들어 앱 write path를 차단한다.

사용자 진입점까지 함께 막아야 하는 리허설 또는 실제 DR 상황에서는 HAProxy 정지도 옵션으로 수행한다.

```bash
STOP_HAPROXY=true \
LB_SSH_TARGETS="kosa@172.16.42.100,kosa@172.16.42.101" \
./scripts/dr-declare-write-freeze.sh
```

주의:

```text
HAProxy를 정지하면 FlaskApp뿐 아니라 같은 Ingress 경로를 사용하는 Grafana 등 다른 서비스 접근도 영향받을 수 있다.
발표용 장애 주입에서는 FlaskApp scale 0만으로 충분할 수 있다.
```

## MariaDB read_only를 건너뛰는 경우

On-prem DB에 접근할 수 없는 전체 장애 상황이거나, read_only 전환 권한이 없는 경우에는 다음처럼 건너뛴다.

```bash
MYSQL_READ_ONLY=false ./scripts/dr-declare-write-freeze.sh
```

이 경우에도 앱 write path 차단과 DMS Task 정지는 수행된다.

## DMS lag 기준

스크립트는 CloudWatch의 `CDCLatencyTarget`을 조회한다.

기본적으로 최근 5분 안에 metric datapoint가 없으면 DMS Task 정지를 거부한다.

```text
REQUIRE_DMS_LAG_DATAPOINT=true
```

리허설 중 metric 수집 지연이 있고 운영자가 수동으로 안전하다고 판단한 경우에만 override한다.

```bash
REQUIRE_DMS_LAG_DATAPOINT=false ./scripts/dr-declare-write-freeze.sh
```

허용 lag 기준을 바꾸려면:

```bash
MAX_DMS_LAG_SECONDS=60 ./scripts/dr-declare-write-freeze.sh
```

## 성공 기준

성공 시 다음 상태가 되어야 한다.

```text
FlaskApp Deployment replicas = 0
On-prem MariaDB read_only = ON (가능한 경우)
DMS Task status = stopped
```

확인 명령:

```bash
kubectl get deploy,pod -n flaskapp-prod
ssh kosa@172.16.43.160 "sudo mysql -Nse \"SHOW GLOBAL VARIABLES LIKE 'read_only';\""
aws dms describe-replication-tasks \
  --query 'ReplicationTasks[*].[ReplicationTaskIdentifier,Status]' \
  --output table
```

## 이후 DR 전환 단계

write freeze가 완료되면 AWS RDS를 서비스 DB로 사용할 수 있다.

후속 단계:

```text
1. AWS RDS 데이터 확인
2. terraform apply -var="dr_active=true"
3. EKS node ready 확인
4. FlaskApp DR values 배포
5. ALB DNS 확인
6. Bind9 DNS를 AWS ALB CNAME으로 전환
7. 같은 도메인으로 FlaskApp 접속 검증
```

## 원복 기준

주의:

```text
AWS DR 환경에서 신규 write가 발생한 경우, 바로 On-prem으로 되돌리면 데이터가 갈라질 수 있다.
Failback은 별도 절차로 수행해야 한다.
```

단순 리허설이며 AWS RDS에 신규 write가 없었다면 다음 순서로 복구할 수 있다.

FlaskApp 복구:

```bash
kubectl scale deployment/flaskapp -n flaskapp-prod --replicas=2
kubectl rollout status deployment/flaskapp -n flaskapp-prod
```

MariaDB read_only 해제:

```bash
ssh kosa@172.16.43.160 "sudo mysql -e \"SET GLOBAL read_only = OFF;\""
```

DMS 재개:

```bash
aws dms start-replication-task \
  --replication-task-arn <DMS_TASK_ARN> \
  --start-replication-task-type resume-processing
```

## 발표용 설명

```text
DR 전환 시 가장 위험한 상황은 On-prem과 AWS가 동시에 write를 받는 split-write이다.
이를 방지하기 위해 DR 선언 직후 On-prem 앱 write path를 차단하고,
가능하면 MariaDB를 read_only로 전환한 뒤,
DMS lag를 확인하고 DMS Task를 정지한다.
그 이후 AWS RDS를 서비스 DB로 사용한다.
```

## GitHub Issue 본문 초안

제목:

```text
[Task] DR 선언 시 On-prem write freeze 및 DMS 안전 정지 스크립트 추가
```

본문:

```md
## 목적

DR 전환 선언 직후 On-prem write path를 차단하고, AWS RDS를 서비스 DB로 사용하기 전 DMS Task를 안전하게 정지하는 스크립트를 추가한다.

이 작업은 split-write 방지를 위한 DR Runbook 자동화이다.

## 배경

DR 전환 중 On-prem 일부가 살아 있는 상태에서 On-prem DB에 write가 발생하고, 동시에 AWS RDS가 서비스 DB로 사용되면 On-prem DB와 AWS RDS 데이터가 갈라질 수 있다.

따라서 DMS Task를 정지하기 전에 On-prem 앱 write path를 먼저 차단하고, 가능한 경우 MariaDB를 read_only로 전환해야 한다.

## 변경 내용

- `infra/scripts/dr-declare-write-freeze.sh` 추가
- `infra/aws/terraform/envs/dr/outputs.tf`의 `operator_runbook`에 `step_3_5_write_freeze` 추가

## 스크립트 동작

1. `flaskapp-prod` namespace의 `flaskapp` Deployment를 replicas=0으로 scale
2. 옵션으로 HAProxy 정지
3. 옵션으로 On-prem MariaDB `read_only=ON`
4. CloudWatch `CDCLatencyTarget`으로 DMS lag 확인
5. 기준 이하이면 DMS Task 정지
6. DMS Task가 `stopped` 상태가 될 때까지 확인

## 기본 실행

```bash
cd infra
./scripts/dr-declare-write-freeze.sh
```

## 옵션

```bash
# HAProxy까지 정지
STOP_HAPROXY=true \
LB_SSH_TARGETS="kosa@172.16.42.100,kosa@172.16.42.101" \
./scripts/dr-declare-write-freeze.sh

# MariaDB read_only 건너뛰기
MYSQL_READ_ONLY=false ./scripts/dr-declare-write-freeze.sh

# 허용 DMS lag 기준 변경
MAX_DMS_LAG_SECONDS=60 ./scripts/dr-declare-write-freeze.sh
```


## 주의사항

- 실제 DR 실행 전 `kubectl`, `terraform`, `aws`, `ssh` 권한이 필요하다.
- AWS DR 환경에서 신규 write가 발생한 뒤에는 단순히 On-prem으로 원복하면 안 된다.
- Failback은 별도 절차로 설계해야 한다.
```

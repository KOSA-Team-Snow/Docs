# DR Write Freeze 상황별 실행 옵션

작성일: 2026-06-01

## 목적

이 문서는 `infra/scripts/dr-declare-write-freeze.sh` 실행 시 DR 상황에 따라 운영자가 선택해야 하는 환경 변수 옵션을 정리한다.

현재 스크립트는 DR 상황을 자동으로 판별하지 않는다. 따라서 운영자는 MariaDB 접근 가능 여부, DMS lag metric 수집 상태, HAProxy 차단 필요 여부를 판단한 뒤 실행 옵션을 선택해야 한다.

기본 실행 위치:

```bash
cd /path/to/infra
```

## 옵션 선택 기준

```text
MariaDB 접근 가능
-> MYSQL_READ_ONLY 기본값 true 유지

MariaDB 접근 불가
-> MYSQL_READ_ONLY=false

DMS lag datapoint 있음
-> REQUIRE_DMS_LAG_DATAPOINT 기본값 true 유지

DMS lag datapoint 없음 + 운영자가 수동으로 안전하다고 판단
-> REQUIRE_DMS_LAG_DATAPOINT=false

HAProxy까지 막아야 함
-> STOP_HAPROXY=true + LB_SSH_TARGETS 지정
```

## 1. On-prem 전체가 정상이고 DR 전환만 선언한 경우

MariaDB 접근 가능, DMS metric 정상, HAProxy는 유지하는 기본 케이스이다.

```bash
./scripts/dr-declare-write-freeze.sh
```

수행 내용:

```text
FlaskApp Deployment replicas=0
MariaDB read_only=ON
DMS lag 확인
DMS Task stop
```

## 2. 사용자 진입점까지 완전히 차단해야 하는 경우

FlaskApp scale 0뿐 아니라 HAProxy까지 정지한다.

```bash
STOP_HAPROXY=true \
LB_SSH_TARGETS="kosa@172.16.42.100,kosa@172.16.42.101" \
./scripts/dr-declare-write-freeze.sh
```

수행 내용:

```text
FlaskApp Deployment replicas=0
HAProxy stop
MariaDB read_only=ON
DMS lag 확인
DMS Task stop
```

주의:

```text
HAProxy를 정지하면 FlaskApp뿐 아니라 같은 Ingress 경로를 사용하는 다른 서비스 접근도 영향받을 수 있다.
```

## 3. MariaDB DB 프로세스가 다운된 경우

MariaDB VM에는 접속 가능하더라도 `sudo mysql` 명령이 실패할 수 있으므로 read_only 단계를 건너뛴다.

```bash
MYSQL_READ_ONLY=false \
./scripts/dr-declare-write-freeze.sh
```

DMS lag metric이 없어서 실패하면, 운영자가 수동으로 안전하다고 판단한 경우에만 다음처럼 재실행한다.

```bash
MYSQL_READ_ONLY=false \
REQUIRE_DMS_LAG_DATAPOINT=false \
./scripts/dr-declare-write-freeze.sh
```

## 4. MariaDB VM 자체가 다운됐거나 SSH가 불가능한 경우

MariaDB SSH 단계에서 스크립트가 중단되지 않도록 read_only 단계를 건너뛴다.

```bash
MYSQL_READ_ONLY=false \
./scripts/dr-declare-write-freeze.sh
```

DMS lag metric이 없어서 실패하면, 운영자가 수동으로 안전하다고 판단한 경우에만 다음처럼 재실행한다.

```bash
MYSQL_READ_ONLY=false \
REQUIRE_DMS_LAG_DATAPOINT=false \
./scripts/dr-declare-write-freeze.sh
```

## 5. MariaDB 장애와 사용자 진입점 차단이 모두 필요한 경우

MariaDB read_only는 건너뛰고, HAProxy는 정지한다.

```bash
MYSQL_READ_ONLY=false \
STOP_HAPROXY=true \
LB_SSH_TARGETS="kosa@172.16.42.100,kosa@172.16.42.101" \
./scripts/dr-declare-write-freeze.sh
```

DMS lag metric이 없어서 실패하면, 운영자가 수동으로 안전하다고 판단한 경우에만 다음처럼 재실행한다.

```bash
MYSQL_READ_ONLY=false \
REQUIRE_DMS_LAG_DATAPOINT=false \
STOP_HAPROXY=true \
LB_SSH_TARGETS="kosa@172.16.42.100,kosa@172.16.42.101" \
./scripts/dr-declare-write-freeze.sh
```

## 6. DMS lag metric 수집이 지연되거나 datapoint가 없는 리허설 상황

CloudWatch `CDCLatencyTarget` datapoint가 없으면 기본적으로 DMS Task 정지를 거부한다.

운영자가 수동으로 안전하다고 판단한 경우에만 다음 옵션을 사용한다.

```bash
REQUIRE_DMS_LAG_DATAPOINT=false \
./scripts/dr-declare-write-freeze.sh
```

MariaDB도 접근 불가라면 다음처럼 실행한다.

```bash
MYSQL_READ_ONLY=false \
REQUIRE_DMS_LAG_DATAPOINT=false \
./scripts/dr-declare-write-freeze.sh
```

## 7. DMS lag 허용 기준을 더 엄격하게 하는 경우

예를 들어 DMS lag가 60초 이하일 때만 DMS Task를 정지하려면 다음처럼 실행한다.

```bash
MAX_DMS_LAG_SECONDS=60 \
./scripts/dr-declare-write-freeze.sh
```

MariaDB 장애 상황이면 read_only 단계를 함께 건너뛴다.

```bash
MYSQL_READ_ONLY=false \
MAX_DMS_LAG_SECONDS=60 \
./scripts/dr-declare-write-freeze.sh
```

## 8. namespace나 Deployment 이름이 다른 경우

기본값은 `flaskapp-prod` namespace의 `flaskapp` Deployment이다.

다른 환경에서 실행해야 하면 다음처럼 명시한다.

```bash
KUBE_NAMESPACE=flaskapp-prod \
KUBE_DEPLOYMENT=flaskapp \
FREEZE_REPLICAS=0 \
./scripts/dr-declare-write-freeze.sh
```

## 9. MariaDB SSH 대상이 다른 경우

기본값은 `kosa@172.16.43.160`이다.

다른 MariaDB VM을 대상으로 해야 하면 다음처럼 지정한다.

```bash
MYSQL_SSH_TARGET="kosa@172.16.43.160" \
./scripts/dr-declare-write-freeze.sh
```

## 10. Terraform DR 디렉터리를 명시해야 하는 경우

스크립트를 `infra` 밖에서 실행하거나 repo root 탐지가 애매할 때 사용한다.

```bash
TF_DIR="/path/to/infra/aws/terraform/envs/dr" \
./scripts/dr-declare-write-freeze.sh
```

## 운영 시 주의사항

현재 스크립트는 다음 기본값을 가진다.

```text
MYSQL_READ_ONLY=true
REQUIRE_DMS_LAG_DATAPOINT=true
STOP_HAPROXY=false
```

따라서 MariaDB가 다운된 상황에서 기본 실행을 하면 read_only 설정 단계에서 스크립트가 중단될 수 있다.

MariaDB 장애 상황에서는 최소한 다음 옵션을 먼저 고려한다.

```bash
MYSQL_READ_ONLY=false \
./scripts/dr-declare-write-freeze.sh
```

DMS lag datapoint까지 없는 경우에는 운영자가 수동으로 안전성을 확인한 뒤에만 다음 옵션을 추가한다.

```bash
REQUIRE_DMS_LAG_DATAPOINT=false
```

## 요약

```text
정상 DR 전환
-> ./scripts/dr-declare-write-freeze.sh

MariaDB 접근 불가
-> MYSQL_READ_ONLY=false ./scripts/dr-declare-write-freeze.sh

MariaDB 접근 불가 + DMS lag datapoint 없음
-> MYSQL_READ_ONLY=false REQUIRE_DMS_LAG_DATAPOINT=false ./scripts/dr-declare-write-freeze.sh

HAProxy까지 차단
-> STOP_HAPROXY=true LB_SSH_TARGETS="..." ./scripts/dr-declare-write-freeze.sh
```

# 05-1. 데이터 상세 구현 설계

> 이전 문서 `aws-dr-architecture.md`의 **5장 데이터**를 더 깊이 다룹니다.

## 📌 한 줄 요약

> **온프렘 MariaDB의 변경사항을 DMS가 binlog로 읽어 실시간으로 AWS RDS에 복사하고, 사진/파일은 양쪽에서 같은 S3 버킷을 공유합니다. 모든 데이터는 KMS로 암호화되고, 인터넷에 노출되지 않습니다. 장애 시엔 RDS가 즉시 쓰기 가능 상태이므로 Promote 절차도 단순합니다.**

## 목차

- [0. 이 문서 읽기 전에 알아둘 용어](#0-이-문서-읽기-전에-알아둘-용어)
- [1. 데이터 흐름 전체 그림](#1-데이터-흐름-전체-그림)
- [2. RDS 인스턴스 설계](#2-rds-인스턴스-설계)
- [3. RDS Parameter Group — DB의 동작 설정](#3-rds-parameter-group--db의-동작-설정)
- [4. RDS 백업·스냅샷·암호화](#4-rds-백업스냅샷암호화)
- [5. DMS Replication Instance — 복제 일꾼](#5-dms-replication-instance--복제-일꾼)
- [6. DMS Endpoints — 양쪽 DB 연결 정보](#6-dms-endpoints--양쪽-db-연결-정보)
- [7. DMS Task — 실제 복제 작업](#7-dms-task--실제-복제-작업)
- [8. S3 — `proddata` 버킷 상세](#8-s3--proddata-버킷-상세)
- [9. S3 — `tfstate` 버킷 상세](#9-s3--tfstate-버킷-상세)
- [10. Failover & Failback 명령 매트릭스](#10-failover--failback-명령-매트릭스)
- [11. Terraform 모듈 구조](#11-terraform-모듈-구조)
- [12. 검증 체크리스트](#12-검증-체크리스트)

---

## 0. 이 문서 읽기 전에 알아둘 용어

### 0.1 데이터베이스 관련

| 용어 | 한 줄 설명 |
|---|---|
| **RDS** | AWS가 관리해주는 데이터베이스 (MariaDB/MySQL 등) |
| **Multi-AZ** | RDS가 두 AZ에 동시 배치되는 모드. 한쪽 죽어도 자동 전환 |
| **Parameter Group** | DB의 동작 설정 모음 (Linux의 `/etc/my.cnf` 같은 것) |
| **binlog (binary log)** | MariaDB/MySQL이 모든 변경사항을 기록하는 로그 |
| **CDC** (Change Data Capture) | binlog를 읽어서 변경분만 다른 DB에 복사 |
| **Full Load** | DMS가 처음 한 번 전체 데이터를 복사하는 단계 |
| **GTID** | 트랜잭션마다 부여되는 전 세계 유일 ID |
| **read_only** | DB를 읽기 전용으로 만드는 설정 |
| **Snapshot** | 특정 시점의 DB 통째 백업 |
| **Point-in-Time Recovery** | 백업 기간 내 어느 시점으로든 복구 |

### 0.2 DMS 관련

| 용어 | 한 줄 설명 |
|---|---|
| **DMS** (Database Migration Service) | AWS의 DB 마이그레이션/실시간 복제 도구 |
| **Replication Instance** | DMS의 작업 일꾼 (EC2 같은 컴퓨팅 자원) |
| **Endpoint** | DMS가 연결할 DB 정보 (Source / Target) |
| **Task** | "이 Source에서 이 Target으로 복제해" 명령 |
| **Table Mapping** | "어떤 테이블을 복제할지" 규칙 |
| **CDCLatencyTarget** | 변경사항이 타깃에 적용되기까지의 지연 시간 (RPO 지표) |
| **Validation** | DMS가 행 단위로 양쪽 데이터 일치 검증 |

### 0.3 보안/기타

| 용어 | 한 줄 설명 |
|---|---|
| **KMS / CMK** | AWS의 암호화 키 관리 서비스 / Customer Managed Key |
| **Secrets Manager** | DB 비번 같은 비밀값을 안전하게 저장하고 자동 회전 |
| **Versioning** | S3에서 객체가 변경/삭제돼도 이전 버전 보존 |
| **Lifecycle** | "30일 후엔 저렴한 스토리지로, 1년 후엔 더 저렴하게" 자동화 |
| **Bucket Policy** | S3 버킷 단위 권한 규칙 |
| **Split-brain** | 양쪽이 동시에 Primary가 되어 데이터가 갈라지는 사태 |

---

## 1. 데이터 흐름 전체 그림

### 1.1 토폴로지

```mermaid
flowchart TB
    subgraph OnPrem["🏢 On-prem (172.16.43.0/24)"]
        APP1[FlaskApp Pods]
        MDB[(MariaDB Primary<br/>172.16.43.160:3306)]
        BINLOG[binlog<br/>ROW format]
    end

    subgraph AWS["☁️ AWS DR (서울 리전)"]
        subgraph DataSubnet["Private Data Subnet"]
            DMSI[DMS Replication Instance<br/>Multi-AZ]
            RDS[(RDS MariaDB<br/>Read Replica)]
            RDS_SB[(RDS Standby<br/>Multi-AZ)]
        end

        subgraph AppPlane["Private App Subnet (dr_active=true)"]
            APP2[FlaskApp Pods]
        end

        S3D[(S3<br/>flaskapp-proddata-...)]
        S3T[(S3<br/>flaskapp-tfstate-...)]
        SM[Secrets Manager<br/>flaskapp-db-...]
        KMS_R[KMS: rds key]
        KMS_S[KMS: s3 key]
    end

    USER([Users])
    VPN[Site-to-Site VPN]

    USER -. 정상 .-> APP1
    APP1 --> MDB
    APP1 ==업로드/조회==> S3D
    MDB --> BINLOG
    BINLOG -- VPN: 3306 read --> DMSI
    DMSI -- "Apply (CDC)" --> RDS
    RDS -. Multi-AZ sync .- RDS_SB

    USER -. Failover .-> APP2
    APP2 -. Promoted .-> RDS
    APP2 ==동일 버킷==> S3D

    RDS -. encrypted by .- KMS_R
    S3D -. encrypted by .- KMS_S
    APP2 -. ESO 동기화 .- SM
    SM -. credential .- RDS
```

### 1.2 5가지 핵심 원칙

**1️⃣ 단방향 복제만 (On-prem → AWS)**
- 양방향은 **split-brain**(데이터 갈라짐) 위험
- 비유: 양쪽 사장이 동시에 결재하면 회사 망함

**2️⃣ 단일 버킷 — 사진은 양쪽에서 공유**
- `flaskapp-proddata-...` 하나만 사용
- 환경별로 분리하면 동기화 부담 + 정합성 문제

**3️⃣ 암호화는 기본값**
- RDS, S3, DMS, Secrets 모두 KMS CMK 사용
- AWS 관리 키(aws/rds 등)는 키 정책 못 건드림 → CMK 필수

**4️⃣ 인터넷 노출 0**
- RDS Public Access **영구 비활성**
- 접근은 Security Group으로만 통제

**5️⃣ DR 전환 포인트는 3곳**
- ① RDS Promote (DMS Task 정지)
- ② App 환경변수 (`DATABASE_HOST` → RDS endpoint)
- ③ Route 53 (DNS → AWS ALB)

---

## 2. RDS 인스턴스 설계

### 2.1 기본 설계값

| 항목 | 값 | 왜 이렇게? |
|---|---|---|
| DB Identifier | `rds-flaskapp-kosa-project-team3-snow` | 명명 규칙 |
| Engine | **MariaDB 10.11** (또는 MySQL 8.0) | ⭐ 온프렘 버전과 정확히 일치 필수 |
| Instance Class | `db.t3.medium` (2 vCPU / 4 GiB) | 평시 부하 거의 없음 |
| Storage Type | `gp3` | gp2보다 저렴, IOPS 분리 |
| Allocated Storage | 100 GiB | |
| Max Allocated Storage | 500 GiB | 자동 확장 활성 |
| Storage Encryption | **활성 (KMS CMK)** | 한 번 만들면 변경 불가 |
| Multi-AZ | **활성** | Failover 후 Primary가 되니까 |
| Subnet Group | Private Data Subnet × 2 | |
| **Publicly Accessible** | **false (영구)** | 인터넷 노출 금지 |
| Deletion Protection | **true** | 실수 삭제 방지 |
| Performance Insights | 활성 (7일, 무료) | 쿼리 단위 진단 |
| Enhanced Monitoring | 60초 간격 | OS 레벨 메트릭 |

> ⭐ **엔진 버전 일치 필수**:
> 온프렘이 MariaDB 10.11.6이면 RDS도 10.11.6. 버전 차이로 binlog 포맷이 호환 안 되면 DMS Task가 실패합니다.

### 2.2 Read Replica인가 별도 인스턴스인가?

**중요한 오해 풀기**: 이 설계의 RDS는 **진짜 Read Replica가 아닙니다**.

| 구분 | 진짜 Read Replica | 이 설계 (DMS Target) |
|---|---|---|
| 어디서 데이터 옴? | AWS가 자체 복제 | DMS가 binlog로 흘려 넣음 |
| Promote 명령 | `aws rds promote-read-replica` | 그냥 DMS Task 정지하면 됨 |
| 온프렘 ↔ AWS | 직접 Replica 불가 | DMS 경유 가능 |

> 📌 **명칭만 "Read Replica"**:
> 실제로는 **DMS가 흘려넣어주는 일반 RDS 인스턴스**.
> Failover 시점에 "Promote"라고 부르지만, AWS 콘솔의 "Promote Read Replica" 기능과는 다릅니다. 실제 절차는 §10 참조.

### 2.3 Terraform 코드

```hcl
# modules/rds/main.tf
resource "aws_db_subnet_group" "main" {
  name       = "rds-flaskapp-kosa-project-team3-snow"
  subnet_ids = var.data_subnet_ids

  tags = { Name = "rds-flaskapp-subnet-group" }
}

resource "aws_db_instance" "flaskapp" {
  identifier     = "rds-flaskapp-kosa-project-team3-snow"
  engine         = "mariadb"
  engine_version = "10.11.6"   # 온프렘과 일치
  instance_class = "db.t3.medium"

  allocated_storage     = 100
  max_allocated_storage = 500
  storage_type          = "gp3"
  storage_encrypted     = true
  kms_key_id            = var.kms_rds_arn

  db_name  = "flaskapp"
  username = "admin"
  # ⭐ 비밀번호는 Secrets Manager가 자동 관리 + 자동 회전
  manage_master_user_password   = true
  master_user_secret_kms_key_id = var.kms_secrets_arn

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [var.rds_sg_id]
  publicly_accessible    = false
  multi_az               = true

  parameter_group_name = aws_db_parameter_group.flaskapp.name

  backup_retention_period = 7
  backup_window           = "16:00-17:00"     # UTC (KST 새벽 1~2시)
  maintenance_window      = "sun:17:00-sun:18:00"
  copy_tags_to_snapshot   = true

  deletion_protection       = true
  skip_final_snapshot       = false
  final_snapshot_identifier = "rds-flaskapp-final-${formatdate("YYYYMMDDhhmm", timestamp())}"

  performance_insights_enabled          = true
  performance_insights_retention_period = 7
  performance_insights_kms_key_id       = var.kms_rds_arn

  enabled_cloudwatch_logs_exports = ["error", "slowquery", "audit"]

  monitoring_interval = 60
  monitoring_role_arn = aws_iam_role.rds_monitoring.arn

  apply_immediately = false   # 운영 중에는 false, 명시적 변경만 즉시 적용

  tags = { Name = "rds-flaskapp-kosa-project-team3-snow" }
}
```

> 💡 **`manage_master_user_password = true`의 마법**:
> RDS가 Secrets Manager에 비밀번호를 자동 생성 + 7/30/90일 주기로 자동 회전.
> 운영자가 비번 따로 관리할 필요 없습니다.

---

## 3. RDS Parameter Group — DB의 동작 설정

### 3.1 DMS CDC를 위한 필수 파라미터

DMS가 CDC로 동작하려면 **소스 DB(온프렘 MariaDB)** 에 특정 설정이 필요합니다. **타깃 DB(RDS)** 에도 같이 적용:

| 파라미터 | 값 | 왜? |
|---|---|---|
| `binlog_format` | `ROW` | DMS는 STATEMENT 포맷 지원 안 함 |
| `binlog_row_image` | `FULL` | 부분 row면 PK 외 컬럼 추적 불가 |
| `binlog_checksum` | `NONE` | DMS가 checksum 미지원 |
| `expire_logs_days` | `7` 이상 | DMS 일시 중단 시 따라잡을 시간 확보 |
| `log_bin` | `ON` | binlog 자체 활성화 |
| `server_id` | 고유값 | 양쪽 다 다른 값 |
| `gtid_strict_mode` | `OFF` | DMS 호환 |
| `read_only` | `OFF` (평시) | Failover 후엔 운영 정책 따라 |

> 💡 **이게 빠지면 DMS Task 시작조차 안 됨**.
> "왜 안 되지?" 의 첫 번째 의심 대상이 이 설정들.

### 3.2 Custom Parameter Group Terraform

```hcl
resource "aws_db_parameter_group" "flaskapp" {
  name        = "pg-mariadb-flaskapp-kosa-project-team3-snow"
  family      = "mariadb10.11"
  description = "FlaskApp DR용 — DMS CDC 호환 + 성능 튜닝"

  # DMS CDC 필수
  parameter {
    name  = "binlog_format"
    value = "ROW"
  }
  parameter {
    name  = "binlog_row_image"
    value = "FULL"
  }
  parameter {
    name         = "binlog_checksum"
    value        = "NONE"
    apply_method = "pending-reboot"
  }

  # 성능 튜닝
  parameter {
    name  = "max_connections"
    value = "200"
  }
  parameter {
    name         = "innodb_buffer_pool_size"
    value        = "{DBInstanceClassMemory*3/4}"   # 가용 메모리의 75%
    apply_method = "pending-reboot"
  }
  parameter {
    name  = "slow_query_log"
    value = "1"
  }
  parameter {
    name  = "long_query_time"
    value = "2"   # 2초 이상 쿼리 기록
  }

  # 시간대 + 문자셋
  parameter {
    name  = "time_zone"
    value = "Asia/Seoul"
  }
  parameter {
    name  = "character_set_server"
    value = "utf8mb4"
  }
  parameter {
    name  = "collation_server"
    value = "utf8mb4_unicode_ci"
  }

  tags = { Name = "pg-mariadb-flaskapp-kosa-project-team3-snow" }
}
```

### 3.3 온프렘 MariaDB 측 설정

RDS에만 적용하면 안 됨. **온프렘 MariaDB에도 동일 설정 필요**:

```ini
# /etc/mysql/mariadb.conf.d/50-server.cnf
[mysqld]
server_id              = 1
log_bin                = /var/log/mysql/mariadb-bin
binlog_format          = ROW
binlog_row_image       = FULL
binlog_checksum        = NONE
expire_logs_days       = 7
gtid_strict_mode       = OFF
```

DMS 전용 복제 유저도 만들어야 함:

```sql
CREATE USER 'dms_user'@'%' IDENTIFIED BY '<강력한_비밀번호>';
GRANT REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO 'dms_user'@'%';
GRANT SELECT ON flaskapp.* TO 'dms_user'@'%';
FLUSH PRIVILEGES;
```

> ⚠️ **DMS 유저에게 RELOAD, SUPER 권한 절대 금지** (AWS 공식 가이드).
> 보안 사고 시 영향 범위가 커집니다. 최소 권한만.

---

## 4. RDS 백업·스냅샷·암호화

### 4.1 백업 정책

| 항목 | 값 | 비고 |
|---|---|---|
| 자동 백업 보존 | 7일 | 비용/복구 가능성 균형 |
| 백업 윈도우 | `16:00-17:00 UTC` (KST 새벽 1~2시) | 트래픽 최저 시간 |
| 스냅샷 Cross-Region | 비활성 (선택) | 리전 장애 대비 시 활성 |
| Point-in-Time Recovery | 자동 활성 (백업 기간 내) | **1초 단위 복구 가능** |
| 수동 스냅샷 | 월 1회 (Lambda 자동화 권장) | 자동 백업 만료 후에도 보존 |

> 💡 **Point-in-Time Recovery란?**
> "어제 오후 3시 27분 12초 상태로 돌려줘"가 가능. 자동 백업 기간(7일) 안의 어느 시점이든 OK.

### 4.2 KMS 키 정책 (RDS 전용)

```json
{
  "Version": "2012-10-17",
  "Id": "kms-flaskapp-rds-key-policy",
  "Statement": [
    {
      "Sid": "Account Root Admin",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::<ACCOUNT_ID>:root" },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "Allow RDS to use the key",
      "Effect": "Allow",
      "Principal": { "Service": "rds.amazonaws.com" },
      "Action": [
        "kms:Encrypt", "kms:Decrypt", "kms:ReEncrypt*",
        "kms:GenerateDataKey*", "kms:DescribeKey", "kms:CreateGrant"
      ],
      "Resource": "*"
    },
    {
      "Sid": "Allow Performance Insights",
      "Effect": "Allow",
      "Principal": { "Service": "rds.amazonaws.com" },
      "Action": ["kms:Decrypt", "kms:GenerateDataKey"],
      "Resource": "*",
      "Condition": {
        "StringEquals": { "kms:ViaService": "rds.ap-northeast-2.amazonaws.com" }
      }
    }
  ]
}
```

### 4.3 수동 스냅샷 자동화 (Lambda + EventBridge)

자동 백업은 7일이면 사라집니다. 더 오래 보관할 월 스냅샷을 자동 생성:

```hcl
# modules/rds/snapshot_lambda.tf
resource "aws_cloudwatch_event_rule" "monthly_snapshot" {
  name                = "rds-monthly-snapshot-flaskapp"
  schedule_expression = "cron(0 15 1 * ? *)"   # 매월 1일 UTC 15:00 (KST 24:00)
}

resource "aws_cloudwatch_event_target" "snapshot_lambda" {
  rule = aws_cloudwatch_event_rule.monthly_snapshot.name
  arn  = aws_lambda_function.snapshot.arn
}
```

Lambda는 boto3로 `create_db_snapshot` + 6개월 지난 스냅샷 자동 삭제.

---

## 5. DMS Replication Instance — 복제 일꾼

### 5.1 DMS는 무엇인가?

비유하자면, DMS는 **양쪽 DB 사이에 앉아서 짐을 옮기는 일꾼**입니다:
- 소스(온프렘 MariaDB)에서 binlog 읽기
- 타깃(AWS RDS)에 적용
- 24/7 상시 가동

### 5.2 설계값

| 항목 | 값 | 이유 |
|---|---|---|
| Identifier | `dms-flaskapp-kosa-project-team3-snow` | |
| Engine Version | 최신 안정 버전 (3.5.x) | |
| Instance Class | `dms.t3.medium` (2 vCPU / 4 GiB) | 시작값. Lag 발생 시 업그레이드 |
| Allocated Storage | 50 GiB | binlog 임시 저장 |
| **Multi-AZ** | **활성** | 일꾼 죽으면 RPO 망가짐 |
| Subnet Group | Private Data Subnet × 2 | |
| Publicly Accessible | false | |
| KMS Key | RDS 키 (또는 별도) | 내부 캐시 암호화 |

### 5.3 인스턴스 사이징 가이드

| 워크로드 | 권장 인스턴스 | 예상 처리량 |
|---|---|---|
| 소규모 (< 100 tx/s) | `dms.t3.medium` | ~10 MB/s |
| 중규모 (100~1000 tx/s) | `dms.c5.large` | ~50 MB/s |
| 대규모 (> 1000 tx/s) | `dms.c5.xlarge` | ~200 MB/s |

> 💡 **시작 전략**:
> `dms.t3.medium`으로 시작 → **`CDCLatencyTarget`이 5분 이상으로 치솟으면** 한 단계 업.

### 5.4 Terraform 코드

```hcl
# modules/dms/instance.tf
resource "aws_dms_replication_subnet_group" "main" {
  replication_subnet_group_id          = "dms-subnet-flaskapp"
  replication_subnet_group_description = "DMS subnet group for FlaskApp DR"
  subnet_ids                           = var.data_subnet_ids

  tags = { Name = "dms-subnet-flaskapp" }
}

resource "aws_dms_replication_instance" "main" {
  replication_instance_id      = "dms-flaskapp-kosa-project-team3-snow"
  replication_instance_class   = "dms.t3.medium"
  allocated_storage            = 50
  engine_version               = "3.5.2"
  multi_az                     = true
  publicly_accessible          = false
  auto_minor_version_upgrade   = true
  apply_immediately            = false

  replication_subnet_group_id = aws_dms_replication_subnet_group.main.id
  vpc_security_group_ids      = [var.dms_sg_id]
  kms_key_arn                 = var.kms_rds_arn

  preferred_maintenance_window = "sun:18:00-sun:19:00"

  tags = { Name = "dms-flaskapp-kosa-project-team3-snow" }
}
```

---

## 6. DMS Endpoints — 양쪽 DB 연결 정보

DMS가 **소스/타깃 DB에 어떻게 접속할지** 알려주는 설정. 두 개 만듭니다.

### 6.1 Source Endpoint (온프렘 MariaDB)

| 항목 | 값 |
|---|---|
| Endpoint ID | `dms-src-onprem-mariadb` |
| Endpoint Type | `source` |
| Engine | `mariadb` |
| Server Name | `172.16.43.160` (VPN 너머 사설 IP) |
| Port | `3306` |
| Username | `dms_user` (§3.3에서 생성) |
| Password | Secrets Manager 참조 |
| Database Name | `flaskapp` |
| Extra Connection Attrs | `initstmt=SET FOREIGN_KEY_CHECKS=0;` |

### 6.2 Target Endpoint (RDS)

| 항목 | 값 |
|---|---|
| Endpoint ID | `dms-tgt-rds-mariadb` |
| Endpoint Type | `target` |
| Engine | `mariadb` |
| Server Name | RDS endpoint |
| Port | `3306` |
| Database Name | `flaskapp` |
| Username | `admin` (RDS 관리형 secret에서 추출) |
| Password | RDS 관리형 secret에서 추출 |
| SSL Mode | `require` |
| Extra Connection Attrs | `initstmt=SET FOREIGN_KEY_CHECKS=0;parallelLoadThreads=4;` |

### 6.3 ⚠️ 함정 — RDS 관리형 Secret 호환성 문제

이 부분은 운영자가 가장 많이 헤매는 곳입니다.

**문제**: AWS DMS의 `secrets_manager_arn` 옵션은 secret JSON에 **4개 키 (`host`/`port`/`username`/`password`) 모두** 들어있어야 함. 그런데 RDS의 `manage_master_user_password = true`가 만드는 secret은 **`{username, password}` 두 키만** 가짐.

→ secret ARN을 그대로 넘기면 **endpoint test 실패**.

**해결**: secret에서 username/password만 추출해서 endpoint에 plaintext로 주입하고, host/port는 RDS 출력값을 직접 전달.

```hcl
data "aws_secretsmanager_secret_version" "target" {
  secret_id = var.target_secret_arn   # module.rds.master_user_secret_arn
}

resource "aws_dms_endpoint" "target" {
  endpoint_id   = "dms-tgt-rds-mariadb"
  endpoint_type = "target"
  engine_name   = "mariadb"

  server_name   = var.target_server_name   # RDS endpoint
  port          = var.target_port
  database_name = "flaskapp"
  username      = jsondecode(data.aws_secretsmanager_secret_version.target.secret_string)["username"]
  password      = jsondecode(data.aws_secretsmanager_secret_version.target.secret_string)["password"]
  ssl_mode      = "require"

  extra_connection_attributes = "initstmt=SET FOREIGN_KEY_CHECKS=0;parallelLoadThreads=4;"

  tags = { Name = "dms-tgt-rds-mariadb" }
}
```

> ⚠️ **회전 지연 주의**:
> RDS가 비밀번호 자동 회전 → secret의 새 값은 **다음 `terraform apply` 시점에 DMS endpoint에 반영**.
> 회전 직후 짧은 시간 동안 endpoint password가 stale(낡은) 상태가 될 수 있음.
>
> **대처**:
> - (1) 회전 주기를 30/90일 중 **긴 것**으로
> - (2) host/port 포함한 별도 secret + 회전 Lambda 두기 (정공법)

### 6.4 Endpoint 연결 테스트

```bash
aws dms test-connection \
  --replication-instance-arn <REPL_INSTANCE_ARN> \
  --endpoint-arn <SOURCE_ENDPOINT_ARN>

# 응답에서 status: "successful" 확인
aws dms describe-connections \
  --filters Name=endpoint-arn,Values=<SOURCE_ENDPOINT_ARN>
```

**연결 실패 흔한 4가지 원인**:
1. **VPN Tunnel Down** — Route 53/CloudWatch로 확인
2. **SG 룰 누락** — `dms-sg` 아웃바운드에 `172.16.43.160:3306` 허용했는지
3. **온프렘 방화벽(pfSense) 인바운드 룰 누락**
4. **MariaDB 사용자 권한 부족** — `REPLICATION SLAVE` 누락

---

## 7. DMS Task — 실제 복제 작업

### 7.1 Task 기본 설계

Task는 "이 Source에서 이 Target으로 어떻게 복제할지" 명령서입니다.

| 항목 | 값 | 의미 |
|---|---|---|
| Task Identifier | `dms-task-flaskapp` | |
| Migration Type | `full-load-and-cdc` | 초기 전체 복사 + 이후 변경분만 |
| Target Table Prep Mode | `DO_NOTHING` 또는 `DROP_AND_CREATE` | 기존 테이블 유지 / 초기 적재 시 |
| LOB 처리 모드 | `LIMITED` (32KB) 또는 `FULL` | FULL은 메모리 부담 |
| CDC Start Position | `start_from_now` | Full Load 후 즉시 CDC 시작 |

### 7.2 Table Mappings — 어떤 테이블을 복제할지

JSON으로 규칙 정의:

```json
{
  "rules": [
    {
      "rule-type": "selection",
      "rule-id": "1",
      "rule-name": "include-flaskapp-schema",
      "object-locator": {
        "schema-name": "flaskapp",
        "table-name": "%"
      },
      "rule-action": "include"
    },
    {
      "rule-type": "selection",
      "rule-id": "2",
      "rule-name": "exclude-temp-tables",
      "object-locator": {
        "schema-name": "flaskapp",
        "table-name": "tmp_%"
      },
      "rule-action": "exclude"
    },
    {
      "rule-type": "transformation",
      "rule-id": "3",
      "rule-name": "convert-table-name-to-lower",
      "rule-target": "table",
      "object-locator": {
        "schema-name": "flaskapp",
        "table-name": "%"
      },
      "rule-action": "convert-lowercase"
    }
  ]
}
```

규칙 해석:
- `flaskapp` 스키마의 **모든 테이블 포함** (rule 1)
- 단, `tmp_`로 시작하는 임시 테이블은 **제외** (rule 2)
- 테이블 이름은 **모두 소문자로 변환** (rule 3)

### 7.3 Task Settings — 튜닝 핵심

```json
{
  "TargetMetadata": {
    "TargetSchema": "",
    "SupportLobs": true,
    "LimitedSizeLobMode": true,
    "LobMaxSize": 32,
    "ParallelLoadThreads": 4,
    "ParallelLoadBufferSize": 100
  },
  "FullLoadSettings": {
    "TargetTablePrepMode": "DO_NOTHING",
    "CreatePkAfterFullLoad": false,
    "StopTaskCachedChangesApplied": false,
    "StopTaskCachedChangesNotApplied": false,
    "MaxFullLoadSubTasks": 8,
    "TransactionConsistencyTimeout": 600,
    "CommitRate": 10000
  },
  "Logging": {
    "EnableLogging": true,
    "LogComponents": [
      { "Id": "SOURCE_CAPTURE",  "Severity": "LOGGER_SEVERITY_DEFAULT" },
      { "Id": "SOURCE_UNLOAD",   "Severity": "LOGGER_SEVERITY_DEFAULT" },
      { "Id": "TARGET_APPLY",    "Severity": "LOGGER_SEVERITY_DEFAULT" },
      { "Id": "TARGET_LOAD",     "Severity": "LOGGER_SEVERITY_DEFAULT" }
    ],
    "CloudWatchLogGroup": "dms-task-flaskapp",
    "CloudWatchLogStream": null
  },
  "ControlTablesSettings": {
    "ControlSchema": "dms_control",
    "HistoryTimeslotInMinutes": 5,
    "HistoryTableEnabled": true,
    "SuspendedTablesTableEnabled": true,
    "StatusTableEnabled": true
  },
  "ValidationSettings": {
    "EnableValidation": true,
    "ValidationMode": "ROW_LEVEL",
    "ThreadCount": 5,
    "PartitionSize": 10000,
    "FailureMaxCount": 10000,
    "RecordFailureDelayInMinutes": 5,
    "RecordFailureDelayLimitInMinutes": 30
  },
  "ChangeProcessingDdlHandlingPolicy": {
    "HandleSourceTableDropped": true,
    "HandleSourceTableTruncated": true,
    "HandleSourceTableAltered": true
  },
  "ErrorBehavior": {
    "DataErrorPolicy": "LOG_ERROR",
    "DataTruncationErrorPolicy": "LOG_ERROR",
    "DataErrorEscalationPolicy": "SUSPEND_TABLE",
    "DataErrorEscalationCount": 50,
    "TableErrorPolicy": "SUSPEND_TABLE",
    "TableErrorEscalationPolicy": "STOP_TASK",
    "TableErrorEscalationCount": 50,
    "RecoverableErrorCount": -1,
    "RecoverableErrorInterval": 5,
    "RecoverableErrorThrottling": true,
    "RecoverableErrorThrottlingMax": 1800,
    "ApplyErrorDeletePolicy": "IGNORE_RECORD",
    "ApplyErrorInsertPolicy": "LOG_ERROR",
    "ApplyErrorUpdatePolicy": "LOG_ERROR"
  }
}
```

### 7.4 핵심 설정 의미

| 설정 | 효과 |
|---|---|
| `ValidationMode: ROW_LEVEL` | DMS가 자체적으로 행 단위 일치 검증. **RPO 신뢰성 ↑** |
| `HandleSourceTableAltered: true` | 온프렘 `ALTER TABLE` → RDS에도 자동 적용 (제한적) |
| `DataErrorEscalationPolicy: SUSPEND_TABLE` | 오류 50개 누적 시 해당 테이블만 일시 중지 (전체 Task는 계속) |
| `LimitedSizeLobMode + LobMaxSize: 32` | 32KB 초과 LOB는 truncate. FULL 모드는 메모리 폭주 위험 |
| `ParallelLoadThreads: 4` | Full Load 시 4개 테이블 동시 적재 |

### 7.5 Terraform 코드

```hcl
# modules/dms/task.tf
resource "aws_dms_replication_task" "main" {
  replication_task_id      = "dms-task-flaskapp"
  migration_type           = "full-load-and-cdc"
  replication_instance_arn = aws_dms_replication_instance.main.replication_instance_arn
  source_endpoint_arn      = aws_dms_endpoint.source.endpoint_arn
  target_endpoint_arn      = aws_dms_endpoint.target.endpoint_arn

  table_mappings           = file("${path.module}/table_mappings.json")
  replication_task_settings = file("${path.module}/task_settings.json")

  start_replication_task = true

  tags = { Name = "dms-task-flaskapp" }

  lifecycle {
    ignore_changes = [start_replication_task]   # 수동 시작/중지 허용
  }
}
```

### 7.6 핵심 CloudWatch 메트릭

| 메트릭 | 임계치 | 의미 |
|---|---|---|
| **`CDCLatencyTarget`** | < 5분 | ⭐ 타깃 적용 지연 (RPO 직결) |
| `CDCLatencySource` | < 5분 | 소스에서 binlog 읽는 지연 |
| `CDCThroughputRowsTarget` | 모니터링 | 초당 처리 행 수 |
| `FullLoadThroughputRowsTarget` | 진행률 | 초기 적재 진행 |
| `CPUUtilization` | < 80% | 초과 시 인스턴스 업그레이드 |
| `FreeableMemory` | > 500 MB | LOB 처리 문제 시 부족 |

> ⚠️ **`CDCLatencyTarget` 5분 초과 = RPO 5분 초과**.
> **P1 알람 필수**. PagerDuty + Slack `#ops-critical`.

---

## 8. S3 — `proddata` 버킷 상세

### 8.1 버킷 설정 요약

| 항목 | 값 |
|---|---|
| 버킷명 | `flaskapp-proddata-kosa-project-team3-snow-lai9z` |
| 리전 | `ap-northeast-2` |
| Object Ownership | `BucketOwnerEnforced` (ACL 비활성) |
| Block Public Access | **4개 모두 활성** |
| Versioning | 활성 |
| Default Encryption | SSE-KMS (CMK) |
| Bucket Key | 활성 (KMS 비용 절감) |
| Replication | 비활성 |
| Object Lock | 비활성 |

### 8.2 디렉토리 구조 규약

버킷 안을 정리해서 쓰기:

```
flaskapp-proddata-kosa-project-team3-snow-lai9z/
├── uploads/              # 사용자 업로드 (사진/문서)
│   ├── 2026/05/...
│   └── ...
├── static/               # 정적 자산 (CSS/JS/이미지)
├── backups/              # DB 덤프 등 백업
│   └── db/YYYY-MM-DD.sql.gz
├── alb-access-logs/      # ALB Access Log
└── vpc-flow-logs/        # VPC Flow Log
```

### 8.3 라이프사이클 — 자동으로 저렴한 스토리지로

오래된 파일은 자동으로 더 싼 스토리지 등급으로 이동:

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "proddata" {
  bucket = aws_s3_bucket.proddata.id

  # 사용자 업로드는 점점 저렴한 등급으로
  rule {
    id     = "uploads-tiering"
    status = "Enabled"
    filter { prefix = "uploads/" }

    transition {
      days          = 30
      storage_class = "STANDARD_IA"      # 30일 후 IA (저빈도 접근)
    }
    transition {
      days          = 90
      storage_class = "GLACIER_IR"       # 90일 후 Glacier 즉시 검색
    }
    transition {
      days          = 365
      storage_class = "DEEP_ARCHIVE"     # 1년 후 Deep Archive
    }
  }

  # 로그는 90일 후 삭제
  rule {
    id     = "logs-expire"
    status = "Enabled"
    filter { prefix = "alb-access-logs/" }

    expiration { days = 90 }
  }

  # 이전 버전은 30일 후 삭제
  rule {
    id     = "old-versions-expire"
    status = "Enabled"
    filter {}

    noncurrent_version_expiration { noncurrent_days = 30 }
  }

  # 중단된 멀티파트 업로드 정리
  rule {
    id     = "incomplete-multipart"
    status = "Enabled"
    filter {}

    abort_incomplete_multipart_upload { days_after_initiation = 7 }
  }
}
```

> 💡 **스토리지 등급별 비용** (대략):
> - Standard: $0.025/GB
> - Standard-IA: $0.0138/GB (절반)
> - Glacier IR: $0.005/GB (1/5)
> - Deep Archive: $0.002/GB (1/12)

### 8.4 Bucket Policy — 누가 접근 가능한가

3가지 허용 + 1가지 차단:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::flaskapp-proddata-kosa-project-team3-snow-lai9z",
        "arn:aws:s3:::flaskapp-proddata-kosa-project-team3-snow-lai9z/*"
      ],
      "Condition": {
        "Bool": { "aws:SecureTransport": "false" }
      }
    },
    {
      "Sid": "OnPremIAMUserAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<ACCOUNT_ID>:user/flaskapp-onprem-app"
      },
      "Action": [
        "s3:GetObject", "s3:PutObject", "s3:DeleteObject", "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::flaskapp-proddata-kosa-project-team3-snow-lai9z",
        "arn:aws:s3:::flaskapp-proddata-kosa-project-team3-snow-lai9z/uploads/*",
        "arn:aws:s3:::flaskapp-proddata-kosa-project-team3-snow-lai9z/static/*"
      ]
    },
    {
      "Sid": "EKSIRSAAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<ACCOUNT_ID>:role/eks-irsa-flaskapp-kosa-project-team3-snow"
      },
      "Action": [
        "s3:GetObject", "s3:PutObject", "s3:DeleteObject", "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::flaskapp-proddata-kosa-project-team3-snow-lai9z",
        "arn:aws:s3:::flaskapp-proddata-kosa-project-team3-snow-lai9z/uploads/*",
        "arn:aws:s3:::flaskapp-proddata-kosa-project-team3-snow-lai9z/static/*"
      ]
    },
    {
      "Sid": "ALBAccessLogs",
      "Effect": "Allow",
      "Principal": {
        "Service": "logdelivery.elasticloadbalancing.amazonaws.com"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::flaskapp-proddata-kosa-project-team3-snow-lai9z/alb-access-logs/*"
    }
  ]
}
```

해석:
- 🚫 **HTTP 접근 차단** (HTTPS만 허용)
- ✅ 온프렘 IAM User → uploads/static 폴더
- ✅ EKS IRSA → uploads/static 폴더
- ✅ ALB가 access log 쓰기

> ⚠️ **Deny 룰은 가장 위에**:
> 잘못 설정하면 운영자조차 접근 못 함. 콘솔에서 HTTPS로만 접근하는지 확인.

### 8.5 온프렘 vs AWS 자격증명 방식

| 환경 | 자격증명 방식 | 회전 주기 |
|---|---|---|
| 온프렘 K8s | IAM User Access Key (장기) | 90일 수동 회전 |
| AWS EKS | IRSA (1시간 임시) | 자동 |

> 💡 **장기 액세스 키는 보안 취약점**:
> 가능하면 **IAM Roles Anywhere** (2022~)로 전환 검토. 온프렘에서도 임시 자격증명 사용 가능.

---

## 9. S3 — `tfstate` 버킷 상세

### 9.1 버킷 설정

`proddata`보다 더 엄격하게:

| 항목 | 값 | 왜? |
|---|---|---|
| **Versioning** | **반드시 활성** | state 손상 시 복구 유일 수단 |
| Block Public Access | 4개 모두 활성 | |
| Default Encryption | SSE-KMS | |
| MFA Delete | 활성 권장 | 실수 삭제 방지 |
| Lifecycle | Non-current version 90일 후 삭제 | |

### 9.2 Bucket Policy — TerraformDeployRole만 허용

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAllExceptTerraformRole",
      "Effect": "Deny",
      "NotPrincipal": {
        "AWS": [
          "arn:aws:iam::<ACCOUNT_ID>:role/TerraformDeployRole",
          "arn:aws:iam::<ACCOUNT_ID>:root"
        ]
      },
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::flaskapp-tfstate-kosa-project-team3-snow-a3asx",
        "arn:aws:s3:::flaskapp-tfstate-kosa-project-team3-snow-a3asx/*"
      ]
    },
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::flaskapp-tfstate-kosa-project-team3-snow-a3asx",
        "arn:aws:s3:::flaskapp-tfstate-kosa-project-team3-snow-a3asx/*"
      ],
      "Condition": {
        "Bool": { "aws:SecureTransport": "false" }
      }
    }
  ]
}
```

`NotPrincipal`은 "이 사람들 빼고 전부 거부"라는 뜻. 즉 TerraformDeployRole과 Root만 접근 가능.

### 9.3 DynamoDB Lock 테이블

```hcl
resource "aws_dynamodb_table" "tf_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  server_side_encryption {
    enabled     = true
    kms_key_arn = var.kms_secrets_arn
  }

  tags = { Name = "terraform-state-lock" }
}
```

| 항목 | 값 | 의미 |
|---|---|---|
| Partition Key | `LockID` (String) | 잠금 식별자 |
| Billing Mode | On-demand | 호출 적어서 저렴 |
| 암호화 | KMS CMK | 메타데이터도 보호 |

### 9.4 State 복구 시나리오

누가 state를 망쳐버렸을 때:

```bash
# 1. 이전 버전 목록 조회
aws s3api list-object-versions \
  --bucket flaskapp-tfstate-kosa-project-team3-snow-a3asx \
  --prefix envs/dr/terraform.tfstate

# 2. 특정 버전 복구
aws s3api copy-object \
  --bucket flaskapp-tfstate-kosa-project-team3-snow-a3asx \
  --copy-source 'flaskapp-tfstate-kosa-project-team3-snow-a3asx/envs/dr/terraform.tfstate?versionId=<VERSION_ID>' \
  --key envs/dr/terraform.tfstate

# 3. terraform plan으로 검증
terraform plan
```

> 💡 **Versioning이 생명줄**:
> 이게 없으면 state 망가지는 순간 인프라 통제력 상실. 절대 비활성화하지 마세요.

---

## 10. Failover & Failback 명령 매트릭스

### 10.1 Failover (On-prem → AWS)

총괄 문서 §7의 9단계와 1:1 매칭.

#### Step 1~3: 감지 & 선언

```bash
# Health Check 상태 확인
aws route53 get-health-check-status --health-check-id <HC_ID>

# DMS lag 확인 — 5분 미만이어야 안전한 전환
aws cloudwatch get-metric-statistics \
  --namespace AWS/DMS \
  --metric-name CDCLatencyTarget \
  --dimensions Name=ReplicationInstanceIdentifier,Value=dms-flaskapp-kosa-project-team3-snow Name=ReplicationTaskIdentifier,Value=dms-task-flaskapp \
  --start-time $(date -u -d '10 min ago' +%FT%T) \
  --end-time $(date -u +%FT%T) \
  --period 60 --statistics Maximum
```

#### Step 4: RDS Promote (실제 동작)

**중요**: 본 설계의 RDS는 **DMS Target이지 Read Replica가 아님**. 그래서 절차가 다릅니다.

```bash
# 4-1. DMS Task 정지 (양쪽 동기화 멈춤)
aws dms stop-replication-task \
  --replication-task-arn <TASK_ARN>

# 4-2. Task가 STOPPED 상태일 때까지 대기
aws dms describe-replication-tasks \
  --filters Name=replication-task-arn,Values=<TASK_ARN> \
  --query 'ReplicationTasks[0].Status'

# 4-3. RDS는 이미 쓰기 가능 상태! 추가 명령 불필요.
#      대신 의도치 않은 쓰기 방지를 위해
#      read_only를 잠시 ON했다가 OFF로 전환 가능.

# 4-4. RDS endpoint 확인
aws rds describe-db-instances \
  --db-instance-identifier rds-flaskapp-kosa-project-team3-snow \
  --query 'DBInstances[0].Endpoint.Address'
```

> 📌 **진짜 Read Replica였다면** `aws rds promote-read-replica` 명령 필요.
> 본 설계는 DMS Target이라 **불필요**.

> ⚠️ **DMS endpoint stale password 주의**:
> RDS 비번 자동 회전 **직후** Failover를 트리거하면, DMS target endpoint의 password가 이전 값일 수 있음.
> Failover 전에 `terraform apply` 한 번 돌려 동기화하거나, **회전 주기를 길게** 잡아 안정 구간에서 전환.

#### Step 5~8: 인프라 활성화

```bash
# 5. Terraform으로 EKS/노드/ALB 생성
cd terraform/envs/dr
terraform apply -var="dr_active=true" -auto-approve

# 6. App 배포 (ArgoCD 자동 또는 수동)
kubectl apply -k k8s/overlays/dr/

# 7. 환경변수 확인
kubectl get secret flaskapp-db-secret -n flaskapp -o jsonpath='{.data.DATABASE_URL}' | base64 -d
# → rds-flaskapp-... 포함되어 있어야 함

# 8. Route 53 Failover 확인
dig +short flaskapp.example.com
# → ALB IP가 응답해야 함
```

#### Step 9: 검증

```bash
# HTTPS 헬스체크
curl -sf https://flaskapp.example.com/healthz || echo "FAIL"

# DB 쓰기 테스트
curl -X POST https://flaskapp.example.com/api/test-record -d '{"test":"failover"}'

# S3 쓰기 테스트
curl -X POST -F "file=@test.jpg" https://flaskapp.example.com/api/upload
aws s3 ls s3://flaskapp-proddata-kosa-project-team3-snow-lai9z/uploads/ | tail -1
```

### 10.2 Failback (AWS → On-prem) — 가장 어려운 절차

**데이터 정합성이 최우선**. 자동화하지 말고 매뉴얼 + 운영자 판단으로:

```bash
# 1. AWS RDS를 read-only로 (쓰기 차단)
aws rds modify-db-parameter-group \
  --db-parameter-group-name pg-mariadb-flaskapp-kosa-project-team3-snow \
  --parameters "ParameterName=read_only,ParameterValue=1,ApplyMethod=immediate"

# 2. AWS App 일시 점검 모드 (운영 정책에 따라)

# 3. AWS RDS → 온프렘 MariaDB 역방향 동기화
#    옵션 A: mysqldump 후 적재 (다운타임)
#    옵션 B: 임시 DMS Task 역방향

# 옵션 A 예시:
mysqldump -h <RDS_ENDPOINT> -u admin -p flaskapp \
  --single-transaction --master-data=2 --gtid \
  > flaskapp-failback-$(date +%Y%m%d).sql

# 온프렘에서 적재:
mysql -h 172.16.43.160 -u admin -p flaskapp < flaskapp-failback-$(date +%Y%m%d).sql

# 4. 양쪽 행 수 비교 검증
mysql -h 172.16.43.160 -u admin -p -e "SELECT COUNT(*) FROM flaskapp.users;"
mysql -h <RDS_ENDPOINT> -u admin -p -e "SELECT COUNT(*) FROM flaskapp.users;"

# 5. Route 53을 다시 Primary(온프렘)로 (Health Check 회복 시 자동)

# 6. Terraform으로 EKS 제거 (비용 절감)
cd terraform/envs/dr
terraform apply -var="dr_active=false" -auto-approve

# 7. 원래 방향의 DMS Task 재시작 (온프렘 → AWS)
aws dms start-replication-task \
  --replication-task-arn <TASK_ARN> \
  --start-replication-task-type resume-processing
```

### 10.3 Split-brain 방지 체크

가장 위험한 상황: 양쪽이 동시에 쓰기를 받는 상태.

| 시나리오 | 위험 | 대응 |
|---|---|---|
| 온프렘 회복 후 양쪽 동시 쓰기 | 데이터 충돌 | Failback 전 **AWS RDS를 read_only**로 |
| Failover 직후 온프렘 일시 회복 | 일부 트래픽이 다시 온프렘으로 | Route 53 TTL 60초 + 운영자가 수동으로 온프렘 차단 |
| DMS Task 자동 재시작이 역방향 데이터 덮어씀 | 신규 데이터 손실 | `lifecycle.ignore_changes` + 수동 재시작 원칙 |

> ⚠️ **Failback은 절대 자동화하지 말 것**.
> 매뉴얼 절차 + 운영자 판단이 안전. 자동화 시 split-brain 위험이 폭증합니다.

---

## 11. Terraform 모듈 구조

### 11.1 디렉토리

```
terraform/modules/
├── rds/
│   ├── README.md
│   ├── versions.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── main.tf              # aws_db_instance
│   ├── subnet_group.tf
│   ├── parameter_group.tf
│   ├── iam_monitoring.tf    # Enhanced Monitoring Role
│   └── snapshot_lambda.tf   # 월 스냅샷 자동화 (선택)
│
├── dms/
│   ├── README.md
│   ├── versions.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── instance.tf          # replication instance + subnet group
│   ├── endpoints.tf         # source + target
│   ├── task.tf              # replication task
│   ├── table_mappings.json
│   └── task_settings.json
│
└── s3/
    ├── README.md
    ├── versions.tf
    ├── variables.tf
    ├── outputs.tf
    ├── proddata.tf          # 기존 import + 정책 보강
    ├── tfstate.tf           # 기존 import + 정책 보강
    ├── lifecycle.tf
    └── policy.tf
```

### 11.2 모듈 입출력

```hcl
# modules/rds/variables.tf
variable "data_subnet_ids" {
  type = list(string)
}

variable "rds_sg_id" {
  type = string
}

variable "kms_rds_arn" {
  type = string
}

variable "kms_secrets_arn" {
  type = string
}

variable "engine_version" {
  type    = string
  default = "10.11.6"
}

variable "instance_class" {
  type    = string
  default = "db.t3.medium"
}

# outputs.tf
output "endpoint"          { value = aws_db_instance.flaskapp.endpoint }
output "address"           { value = aws_db_instance.flaskapp.address }
output "port"              { value = aws_db_instance.flaskapp.port }
output "master_secret_arn" { value = aws_db_instance.flaskapp.master_user_secret[0].secret_arn }
output "parameter_group"   { value = aws_db_parameter_group.flaskapp.name }
```

### 11.3 envs/dr/main.tf 호출

```hcl
module "rds" {
  source = "../../modules/rds"

  data_subnet_ids = module.network.data_subnet_ids
  rds_sg_id       = module.security.rds_sg_id
  kms_rds_arn     = module.kms.rds_arn
  kms_secrets_arn = module.kms.secrets_arn
}

module "dms" {
  source = "../../modules/dms"

  data_subnet_ids       = module.network.data_subnet_ids
  dms_sg_id             = module.security.dms_sg_id
  kms_rds_arn           = module.kms.rds_arn
  onprem_db_ip          = "172.16.43.160"
  rds_endpoint          = module.rds.address
  rds_master_secret_arn = module.rds.master_secret_arn
}

module "s3" {
  source = "../../modules/s3"

  proddata_bucket_name = "flaskapp-proddata-kosa-project-team3-snow-lai9z"
  tfstate_bucket_name  = "flaskapp-tfstate-kosa-project-team3-snow-a3asx"
  kms_s3_arn           = module.kms.s3_arn
  onprem_iam_user_arn  = var.onprem_iam_user_arn
  irsa_role_arn        = try(module.eks.irsa_flaskapp_arn, "")
}
```

### 11.4 기존 리소스 import — 2 stack 전략

State 책임을 두 stack에 **나눠** 관리:

- **bootstrap stack** → `tfstate 버킷` + `DynamoDB lock 테이블`
- **envs/dr stack** → `proddata` + `ECR`

```bash
# bootstrap stack
cd terraform/bootstrap
terraform init
terraform import aws_s3_bucket.tfstate     flaskapp-tfstate-kosa-project-team3-snow-a3asx
terraform import aws_dynamodb_table.lock   terraform-state-lock

# envs/dr stack
cd terraform/envs/dr
terraform init
terraform import 'module.s3.aws_s3_bucket.proddata'   flaskapp-proddata-kosa-project-team3-snow-lai9z
terraform import 'module.ecr.aws_ecr_repository.this' flaskapp
```

> 💡 **왜 stack을 나누나?**
> 두 stack 모두 `tfstate` 버킷을 관리하면 SSE/lifecycle 설정에서 **drift(불일치)가 영구적으로 발생**.
> **bootstrap 단독 소유** 원칙. envs/dr는 `backend "s3"` 선언만 두고 리소스 블록은 만들지 않습니다.

---

## 12. 검증 체크리스트

### Phase 1: RDS 기본 동작

- [ ] `aws rds describe-db-instances --db-instance-identifier rds-flaskapp-kosa-project-team3-snow` → Status `available`
- [ ] Multi-AZ `true`
- [ ] Publicly Accessible `false`
- [ ] Storage Encrypted `true` (KMS Key ARN 올바른지)
- [ ] Deletion Protection `true`
- [ ] Parameter Group이 custom으로 적용됨 (`InSync` 상태)

### Phase 2: RDS 연결 & 권한

- [ ] EKS Pod에서 `mysql -h <RDS_ENDPOINT> -u admin -p` 성공
- [ ] DMS SG에서 RDS:3306 접근 가능
- [ ] **외부 인터넷에서 RDS:3306 접근 차단** (`nc -zv` 타임아웃)
- [ ] Secrets Manager의 비밀번호로 로그인 가능

### Phase 3: DMS 인스턴스 & Endpoint

- [ ] Replication Instance Status `available`
- [ ] Multi-AZ 활성 확인
- [ ] `aws dms test-connection`이 Source/Target 모두 `successful`
- [ ] CloudWatch Log Group `dms-task-flaskapp` 생성됨

### Phase 4: DMS Task 동작

- [ ] Task Status가 Full Load → `load-complete` 진행
- [ ] CDC 단계 진입 후 `CDCLatencyTarget < 5분` 유지
- [ ] 온프렘에서 테스트 INSERT 후 **30초 내 RDS에 반영**
- [ ] Validation Mode 활성 시 `ValidationFailedCount == 0`
- [ ] Table Mappings 룰대로 `tmp_%` 테이블은 복제 제외 확인

### Phase 5: S3 — proddata

- [ ] 4개 Block Public Access 모두 활성
- [ ] Versioning Enabled
- [ ] Default Encryption SSE-KMS
- [ ] HTTP 접근 시 차단 (`curl http://...` → 403)
- [ ] 온프렘 IAM User로 PUT/GET 성공
- [ ] EKS IRSA로 PUT/GET 성공
- [ ] 라이프사이클 룰 동작 (`aws s3api get-bucket-lifecycle-configuration`)

### Phase 6: S3 — tfstate

- [ ] Versioning Enabled (state 복구 가능)
- [ ] Bucket Policy로 TerraformDeployRole 외 차단
- [ ] DynamoDB Lock 정상 동작 (`terraform plan` 시 lock 획득/해제 확인)
- [ ] State 파일 KMS 암호화

### Phase 7: Failover 시뮬레이션

- [ ] DMS Task 중지 → 즉시 lag 증가 (의도된 동작)
- [ ] Task 재시작 → 누락 데이터 자동 catchup
- [ ] 가짜 Failover 시나리오로 **RTO/RPO 측정**
- [ ] Failback 절차에서 split-brain 발생 안 함 확인

---

📎 상위: [05. 데이터](./05-data.md) | 인덱스: [README](../../README.md)

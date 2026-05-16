# AWS Terraform 작업 규칙

## 1. 목적

AWS DR 인프라를 Terraform으로 안전하게 관리하기 위한 팀 작업 규칙을 정의한다.

AWS 리소스는 실제 비용이 발생할 수 있고, Terraform state 기준으로 관리되므로 코드 작성, plan, apply, destroy 기준을 명확히 구분한다.

## 2. 기본 원칙

- 코드 작성과 `terraform plan`은 자유롭게 수행할 수 있다.
- `terraform apply`와 `terraform destroy`는 팀 합의 후 수행한다.
- 비용 발생 리소스는 `apply` 전 반드시 확인한다.
- 온프렘 연동 정보가 확정되기 전에는 AWS 리소스를 무리하게 생성하지 않는다.
- 기본 작업은 코드 작성과 `plan` 검증 후 GitHub에 반영하는 방식으로 진행한다.

## 3. 실행 환경

| 항목           | 값                                               |
| -------------- | ------------------------------------------------ |
| AWS Profile    | 개인 로컬 환경에 맞게 설정. 예: `kosa-team-snow` |
| Region         | `ap-northeast-2`                                 |
| Terraform 경로 | `infra/aws/terraform/envs/dr`                    |

## 4. 작업 단계

1. 코드 수정
2. `terraform fmt -recursive`
3. `terraform init`
4. `terraform validate`
5. `terraform plan`
6. PR 리뷰 또는 팀 확인
7. 승인 후 `terraform apply`

기본 실행 예시는 아래와 같다.

```bash
cd infra/aws/terraform/envs/dr

export AWS_PROFILE=<본인 AWS profile 이름>
export AWS_REGION=ap-northeast-2

terraform init
terraform validate
terraform plan -var="dr_active=false"
```

DR 활성화 모드 계획은 아래 명령어로 확인한다.

```bash
terraform plan -var="dr_active=true"
```

예를 들어 로컬 profile 이름이 `kosa-team-snow`라면 아래처럼 설정한다.

```bash
export AWS_PROFILE=kosa-team-snow
export AWS_REGION=ap-northeast-2
```

## 5. `dr_active` 기준

| 값      | 의미                  |
| ------- | --------------------- |
| `false` | 평시 Pilot Light 모드 |
| `true`  | DR 활성화 모드        |

평시에는 최소 리소스만 유지한다.
DR 활성화 시에는 장애 대응에 필요한 리소스를 추가로 생성한다.

예시:

| 리소스      | `dr_active=false` | `dr_active=true` |
| ----------- | ----------------: | ---------------: |
| NAT Gateway |               1개 |              2개 |
| EKS         |            미생성 |             생성 |
| ALB         |            미생성 |             생성 |

## 6. 비용 주의 리소스

아래 리소스는 생성 시 비용이 발생하므로 `apply` 전 반드시 확인한다.

- NAT Gateway
- Elastic IP
- RDS
- DMS
- EKS
- EC2 Worker Node
- ALB
- VPC Endpoint
- CloudWatch Logs

`terraform plan`은 비용이 발생하지 않는다.
비용은 `terraform apply`로 실제 리소스를 생성할 때 발생한다.

## 7. Commit 금지 파일

아래 파일은 GitHub에 commit하지 않는다.

```text
.terraform/
terraform.tfstate
terraform.tfstate.backup
terraform.tfvars
.env
*.pem
*.key
```

AWS Access Key, Secret Key, DB 비밀번호 같은 민감정보도 코드, 문서, 이슈 본문에 기록하지 않는다.

## 8. Commit 가능 파일

아래 파일은 commit 가능하다.

```text
*.tf
*.md
terraform.tfvars.example
.terraform.lock.hcl
```

`.terraform.lock.hcl`은 Terraform provider 버전을 고정하기 위한 파일이므로 commit한다.

## 9. Apply 기준

`terraform apply`는 다음 조건을 확인한 뒤 수행한다.

- `terraform validate` 성공
- `terraform plan` 결과 팀 공유
- 비용 발생 리소스 확인
- 작업 담당자와 팀 합의 완료
- 온프렘 연동이 필요한 작업은 관련 값 확정 후 진행

온프렘 연동 전에는 VPC/Subnet/Route Table 등 독립적인 네트워크 뼈대 작업까지만 우선 검토한다.

## 10. Destroy 기준

`terraform destroy`는 리소스를 삭제하는 작업이므로 반드시 별도 승인 후 수행한다.

삭제 전 확인할 내용:

- 삭제 대상 리소스
- 서비스 영향
- Terraform state 위치
- 복구 가능 여부
- 비용 정리 목적 여부

승인 없는 `apply`와 `destroy`는 금지한다.

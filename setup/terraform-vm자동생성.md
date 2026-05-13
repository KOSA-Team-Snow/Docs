# Terraform Proxmox VM Implementation Review

## 목적

Terraform으로 Proxmox VM 생성 과정을 코드화하고, 이후 Ansible이 사용할 inventory까지 자동 생성하는 구조를 검증했다.

이번 작업의 핵심 목표는 아래와 같다.

- Proxmox cloud-init template 기반 VM clone 자동화
- Ceph datastore `TEAM3`에 VM disk 생성
- VLAN tag와 static IP를 cloud-init으로 주입
- Kubernetes control plane / worker 및 운영 보조 VM 생성
- Ansible inventory 자동 생성
- 반복 실행 가능한 인프라 생성 절차 확보

## 최종 역할 분리

### Terraform

Terraform은 VM 인프라 레이어를 담당한다.

- Proxmox provider 설정
- Template VM `9000` clone
- VMID, VM 이름, 배치 Proxmox 노드 지정
- CPU, memory, disk 설정
- VLAN tag 설정
- static IP, gateway, DNS, SSH key cloud-init 주입
- Ceph datastore `TEAM3`에 VM disk 생성
- Ansible inventory 생성

### Terraform에서 제외

아래 항목은 Terraform에서 제외한다.

- pfSense
- kubeadm init/join
- HAProxy/Keepalived 세부 설정
- MetalLB
- ArgoCD Application
- Monitoring stack

### Ansible

Ansible은 VM 생성 이후 내부 OS 설정을 담당한다.

- 공통 패키지 설치
- swap 비활성화
- kernel module 설정
- sysctl 설정
- containerd 설정
- kubelet/kubeadm/kubectl 설치
- HAProxy/Keepalived 패키지 설치

## 작업 디렉터리

```text
terraform/proxmox-vms/
```

주요 파일:

| 파일 | 역할 |
| --- | --- |
| `versions.tf` | Terraform 및 provider 버전 정의 |
| `providers.tf` | Proxmox API provider 설정 |
| `variables.tf` | VM 목록, IP, VLAN, 스펙, 배치 노드 정의 |
| `main.tf` | Proxmox VM clone 및 cloud-init 설정 |
| `outputs.tf` | VM 계획과 inventory 경로 출력 |
| `terraform.tfvars.example` | 전체 VM 생성용 입력 예시 |
| `terraform.tfvars.test.example` | 테스트 VM 1대 생성용 입력 예시 |
| `templates/ansible-inventory.yml.tftpl` | Ansible inventory 템플릿 |

## 사전 준비

### 0. Terraform 실행 서버 조건

- Proxmox API 접근 가능 
- Proxmox provider 다운로드 가능
- Proxmox API 토큰 보유
- SSH public key 보유
- 생성될 VM IP 대역에 접근 가능
- Ansible 가지 이어서 실행하려면 SSH priviate key 도 있어야한다. 

### 1. Terraform 실행 VM 준비

Terraform은 Proxmox API에 접근 가능한 관리 VM에서 실행했다.

Terraform 실행 VM에서 필요한 조건:

- `terraform` 설치
- `https://172.16.30.11:8006/` 접근 가능
- SSH public key 존재

SSH key가 없을 경우 생성:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
```

### 2. Proxmox API token 준비

Proxmox에서 Terraform용 group, user, API token을 생성하고 VM clone/config 권한을 부여했다.

참고 사이트 : https://medium.com/geekculture/connect-proxmox-and-terraform-using-provider-and-api-854a91f4b65d

Terraform 파일에는 실제 secret을 저장하지 않고, `terraform.tfvars`에만 입력한다.

```hcl
proxmox_api_token = "terraform@pve!team3=REPLACE_WITH_TOKEN_SECRET"
```

`terraform.tfvars`는 `.gitignore`에 포함되어야 한다.

### 3. Proxmox template 확인

이번 작업에서는 이미 생성된 template VM `9000`을 사용했다.

Proxmox host에서 확인:

```bash
qm config 9000
```

확인 포인트:

- template VM이 존재
- cloud-init template로 사용 가능
- disk가 `TEAM3`에 있음
- template disk 크기가 `32G`이므로 clone VM disk는 32G 이상이어야 함

## 테스트 VM 검증

전체 VM 생성 전에 테스트 VM 1대를 먼저 생성했다.

사용 파일:

```text
terraform.tfvars.test.example
```

테스트 VM:

| VMID | 이름 | 노드 | IP | VLAN |
| --- | --- | --- | --- | --- |
| `250` | `tf-test-vlan30` | `kosa-team3-01` | `172.16.43.150` | `30` |

실행:

```bash
cd terraform/proxmox-vms
vi terraform.tfvars
terraform init
terraform plan
terraform apply
```

확인:

```bash
ping -c 3 172.16.43.150
ssh kosa@172.16.43.150
```

테스트 VM 생성이 성공하면서 아래 흐름이 검증되었다.

- Proxmox provider 인증
- Template `9000` clone
- Ceph `TEAM3` disk 생성
- cloud-init static IP 주입
- VLAN 30 적용
- SSH key 주입
- VM 부팅 및 접속

## 전체 VM 생성

테스트 성공 후 전체 VM 생성을 진행했다.

실행:

```bash
cp terraform.tfvars.example terraform.tfvars
vi terraform.tfvars
terraform plan -parallelism=1
terraform apply -parallelism=1
```

`-parallelism=1`을 사용한 이유:

- Terraform은 기본적으로 여러 리소스를 병렬 생성한다.
- 여러 VM full clone이 동시에 `TEAM3`에 접근하면 Proxmox/Ceph storage lock timeout이 발생할 수 있다.
- `-parallelism=1`로 VM을 하나씩 생성하면 lock 충돌 가능성을 줄일 수 있다.

## 생성 대상 VM

| VMID | 이름 | Proxmox 노드 | IP | VLAN |
| --- | --- | --- | --- | --- |
| `201` | `k8s-cp-1` | `kosa-team3-01` | `172.16.43.100` | `30` |
| `202` | `k8s-cp-2` | `kosa-team3-02` | `172.16.43.101` | `30` |
| `203` | `k8s-cp-3` | `kosa-team3-03` | `172.16.43.102` | `30` |
| `211` | `k8s-worker-1` | `kosa-team3-01` | `172.16.43.110` | `30` |
| `212` | `k8s-worker-2` | `kosa-team3-02` | `172.16.43.111` | `30` |
| `213` | `k8s-worker-3` | `kosa-team3-03` | `172.16.43.112` | `30` |
| `221` | `lb-1` | `kosa-team3-04` | `172.16.42.100` | `20` |
| `222` | `lb-2` | `kosa-team3-05` | `172.16.42.101` | `20` |
| `231` | `bastion` | `kosa-team3-01` | `172.16.44.100` | `40` |
| `232` | `monitoring` | `kosa-team3-04` | `172.16.44.101` | `40` |
| `233` | `argocd` | `kosa-team3-03` | `172.16.44.102` | `40` |
| `241` | `mariadb-1` | `kosa-team3-05` | `172.16.43.160` | `30` |


## Terraform state 기준

Terraform은 `terraform.tfstate`에 성공적으로 생성한 리소스를 기록한다.

다시 `terraform apply`를 실행하면:

- state에 있는 VM은 다시 만들지 않는다.
- state에 없는 VM만 생성하려고 한다.
- 코드에서 제거된 VM은 삭제 대상으로 표시한다.
- 일부 속성 변경은 재생성으로 계획될 수 있다.

따라서 실패 후에는 아래 두 상태를 비교해야 한다.

```bash
terraform state list
qm list
```

Proxmox에는 VM이 있지만 Terraform state에는 없다면:

- Terraform import를 하거나
- 해당 VM을 삭제하고 다시 apply한다.

이번 프로젝트 진행에서는 빠른 복구를 위해 부분 생성된 VM/disk를 정리하고 재실행하는 방식을 사용했다.

## 결과물

Terraform apply 이후 생성되는 주요 결과:

- Proxmox VM들
- Ceph `TEAM3` VM disk
- cloud-init 기반 static IP 설정
- SSH key 기반 접속 가능 VM
- Ansible inventory

Inventory 경로:

```text
ansible/inventories/onprem/hosts.yml
```

다음 단계:

```bash
cd ansible
ansible -m ping all
ansible-playbook playbooks/k8s-prereq.yml
ansible-playbook playbooks/lb-prereq.yml
```


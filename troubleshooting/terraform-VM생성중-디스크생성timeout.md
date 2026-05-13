## 주요 오류와 대응

### 1. Disk resize failure

오류:

```text
requested size (20G) is lower than current size (32G)
```

원인:

- Template `9000`의 disk가 이미 32G였다.
- Terraform에서 일부 VM disk를 20G 또는 30G로 지정해 template보다 작은 크기로 clone하려고 했다.

대응:

- 모든 VM disk를 최소 32G 이상으로 수정했다.
- `lb-1`, `lb-2`, `bastion` disk를 32G 이상으로 변경했다.## 주요 오류와 대응

### 1. Disk resize failure

오류:

```text
requested size (20G) is lower than current size (32G)
```

원인:

- Template `9000`의 disk가 이미 32G였다.
- Terraform에서 일부 VM disk를 20G 또는 30G로 지정해 template보다 작은 크기로 clone하려고 했다.

대응:

- 모든 VM disk를 최소 32G 이상으로 수정했다.
- `lb-1`, `lb-2`, `bastion` disk를 32G 이상으로 변경했다.

### 2. Storage lock timeout

오류:

```text
cfs-lock 'storage-TEAM3' error: got lock request timeout
HTTP 596 response - Reason: Connection timed out
```

원인:

- 여러 VM full clone이 동시에 `TEAM3` storage lock을 요청했다.
- Proxmox/Ceph 응답 지연으로 Terraform provider가 timeout을 받았다.

대응:

```bash
terraform apply -parallelism=1
```

### 3. Cloud-init RBD 잔재

오류:

```text
rbd create 'vm-202-cloudinit' error: (17) File exists
rbd image vm-232-cloudinit already exists
```

원인:

- VM clone 과정 초반에 cloud-init RBD volume이 먼저 생성된다.
- 이후 timeout이나 clone 실패가 발생하면 VM config는 생성되지 않았지만 `vm-*-cloudinit`만 남을 수 있다.
- 다음 apply에서 같은 VMID로 다시 cloud-init disk를 만들려고 하면서 `File exists` 오류가 발생한다.

확인:

```bash
pvesm list TEAM3 | egrep 'vm-201|vm-202|vm-203|vm-211|vm-212|vm-213|vm-221|vm-222|vm-231|vm-232|vm-233|vm-241'
```

정리 예시:

```bash
pvesm free TEAM3:vm-202-cloudinit
pvesm free TEAM3:vm-203-cloudinit
pvesm free TEAM3:vm-231-cloudinit
pvesm free TEAM3:vm-232-cloudinit
```

주의:

- VM이 실제로 존재하는 경우에는 먼저 `qm list`로 확인해야 한다.
- VM이 존재하면 `pvesm free`로 disk만 지우지 말고 `qm destroy <VMID> --purge`를 사용한다.

## 상태 확인 명령

Terraform 실행 VM에서:

```bash
terraform state list
terraform output
```

Proxmox host에서:

```bash
qm list
pvesm list TEAM3
```

특정 VM 확인:

```bash
qm list | grep 222
pvesm list TEAM3 | grep 'vm-222'
```

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


### 2. Storage lock timeout

오류:

```text
cfs-lock 'storage-TEAM3' error: got lock request timeout
HTTP 596 response - Reason: Connection timed out
```

원인:

- 여러 VM full clone이 동시에 `TEAM3` storage lock을 요청했다.
- Proxmox/Ceph 응답 지연으로 Terraform provider가 timeout을 받았다.

대응:

```bash
terraform apply -parallelism=1
```

### 3. Cloud-init RBD 잔재

오류:

```text
rbd create 'vm-202-cloudinit' error: (17) File exists
rbd image vm-232-cloudinit already exists
```

원인:

- VM clone 과정 초반에 cloud-init RBD volume이 먼저 생성된다.
- 이후 timeout이나 clone 실패가 발생하면 VM config는 생성되지 않았지만 `vm-*-cloudinit`만 남을 수 있다.
- 다음 apply에서 같은 VMID로 다시 cloud-init disk를 만들려고 하면서 `File exists` 오류가 발생한다.

확인:

```bash
pvesm list TEAM3 | egrep 'vm-201|vm-202|vm-203|vm-211|vm-212|vm-213|vm-221|vm-222|vm-231|vm-232|vm-233|vm-241'
```

정리 예시:

```bash
pvesm free TEAM3:vm-202-cloudinit
pvesm free TEAM3:vm-203-cloudinit
pvesm free TEAM3:vm-231-cloudinit
pvesm free TEAM3:vm-232-cloudinit
```

주의:

- VM이 실제로 존재하는 경우에는 먼저 `qm list`로 확인해야 한다.
- VM이 존재하면 `pvesm free`로 disk만 지우지 말고 `qm destroy <VMID> --purge`를 사용한다.

## 상태 확인 명령

Terraform 실행 VM에서:

```bash
terraform state list
terraform output
```

Proxmox host에서:

```bash
qm list
pvesm list TEAM3
```

특정 VM 확인:

```bash
qm list | grep 222
pvesm list TEAM3 | grep 'vm-222'
```
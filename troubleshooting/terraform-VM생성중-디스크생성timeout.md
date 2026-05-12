### 문제 상황

대상 VM 정보:

- VM 이름: `lb-2`
- VMID: `222`
- 배치 노드: `kosa-team3-05`
- IP: `172.16.42.101`
- VLAN: `20`
- Storage: `TEAM3`
- Template: `9000`

전체 VM 생성 과정에서 `lb-2` 생성/조회 중 Proxmox API timeout이 발생했다.

<img width="1050" height="139" alt="Image" src="https://github.com/user-attachments/assets/74c16a41-5e5e-4600-8d42-689e95970c30" />

현재 추정 원인:

- 이전 실패로 인해 `TEAM3:vm-222-*` 디스크 잔재가 남았을 가능성
- Proxmox에는 일부 리소스가 생성되었지만 Terraform state에는 반영되지 않은 반쪽 생성 상태 가능성
- Ceph datastore `TEAM3` lock 또는 응답 지연
- `kosa-team3-05` 노드 또는 `TEAM3` storage 조회 지연 가능성

현재 대응 방향:

1. Terraform state에서 `lb-2`가 관리 대상에 들어갔는지 확인
2. Proxmox에서 VMID `222` 존재 여부 확인
3. Ceph storage `TEAM3`에 `vm-222-*` 디스크 잔재가 있는지 확인
4. 잔재가 있으면 정리 후 `terraform apply -parallelism=1`로 재실행
5. 동일 문제가 반복되면 `lb-2`의 VMID를 `222`에서 다른 번호로 변경 검토

### 로그 / 에러

```shell

```

### 확인한 내용 / 다음 액션

```
# Terraform 실행 VM에서
terraform state list | grep 'lb-2'

# Proxmox host에서
qm list | grep 222
pvesm list TEAM3 | grep 'vm-222'

# Proxmox에 VM 222가 있으면
qm destroy 222 --purge

# VM은 없고 TEAM3에 디스크 잔재만 있으면
pvesm free TEAM3:vm-222-disk-0
pvesm free TEAM3:vm-222-cloudinit

# Terraform state에 lb-2가 남아있는데 실제 VM이 없으면
terraform state rm 'proxmox_virtual_environment_vm.onprem["lb-2"]'
```

pvesm free TEAM3:vm-222-disk-0
pvesm free TEAM3:vm-222-cloudinit
명령어로 오류 해결!
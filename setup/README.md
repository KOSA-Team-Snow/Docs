# Setup 문서 인덱스

이 디렉터리는 최초 구축 절차, 설치 방법, 초기 설정 문서를 관리한다.

## 문서 상태

| 문서 | 상태 | 설명 |
| --- | --- | --- |
| [초기-하드웨어-세팅-과정.md](./초기-하드웨어-세팅-과정.md) | 초기 기록 | 물리/네트워크/Proxmox 구축 기록. 일부 MetalLB 계획은 과거 기준 |
| [proxmox-ceph-storage-registration.md](./proxmox-ceph-storage-registration.md) | 구축 기록 | Proxmox에 Ceph RBD/CephFS 등록 |
| [VM-Temaplate-생성.md](./VM-Temaplate-생성.md) | 구축 기준 | Kubernetes-ready Ubuntu VM template 생성 |
| [terraform-vm자동생성.md](./terraform-vm자동생성.md) | 구축 기록 | Terraform Proxmox VM 생성과 inventory 자동 생성 |
| [ansible-kubernetes-bootstrap-review.md](./ansible-kubernetes-bootstrap-review.md) | 구축 기록 | Ansible 기반 Kubernetes bootstrap 리뷰 |
| [kubernetes-ansible-helm-manifest-final-documentation.md](./kubernetes-ansible-helm-manifest-final-documentation.md) | 구축 기록 | Kubernetes/Helm/Manifest 작업 정리 |

## 최신 기준

- Kubernetes 노드는 VM template clone 후 Terraform/Ansible로 구성한다.
- VM disk와 template disk는 Ceph RBD `TEAM3` 기준이다.
- Kubernetes 클러스터는 control plane 3대, worker 5대다.
- 외부 사용자 진입점은 MetalLB가 아니라 HAProxy/Keepalived VIP `172.16.42.99`다.

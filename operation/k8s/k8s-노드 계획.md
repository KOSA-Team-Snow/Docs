# Day 2 - 1.Kubernetes 노드 배치 계획

## 1. 작업 목적

팀원 B의 Kubernetes Platform 역할에서 온프레미스 Kubernetes 클러스터의 Control Plane과 Worker Node 배치 계획을 확정한다.

이번 프로젝트의 물리 구조는 5대 Proxmox 노드, pfSense 기반 VLAN 분리, 1G 관리망, 10G 스토리지망으로 구성된다. Kubernetes 노드는 내부망인 VLAN 30에 배치하고, 외부 접근과 운영자 접근은 각각 VLAN 10/20/40 역할에 맞게 분리한다.

## 2. 기준 네트워크 구조

| 구분 | 대역 | 용도 |
| --- | --- | --- |
| 1G 관리망 | `172.16.30.0/24` | Proxmox 호스트, 라우터, 스위치, pfSense WAN 관리 |
| VLAN 10 | `172.16.41.100-172.16.41.199` | 공용망, Ingress, MetalLB, 외부 접근 서비스 |
| VLAN 20 | `172.16.42.100-172.16.42.199` | DMZ, HAProxy, reverse proxy |
| VLAN 30 | `172.16.43.100-172.16.43.199` | Kubernetes node, DB, 내부 서비스 |
| VLAN 40 | `172.16.44.100-172.16.44.199` | Bastion, kubectl, ansible, 운영자 접근 |
| 10G 스토리지망 | `10.10.10.39-10.10.10.43` | Proxmox/Ceph 전용 |

Kubernetes VM의 Node IP는 VLAN 30을 사용한다. 10G 스토리지망은 Proxmox/Ceph 전용으로 유지하고, Kubernetes 서비스 IP나 Pod 통신망으로 사용하지 않는다.

## 3. 노드 배치 결정 요약

| 역할 | VM | IP | Proxmox 배치 추천 | 설명 |
| --- | --- | --- | --- | --- |
| Control Plane | `k8s-cp-1` | `172.16.43.100` | `pve-1` | 첫 번째 Kubernetes 제어 노드 |
| Control Plane | `k8s-cp-2` | `172.16.43.101` | `pve-3` | 두 번째 Kubernetes 제어 노드 |
| Control Plane | `k8s-cp-3` | `172.16.43.102` | `pve-4` | 세 번째 Kubernetes 제어 노드 |
| Worker | `k8s-worker-1` | `172.16.43.110` | `pve-2` | 애플리케이션 Pod 실행 노드 |
| Worker | `k8s-worker-2` | `172.16.43.111` | `pve-5` | 애플리케이션 Pod 실행 노드 |
| Worker | `k8s-worker-3` | `172.16.43.112` | `pve-1`, `pve-3`, `pve-4` 중 여유 노드 | 애플리케이션 Pod 실행 노드 |

Control Plane은 3대로 구성한다. 각 Control Plane VM은 서로 다른 Proxmox 물리 노드에 배치해서 특정 물리 노드 장애가 Kubernetes 제어부 전체 장애로 이어지지 않도록 한다.

`pve-2`에는 pfSense VM이 배치되므로 Control Plane 배치는 피하고 Worker Node 위주로 사용한다. pfSense는 VLAN 라우팅과 외부 접근 경로의 중심이기 때문에, 네트워크 핵심 VM과 Kubernetes 제어부를 같은 물리 노드에 묶지 않는 것이 좋다.

## 4. Kubernetes API VIP 계획

Kubernetes API Server 접근을 위해 VLAN 30 내부망에 VIP를 둔다.

| 항목 | 값 |
| --- | --- |
| API VIP | `172.16.43.10` |
| 대상 | `k8s-cp-1`, `k8s-cp-2`, `k8s-cp-3` |
| 구성 후보 | `kube-vip` 또는 HAProxy/Keepalived |

프로젝트 설명 관점에서는 HAProxy/Keepalived가 네트워크 구조를 설명하기 쉽다. Kubernetes 친화적인 방식으로 구성하려면 `kube-vip`를 사용할 수 있다.

## 5. CIDR 계획

Kubernetes Pod CIDR와 Service CIDR는 VLAN 대역과 겹치지 않게 별도 사설 대역으로 설정한다.

| 구분 | 대역 |
| --- | --- |
| Node CIDR | `172.16.43.0/24` |
| Pod CIDR 후보 | `10.244.0.0/16` |
| Service CIDR 후보 | `10.96.0.0/12` |

CNI를 Calico로 선택할 경우 Pod CIDR는 실제 설치 매니페스트 또는 kubeadm 설정과 일치시킨다.

## 6. VLAN별 역할 분리 방향

Kubernetes 노드는 VLAN 30에 두되, 모든 접근을 VLAN 30으로 직접 열지 않는다.

```text
외부 사용자
  -> VLAN 10 Ingress VIP 또는 MetalLB IP
  -> VLAN 20 DMZ Load Balancer 또는 Reverse Proxy
  -> VLAN 30 Kubernetes Service
  -> VLAN 30 Pod / DB
```

운영자는 VLAN 40의 Bastion을 통해 Kubernetes 클러스터에 접근한다.

```text
운영자
  -> VLAN 40 bastion
  -> kubectl / ansible / ssh
  -> VLAN 30 Kubernetes Node
```

이 구조를 사용하면 VLAN 30은 내부망 역할을 유지하고, 외부 노출 지점은 VLAN 10/20으로 분리할 수 있다.

## 7. 팀원별 협업 포인트

| 팀원 | 확인할 내용 |
| --- | --- |
| 네트워크 담당 | pfSense VLAN 30 Gateway `172.16.43.1`, Trunk/Access 포트, VLAN 라우팅 정책 |
| Kubernetes 담당 | kubeadm HA Control Plane, API VIP, CNI, Node join 순서 |
| 애플리케이션 담당 | FlaskApp Service, Ingress, DB 연결 endpoint |
| 모니터링 담당 | Prometheus/Grafana 배포 위치와 외부 접근 방식 |
| AWS DR 담당 | on-prem node 구조와 EKS node group 구조 비교 |

## 8. 완료 기준

- Kubernetes Control Plane 3대의 IP와 배치 대상 Proxmox 노드가 정리되어 있다.
- Worker Node 3대의 IP와 배치 대상 Proxmox 노드가 정리되어 있다.
- Kubernetes API VIP 후보가 정리되어 있다.
- Pod CIDR와 Service CIDR 후보가 VLAN 대역과 겹치지 않게 정리되어 있다.
- VLAN 10/20/30/40의 역할 분리가 Kubernetes 배치 계획에 반영되어 있다.
- `pve-2`의 pfSense VM 배치를 고려해 Control Plane을 분산하는 이유가 정리되어 있다.

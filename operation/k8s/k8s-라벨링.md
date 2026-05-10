# Day 2 - Kubernetes 노드명, Role, Label 계획

## 1. 작업 목적

팀원 B의 Kubernetes Platform 역할에서 kubeadm 기반 HA Kubernetes 클러스터에 사용할 노드명, role, label, taint/toleration 기준을 정리한다.

이번 작업은 WBS Day 2의 완료 기준인 `노드명, role, label 계획 작성`에 해당한다. 이후 Day 5의 HA 클러스터 구성, Day 6의 namespace/RBAC/Ingress 구성, Day 9-10의 HPA 및 Taints/Tolerations 적용 작업의 기준으로 사용한다.

## 2. 기준 환경

Kubernetes 노드는 팀 네트워크 설계에 따라 VLAN 30 내부망에 배치한다.

| 구분 | 값 |
| --- | --- |
| Kubernetes Node VLAN | VLAN 30 |
| VLAN 30 Gateway | `172.16.43.1` |
| Kubernetes API VIP 후보 | `172.16.43.10` |
| Pod CIDR 후보 | `10.244.0.0/16` |
| Service CIDR 후보 | `10.96.0.0/12` |
| 설치 방식 | kubeadm |
| Container Runtime | containerd |
| CNI 후보 | Calico 우선, Flannel 대안 |

10G `10.10.10.x` 대역은 Proxmox/Ceph 스토리지 전용으로 유지한다. Kubernetes Node IP, Service IP, Pod CIDR에는 사용하지 않는다.

## 3. 노드명 및 역할 계획

| Hostname | IP | Kubernetes Role | Proxmox 배치 후보 | 주요 용도 |
| --- | --- | --- | --- | --- |
| `k8s-cp-1` | `172.16.43.100` | control-plane | `pve-1` | kubeadm init 기준 Control Plane |
| `k8s-cp-2` | `172.16.43.101` | control-plane | `pve-3` | HA Control Plane |
| `k8s-cp-3` | `172.16.43.102` | control-plane | `pve-4` | HA Control Plane |
| `k8s-worker-1` | `172.16.43.110` | worker | `pve-2` | FlaskApp, 일반 애플리케이션 Pod |
| `k8s-worker-2` | `172.16.43.111` | worker | `pve-5` | FlaskApp, 일반 애플리케이션 Pod |
| `k8s-worker-3` | `172.16.43.112` | worker/infra 후보 | `pve-1`, `pve-3`, `pve-4` 중 여유 노드 | Ingress, ArgoCD, Monitoring 등 infra Pod 후보 |

Control Plane은 3대로 구성한다. 각 Control Plane VM은 서로 다른 Proxmox 물리 노드에 분산해서 특정 물리 노드 장애가 Kubernetes 제어부 전체 장애로 이어지지 않도록 한다.

`pve-2`에는 pfSense VM이 있으므로 Control Plane 배치를 피한다. pfSense는 VLAN 라우팅과 외부 접근 경로의 중심이기 때문에, Kubernetes 제어부와 네트워크 핵심 VM을 같은 물리 노드에 묶지 않는 방향으로 계획한다.

## 4. Kubernetes 기본 Role 기준

kubeadm으로 Control Plane을 구성하면 Control Plane 노드에는 기본적으로 다음 label이 붙는다.

```text
node-role.kubernetes.io/control-plane=
```

Worker Node는 기본 상태에서는 명시적인 role label이 없을 수 있으므로, 운영 목적에 맞게 직접 label을 추가한다.

## 5. Label 계획

### 5.1 공통 Label

모든 Kubernetes 노드에는 환경과 위치를 식별할 수 있는 label을 부여한다.

| Label Key | Label Value | 대상 | 목적 |
| --- | --- | --- | --- |
| `env` | `onprem` | 전체 노드 | On-prem/EKS 환경 구분 |
| `topology.kubernetes.io/zone` | `onprem-pve-1` 등 | 전체 노드 | Proxmox 물리 노드 위치 구분 |
| `network.vlan` | `vlan30` | 전체 노드 | Kubernetes Node VLAN 식별 |

### 5.2 Role Label

| Hostname | 추가 Label | 목적 |
| --- | --- | --- |
| `k8s-cp-1` | `node-role=control-plane` | Control Plane 식별 |
| `k8s-cp-2` | `node-role=control-plane` | Control Plane 식별 |
| `k8s-cp-3` | `node-role=control-plane` | Control Plane 식별 |
| `k8s-worker-1` | `node-role=app` | FlaskApp 등 일반 애플리케이션 배치 |
| `k8s-worker-2` | `node-role=app` | FlaskApp 등 일반 애플리케이션 배치 |
| `k8s-worker-3` | `node-role=infra` | Ingress, ArgoCD, Monitoring 배치 후보 |

### 5.3 Workload Label

| Label Key | Label Value | 대상 | 배치 후보 |
| --- | --- | --- | --- |
| `workload` | `system` | `k8s-cp-*` | Kubernetes 제어부 |
| `workload` | `app` | `k8s-worker-1`, `k8s-worker-2` | FlaskApp, 일반 서비스 |
| `workload` | `infra` | `k8s-worker-3` | Ingress Controller, ArgoCD, Prometheus/Grafana |

## 6. Label 적용 명령어 초안

클러스터 구성 후 다음 명령어로 label을 적용한다.

```bash
kubectl label node k8s-cp-1 env=onprem network.vlan=vlan30 node-role=control-plane workload=system topology.kubernetes.io/zone=onprem-pve-1
kubectl label node k8s-cp-2 env=onprem network.vlan=vlan30 node-role=control-plane workload=system topology.kubernetes.io/zone=onprem-pve-3
kubectl label node k8s-cp-3 env=onprem network.vlan=vlan30 node-role=control-plane workload=system topology.kubernetes.io/zone=onprem-pve-4

kubectl label node k8s-worker-1 env=onprem network.vlan=vlan30 node-role=app workload=app topology.kubernetes.io/zone=onprem-pve-2
kubectl label node k8s-worker-2 env=onprem network.vlan=vlan30 node-role=app workload=app topology.kubernetes.io/zone=onprem-pve-5
kubectl label node k8s-worker-3 env=onprem network.vlan=vlan30 node-role=infra workload=infra topology.kubernetes.io/zone=onprem-pve-extra
```

`k8s-worker-3`의 `topology.kubernetes.io/zone` 값은 실제 VM 배치가 확정된 뒤 `onprem-pve-1`, `onprem-pve-3`, `onprem-pve-4` 중 하나로 수정한다.

## 7. Taint / Toleration 계획

Control Plane 노드에는 일반 애플리케이션 Pod를 배치하지 않는 것을 기본 원칙으로 한다. kubeadm 기본 taint가 적용되어 있으면 유지한다.

```text
node-role.kubernetes.io/control-plane:NoSchedule
```

`k8s-worker-3`은 infra node 후보로 사용한다. Ingress Controller, ArgoCD, Prometheus/Grafana 같은 인프라성 Pod를 분리 배치할 수 있도록 다음 taint를 적용할 수 있다.

```bash
kubectl taint node k8s-worker-3 dedicated=infra:NoSchedule
```

infra Pod에는 다음 toleration과 nodeSelector를 적용한다.

```yaml
nodeSelector:
  node-role: infra

tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "infra"
    effect: "NoSchedule"
```

프로젝트 초기 구축 단계에서는 모든 Worker Node를 일반 worker로 사용하고, Day 10의 Taints/Tolerations 작업 시점에 `k8s-worker-3`을 infra node로 분리해도 된다.

## 8. 워크로드 배치 기준

| 워크로드 | 배치 대상 | 기준 |
| --- | --- | --- |
| Kubernetes Control Plane | `k8s-cp-1`, `k8s-cp-2`, `k8s-cp-3` | kube-apiserver, etcd, scheduler, controller-manager |
| FlaskApp | `k8s-worker-1`, `k8s-worker-2` | HPA 테스트와 일반 서비스 배치 |
| Nginx Ingress Controller | `k8s-worker-3` 우선 | 외부 진입 경로 담당 |
| ArgoCD | `k8s-worker-3` 우선 | GitOps 관리 도구 |
| Prometheus/Grafana | `k8s-worker-3` 우선 | 모니터링 구성 요소 |
| BusyBox 테스트 Pod | `k8s-worker-*` | DNS, Service, DB 연결 확인 |

## 9. 확인 명령어

노드 구성 후 다음 명령어로 상태를 확인한다.

```bash
kubectl get nodes -o wide
kubectl get nodes --show-labels
kubectl describe node k8s-worker-3
kubectl get pods -A -o wide
```

label 기준 배치가 필요한 경우 다음 명령어로 대상 노드를 확인한다.

```bash
kubectl get nodes -l node-role=app
kubectl get nodes -l node-role=infra
kubectl get nodes -l env=onprem,network.vlan=vlan30
```

## 10. 완료 기준

- Kubernetes 노드명이 `k8s-cp-*`, `k8s-worker-*` 형식으로 정리되어 있다.
- Control Plane 3대와 Worker Node 3대의 role이 정리되어 있다.
- 각 노드의 VLAN 30 IP와 Proxmox 배치 후보가 정리되어 있다.
- 공통 label, role label, workload label 기준이 정리되어 있다.
- infra node 후보와 taint/toleration 적용 방향이 정리되어 있다.
- FlaskApp, Ingress, ArgoCD, Monitoring의 배치 기준이 정리되어 있다.
- 클러스터 구성 후 label과 node 상태를 검증할 명령어가 정리되어 있다.

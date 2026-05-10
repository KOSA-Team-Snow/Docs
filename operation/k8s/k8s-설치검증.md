# Day 1 - Kubernetes 설치 검증 및 Day 2 TODO 정리

## 1. 작업 목적

팀원 B의 Kubernetes Platform 역할에서 kubeadm 기반 Kubernetes 설치 후 반드시 확인해야 할 검증 명령어를 정리한다.

Day 1에서는 실제 클러스터 설치를 완료하기 전이라도, 설치가 끝났을 때 어떤 기준으로 정상 여부를 판단할지 먼저 정의한다. 또한 Day 2에서 확정해야 할 노드명, role, label, IP, CNI 관련 TODO를 체크리스트로 정리해 이후 작업의 기준으로 사용한다.

## 2. 설치 후 검증 대상

설치 검증은 `k8s-cp-1`에서 `kubectl` 접근이 가능한 상태를 기준으로 수행한다.

| 검증 구분 | 목적 |
| --- | --- |
| Node 상태 | Control Plane과 Worker Node가 Kubernetes 클러스터에 정상 Join 되었는지 확인 |
| kube-system Pod 상태 | CoreDNS, kube-proxy, CNI 관련 Pod가 정상 동작하는지 확인 |
| cluster-info | Kubernetes API Server와 CoreDNS 접근 정보 확인 |
| 테스트 Pod 배포 | Pod 생성, 스케줄링, DNS, Service 통신 가능 여부 확인 |

## 3. Node 상태 확인

클러스터 구성 후 전체 노드가 `Ready` 상태인지 확인한다.

```bash
kubectl get nodes
kubectl get nodes -o wide
```

예상 기준:

```text
NAME           STATUS   ROLES           AGE   VERSION   INTERNAL-IP
k8s-cp-1       Ready    control-plane   ...   v1.xx.   172.16.43.100
k8s-cp-2       Ready    control-plane   ...   v1.xx.   172.16.43.101
k8s-cp-3       Ready    control-plane   ...   v1.xx.   172.16.43.102
k8s-worker-1   Ready    <none>          ...   v1.xx.   172.16.43.110
k8s-worker-2   Ready    <none>          ...   v1.xx.   172.16.43.111
k8s-worker-3   Ready    <none>          ...   v1.xx.   172.16.43.112
```

확인 포인트:

- 모든 Control Plane과 Worker Node가 목록에 보여야 한다.
- `STATUS`가 `Ready`여야 한다.
- `INTERNAL-IP`가 VLAN 30 대역인 `172.16.43.`로 표시되어야 한다.
- Worker Node의 `ROLES`가 `<none>`으로 보일 경우 Day 2 label 계획에 따라 별도 label을 적용한다.

## 4. kube-system Pod 상태 확인

Kubernetes 기본 구성 요소가 정상 실행 중인지 확인한다.

```bash
kubectl get pods -n kube-system
kubectl get pods -n kube-system -o wide
```

확인 포인트:

- `coredns` Pod가 `Running` 상태여야 한다.
- `kube-proxy` Pod가 각 노드에 배포되어 있어야 한다.
- CNI 설치 후 Calico 또는 Flannel 관련 Pod가 `Running` 상태여야 한다.
- `CrashLoopBackOff`, `ImagePullBackOff`, `Pending` 상태가 없어야 한다.

문제가 있을 경우 다음 명령어로 원인을 확인한다.

```bash
kubectl describe pod <pod-name> -n kube-system
kubectl logs <pod-name> -n kube-system
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

## 5. cluster-info 확인

Kubernetes API Server와 CoreDNS 접근 정보를 확인한다.

```bash
kubectl cluster-info
```

확인 포인트:

- Kubernetes control plane 주소가 API VIP 또는 Control Plane endpoint로 표시되어야 한다.
- CoreDNS 정보가 출력되어야 한다.
- API VIP를 사용할 경우 endpoint가 `172.16.43.10` 기준으로 접근 가능한지 확인한다.

추가 확인:

```bash
kubectl config current-context
kubectl config view --minify
```

## 6. 테스트 Pod 배포 확인

Pod 생성과 스케줄링이 가능한지 BusyBox 또는 Nginx Pod로 확인한다.

```bash
kubectl run nginx-test --image=nginx --restart=Never
kubectl get pod nginx-test -o wide
```

확인 포인트:

- `nginx-test` Pod가 `Running` 상태가 되어야 한다.
- Pod가 Worker Node에 스케줄링되는지 확인한다.
- Control Plane taint가 유지되는 경우 일반 Pod는 Control Plane에 배치되지 않아야 한다.

테스트가 끝나면 Pod를 삭제한다.

```bash
kubectl delete pod nginx-test
```

DNS와 외부 통신은 BusyBox로 확인한다.

```bash
kubectl run busybox-test --image=busybox:1.36 --restart=Never -- sleep 3600
kubectl exec busybox-test -- nslookup kubernetes.default
kubectl delete pod busybox-test
```

## 7. Day 2 TODO 체크리스트

Day 2에서는 실제 설치 전에 노드 계획과 네트워크/CNI 기준을 확정한다.

- [] Kubernetes 노드명을 `k8s-cp-*`, `k8s-worker-*` 형식으로 정리한다.
- [] Control Plane 3대와 Worker Node 3대의 role을 정리한다.
- [] 각 노드의 VLAN 30 IP를 정리한다.
- [] 각 노드의 Proxmox 배치 후보를 정리한다.
- [] Kubernetes API VIP 후보를 정리한다.
- [] Pod CIDR와 Service CIDR 후보를 정리한다.
- [] 공통 label 기준을 정리한다.
- [] role label과 workload label 기준을 정리한다.
- [] infra node 후보와 taint/toleration 적용 방향을 정리한다.
- [] CNI는 Calico 우선, Flannel 대안으로 정리하고 Day 3 설치 초안에서 적용 기준을 이어간다.

## 8. Day 2 산출물 연결

Day 2 TODO는 다음 문서로 이어진다.

| TODO 구분 | Day 2 산출물 |
| --- | --- |
| 노드 배치, IP, API VIP, CIDR | `docs/day2/issue1-kubernetes-node-placement-plan.md` |
| 노드명, role, label, taint/toleration | `docs/day2/issue2-kubernetes-node-role-label-plan.md` |

## 9. 완료 기준

- 설치 후 노드 상태 확인 명령어가 정리되어 있다.
- `kube-system` Pod 상태 확인 명령어가 정리되어 있다.
- `cluster-info` 확인 명령어가 정리되어 있다.
- 테스트 Pod 배포 가능 여부 확인 항목이 정리되어 있다.
- Day 2에서 확정할 노드명 TODO가 체크리스트로 정리되어 있다.
- Day 2에서 확정할 role, label, IP 계획 TODO가 체크리스트로 정리되어 있다.
- CNI 선택 확정 TODO가 정리되어 있다.
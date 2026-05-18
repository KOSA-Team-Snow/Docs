# Kubernetes Ansible, Helm, Manifest 작업 정리

## 목적

이 문서는 우리 팀 Kubernetes HA Control Plane 구성 이후 진행한 Ansible, Helm, manifest 작업을 정리한다.
현재 기준으로 Ansible은 VM 내부 설정과 kubeadm 기반 클러스터 bootstrap을 담당하고, Kubernetes addon 리소스는 manifest 또는 추후 GitOps/Helm 방식으로 분리해서 관리한다.

## 현재 인프라 전제

- Kubernetes version: `v1.30.1`
- Control Plane:
  - `k8s-cp-1`: `172.16.43.100`
  - `k8s-cp-2`: `172.16.43.101`
  - `k8s-cp-3`: `172.16.43.102`
- Worker:
  - `k8s-worker-1`: `172.16.43.110`
  - `k8s-worker-2`: `172.16.43.111`
  - `k8s-worker-3`: `172.16.43.112`
  - `k8s-worker-4`: `172.16.43.113`
  - `k8s-worker-5`: `172.16.43.114`

- Kubernetes API VIP: `172.16.43.99:6443`
- CNI target: Calico `v3.28.5`
- Pod CIDR: `10.244.0.0/16`
- Service CIDR: `10.96.0.0/12`
- 관리 실행 위치: `bastion`

## 역할 구분

### Ansible

Ansible은 서버 내부 설정과 실행 절차 자동화를 담당한다.

- apt/dpkg 사전 정리
- containerd 설정
- kubelet, kubeadm, kubectl 설치
- kubeadm init/join 실행
- kube-vip static pod manifest 배치
- kubeconfig 복사
- control plane/worker join 검증
- bastion에서 `kubectl` 실행 환경 준비

### Helm

Helm은 Kubernetes addon을 chart/release 단위로 관리할 때 사용한다.

이번 Calico 설치에서는 Helm 방식을 시도했지만, Kubernetes `v1.30.1`과 Calico `v3.28.5` 조합에서 CRD chart 방식이 맞지 않아 manifest 방식으로 전환했다.

현재 Helm 관련 파일은 참고/legacy 성격으로 남겨둔다.

- `ansible/playbooks/06_calico_helm.yml`
- `ansible/files/calico-values.yml`

### Manifest

Manifest는 Kubernetes 안에 생성될 리소스 자체를 정의한다.

Calico manifest는 다음을 정의한다.

- Tigera Operator
- Calico CRD
- `Installation` custom resource
- `APIServer` custom resource
- Calico 네트워크 설정
- Pod CIDR `10.244.0.0/16`
- VXLAN
- BGP Disabled

Manifest는 Ansible과 별개로 관리하고, Ansible은 필요하면 manifest를 적용하는 실행 계층으로만 사용한다.

## 추천 디렉터리 구조

repo 기준으로 Kubernetes manifest는 아래 위치에 두는 것을 추천한다.

```text
kubernetes/
  calico/
    README.md
    tigera-operator.yaml
    custom-resources.yaml
```

현재 bastion에서 수동 작업 중이라면 실제 경로는 다음과 같이 둘 수 있다.

```text
~/kubernetes/calico/
  tigera-operator.yaml
  custom-resources.yaml
```

## Ansible 파일 정리

### 표준 흐름

`ansible/playbooks/01_pre.yml`

- apt background service 정리
- dpkg 상태 복구
- apt cache update

`ansible/playbooks/02_setup.yml`

- Kubernetes 노드 공통 설정
- swap 비활성화
- kernel module, sysctl 설정
- containerd 설정
- kubelet/kubeadm/kubectl 설치
- kubelet node IP 설정

`ansible/playbooks/03_init.yml`

- `k8s-cp-1`에서 첫 control plane 초기화
- 임시로 `172.16.43.99/32`를 붙여 kubeadm init 통과
- `/home/kosa/.kube/config` 생성

`ansible/playbooks/03_kube_vip_handoff.yml`

- `k8s-cp-1`에 kube-vip static pod manifest 배치
- `super-admin.conf`를 사용해 kube-vip가 API server에 접근하도록 구성
- kube-vip가 API VIP를 관리하는지 확인

`ansible/playbooks/04_join.yml`

- `k8s-cp-1`에서 join command 생성
- `k8s-cp-2`, `k8s-cp-3` control plane join
- worker join
- cp2/cp3에는 `/etc/kubernetes/admin.conf` 기반 kube-vip manifest 배치
- 최종 `kubectl get nodes -o wide` 출력

주의: 이미 cp2/cp3와 kube-vip failover를 수동으로 완료한 상태라면 전체 실행하지 않는다. worker만 조인하려면 아래처럼 제한 실행한다.

```bash
ansible-playbook playbooks/04_join.yml --limit 'k8s-cp-1,worker'
```

`ansible/playbooks/05_verify_control_plane_failover.yml`

- 각 control plane의 kube-vip manifest 확인
- kube-vip 컨테이너 running 확인
- 사용자 kubeconfig 확인
- API VIP readiness 확인

### Reset helper

`ansible/playbooks/k8s-reset-first-control-plane.yml`

- 실패한 `kubeadm init` 이후 cp1 상태 정리

`ansible/playbooks/k8s-reset-join-nodes.yml`

- cp2/cp3/worker join 실패 흔적 정리
- `/etc/kubernetes`, `/var/lib/etcd`, kubelet/CNI 잔재 제거

## Helm 파일 정리

`ansible/playbooks/06_calico_helm.yml`

- bastion에 `kubectl`, `helm` 설치
- cp1의 `admin.conf`를 bastion으로 복사
- kubeconfig server를 `https://172.16.43.99:6443`으로 변경
- Helm repo 추가
- Calico CRD/Operator 설치 시도

현재 이 파일은 Calico 표준 설치 흐름에서 제외한다.

이유:

- Calico `v3.28.5`에서는 `crd.projectcalico.org.v1` Helm chart 방식이 맞지 않았다.
- CRD 적용 중 `last-applied-configuration` annotation 크기 제한 문제가 발생했다.
- Tigera Operator와 CRD set 불일치로 `LogCollector` CRD 관련 오류가 발생했다.

`ansible/files/calico-values.yml`

- Helm 설치 시 사용하던 values 파일
- manifest 방식으로 전환한 뒤에는 참고용으로 둔다.

## Calico manifest 설치 흐름

### 1. Helm 기반 Calico 잔재 정리

bastion에서 실행한다.

```bash
helm uninstall calico -n tigera-operator
kubectl delete namespace tigera-operator --ignore-not-found=true
kubectl delete namespace calico-system --ignore-not-found=true
kubectl get crd -o name | egrep 'tigera|calico|projectcalico' | xargs -r kubectl delete
```

정리 확인:

```bash
helm list -A
kubectl get ns | egrep 'tigera|calico' || true
kubectl get crd | egrep 'tigera|calico|projectcalico' || true
```

### 2. Manifest 디렉터리 생성

```bash
mkdir -p ~/kubernetes/calico
cd ~/kubernetes/calico
```

### 3. Tigera Operator manifest 다운로드

```bash
curl -L -o tigera-operator.yaml \
  https://raw.githubusercontent.com/projectcalico/calico/v3.28.5/manifests/tigera-operator.yaml
```

### 4. Custom resources manifest 생성

`custom-resources.yaml`

```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  cni:
    type: Calico
  calicoNetwork:
    bgp: Disabled
    ipPools:
      - cidr: 10.244.0.0/16
        encapsulation: VXLAN
        natOutgoing: Enabled
        nodeSelector: all()
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
```

### 5. Manifest 적용

```bash
kubectl create -f tigera-operator.yaml
```

이미 일부 리소스가 남아 있다면:

```bash
kubectl apply --server-side --force-conflicts -f tigera-operator.yaml
```

API Service IP 접근 문제가 있으면 `tigera-operator` namespace에 endpoint override를 둔다.

```bash
kubectl -n tigera-operator create configmap kubernetes-services-endpoint \
  --from-literal=KUBERNETES_SERVICE_HOST=172.16.43.99 \
  --from-literal=KUBERNETES_SERVICE_PORT=6443 \
  --dry-run=client -o yaml | kubectl apply -f -
```

그리고 custom resource를 적용한다.

```bash
kubectl apply -f custom-resources.yaml
```

## 설치 검증

```bash
kubectl get pods -n tigera-operator -o wide
kubectl get pods -n calico-system -o wide
kubectl get nodes -o wide
```

watch:

```bash
watch -n 2 'kubectl get pods -n tigera-operator; echo; kubectl get pods -n calico-system 2>/dev/null; echo; kubectl get nodes -o wide'
```

정상 목표:

```text
tigera-operator Pod Running
calico-system namespace 생성
calico-node Pod Running
calico-kube-controllers Running
노드 STATUS Ready
```

## 자주 발생한 문제와 판단 기준

### bastion에서 API VIP 접근 실패

```text
Unable to connect to the server: dial tcp 172.16.43.99:6443: no route to host
```

확인:

```bash
ssh kosa@172.16.43.100 "ip addr | grep 172.16.43.99 || true"
ssh kosa@172.16.43.101 "ip addr | grep 172.16.43.99 || true"
ssh kosa@172.16.43.102 "ip addr | grep 172.16.43.99 || true"
```

의미:

- API VIP가 어느 control plane에도 없거나
- kube-vip가 Exited 상태이거나
- VIP는 남아 있지만 kube-vip가 관리하지 않는 비정상 상태

### etcd readiness 실패

```text
[-]etcd failed
[-]etcd-readiness failed
```

Calico 설치를 진행하지 말고 먼저 control plane/etcd 상태를 확인한다.

```bash
kubectl get --raw='/readyz?verbose'
kubectl get pods -n kube-system -o wide
```

### worker의 kubectl 동작

worker 노드에는 기본적으로 `/etc/kubernetes/admin.conf`가 없다.
worker에서 `kubectl get nodes`가 안 되는 것은 정상이다.
worker는 `/etc/kubernetes/kubelet.conf`를 통해 kubelet이 API server와 통신한다.

## 최종 권장 상태

- Ansible은 kubeadm cluster bootstrap과 노드 설정까지만 안정적으로 관리한다.
- Calico는 `kubernetes/calico` manifest로 관리한다.
- 향후 ArgoCD를 도입하면 `kubernetes/calico` 디렉터리를 GitOps 대상으로 넘길 수 있다.

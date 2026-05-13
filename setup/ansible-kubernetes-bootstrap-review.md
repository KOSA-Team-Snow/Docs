# Ansible Kubernetes Bootstrap Review

## 목적

Terraform으로 Proxmox VM 생성과 Ansible inventory 생성이 완료된 이후, Ansible로 VM 내부 설정을 적용하는 과정을 정리한다.

이번 단계의 목표는 아래와 같다.

- Kubernetes 노드 공통 사전 설정 자동화
- HAProxy/Keepalived 기반 Kubernetes API Load Balancer 구성
- 첫 번째 Control Plane `kubeadm init` 흐름 정리
- Terraform, Ansible, Helm/GitOps의 역할 분리 명확화
- 이후 Control Plane/Worker join 및 CNI 설치 작업의 기준 마련

## 현재 전제

Terraform 단계에서 아래 작업은 완료된 상태로 본다.

- Proxmox cloud-init template 기반 VM 생성
- VLAN tag와 static IP 설정
- SSH key 주입
- Kubernetes Control Plane/Worker VM 생성
- Load Balancer VM 생성
- Bastion, Monitoring, ArgoCD, MariaDB VM 생성
- Ansible inventory 자동 생성

Inventory 경로:

```text
ansible/inventories/onprem/hosts.yml
```

## 역할 분리

### Terraform

Terraform은 VM 인프라 레이어를 담당한다.

- Proxmox VM clone
- VMID, 이름, 배치 노드 관리
- CPU, memory, disk 관리
- VLAN tag 설정
- static IP, gateway, DNS 설정
- SSH public key cloud-init 주입
- Ansible inventory 생성

Terraform에서 제외하는 범위:

- VM 내부 패키지 설치
- kubeadm init/join
- HAProxy/Keepalived 설정 파일 관리
- CNI 설치
- MetalLB, Ingress, ArgoCD, Monitoring stack 배포

### Ansible

Ansible은 생성된 VM 내부 설정을 담당한다.

- OS 공통 패키지 설치
- swap 비활성화
- Kubernetes kernel module 설정
- Kubernetes sysctl 설정
- containerd 설정
- kubelet/kubeadm/kubectl 설치
- HAProxy/Keepalived 설치
- Kubernetes API Load Balancer 설정
- 첫 번째 Control Plane 초기화

### Helm/GitOps

Kubernetes 내부 리소스는 Ansible보다 Helm 또는 ArgoCD 같은 GitOps 도구로 관리하는 것이 적합하다.

이후 단계에서 Helm/GitOps로 분리할 대상:

- CNI network plugin
- MetalLB
- Ingress Controller
- ArgoCD Application
- Monitoring stack
- Kubernetes application manifests

## Ansible 디렉터리 구조

```text
ansible/
├── ansible.cfg
├── inventories/
│   └── onprem/
│       └── hosts.yml
├── group_vars/
│   └── all/
│       └── k8s.yml
├── playbooks/
│   ├── k8s-prereq.yml
│   ├── lb-prereq.yml
│   ├── lb-config.yml
│   └── k8s-init.yml
└── README.md
```

## Inventory 기준

### 공통 변수

`all.vars`에는 Ansible 접속과 Kubernetes 공통 값이 들어 있다.

- `ansible_user`
- `ansible_ssh_private_key_file`
- `ansible_python_interpreter`
- `kubernetes_api_vip`

현재 Kubernetes API VIP:

```text
172.16.43.99
```

### Kubernetes 노드

| Host | IP | Role |
| --- | --- | --- |
| `k8s-cp-1` | `172.16.43.100` | Control Plane |
| `k8s-cp-2` | `172.16.43.101` | Control Plane |
| `k8s-cp-3` | `172.16.43.102` | Control Plane |
| `k8s-worker-1` | `172.16.43.110` | Worker |
| `k8s-worker-2` | `172.16.43.111` | Worker |
| `k8s-worker-3` | `172.16.43.112` | Worker |

### Load Balancer 노드

| Host | IP | Role |
| --- | --- | --- |
| `lb-1` | `172.16.42.100` | HAProxy + Keepalived |
| `lb-2` | `172.16.42.101` | HAProxy + Keepalived |

## 주요 변수

파일:

```text
ansible/group_vars/all/k8s.yml
```

현재 기준:

```yaml
kubernetes_repo_version: "v1.30"
kubernetes_package_version: "1.30.1-1.1"
```

설치 패키지:

- `kubelet`
- `kubeadm`
- `kubectl`
- `containerd`
- `qemu-guest-agent`

Kubernetes kernel module:

- `overlay`
- `br_netfilter`

Kubernetes sysctl:

- `net.bridge.bridge-nf-call-ip6tables = 1`
- `net.bridge.bridge-nf-call-iptables = 1`
- `net.ipv4.ip_forward = 1`

## Playbook 정리

### 1. `k8s-prereq.yml`

대상:

```yaml
hosts: k8s
```

목적:

Kubernetes Control Plane과 Worker 노드에 kubeadm 기반 클러스터 구성을 위한 공통 사전 설정을 적용한다.

주요 작업:

- 공통 패키지 설치
- swap 즉시 비활성화
- `/etc/fstab` swap 항목 주석 처리
- `overlay`, `br_netfilter` 모듈 영구 설정
- Kubernetes sysctl 설정 영구 적용
- containerd 기본 설정 생성
- containerd systemd cgroup driver 설정
- containerd, qemu guest agent enable/start
- Kubernetes apt key 다운로드
- Kubernetes apt repository 설정
- `kubelet`, `kubeadm`, `kubectl` 설치
- Kubernetes 패키지 hold 설정
- kubelet enable

이 playbook은 `kubeadm init`이나 `kubeadm join`을 실행하지 않는다. 노드가 Kubernetes 클러스터에 들어가기 전 OS와 런타임 상태를 맞추는 역할만 한다.

### 2. `lb-prereq.yml`

대상:

```yaml
hosts: load_balancer
```

목적:

Kubernetes API Load Balancer로 사용할 VM에 HAProxy와 Keepalived 패키지를 설치한다.

주요 작업:

- `haproxy` 설치
- `keepalived` 설치
- HAProxy service enable
- Keepalived service enable

이 playbook은 패키지 설치와 service enable까지만 담당한다. 실제 설정 파일은 `lb-config.yml`에서 관리한다.

### 3. `lb-config.yml`

대상:

```yaml
hosts: load_balancer
```

목적:

HAProxy와 Keepalived를 설정해 Kubernetes API endpoint를 VIP로 제공한다.

현재 기준:

```text
Kubernetes API VIP: 172.16.43.99
Kubernetes API Port: 6443
```

HAProxy 설정:

- TCP mode 사용
- frontend `kubernetes_api`에서 `*:6443` bind
- backend `kubernetes_control_plane`에 Control Plane 3대 등록
- round-robin load balancing
- TCP health check 사용

Keepalived 설정:

- `lb-1`을 MASTER로 설정
- `lb-2`를 BACKUP으로 설정
- HAProxy 프로세스를 health check 대상으로 사용
- VIP를 Load Balancer interface에 할당

주의:

현재 `kubernetes_api_vip`는 VLAN 30 대역인 `172.16.43.99`이다. Load Balancer VM은 VLAN 20 대역에 있으므로, 실제 네트워크에서 이 VIP를 어느 인터페이스와 라우팅 정책으로 제공할지 검증이 필요하다.

### 4. `k8s-init.yml`

대상:

```yaml
hosts: k8s-cp-1
```

목적:

첫 번째 Control Plane 노드에서 `kubeadm init`을 실행한다.

현재 기준:

```text
First Control Plane: k8s-cp-1
Kubernetes Version: v1.30.1
API Port: 6443
Pod CIDR: 10.244.0.0/16
Service CIDR: 10.96.0.0/12
Control Plane Endpoint: 172.16.43.99:6443
```

주요 작업:

- `/tmp/kubeadm-init-config.yaml` 생성
- `controlPlaneEndpoint`에 API VIP 설정
- `certSANs`에 API VIP와 Control Plane 노드 이름/IP 포함
- `kubeadm init --config ... --upload-certs` 실행
- Ansible user의 kubeconfig 디렉터리 생성
- `/etc/kubernetes/admin.conf`를 사용자 kubeconfig로 복사
- kubeadm init 결과 출력

이 playbook은 첫 번째 Control Plane 초기화만 담당한다. 추가 Control Plane join과 Worker join은 아직 포함되어 있지 않다.

## 실행 순서

Terraform으로 VM 생성이 완료된 후 Ansible 디렉터리에서 실행한다.

```bash
cd ansible
```

1. 전체 SSH 접속 확인

```bash
ansible -m ping all
```

2. Kubernetes 노드 사전 설정

```bash
ansible-playbook playbooks/k8s-prereq.yml
```

3. Load Balancer 패키지 설치

```bash
ansible-playbook playbooks/lb-prereq.yml
```

4. Load Balancer 설정 적용

```bash
ansible-playbook playbooks/lb-config.yml
```

5. 첫 번째 Control Plane 초기화

```bash
ansible-playbook playbooks/k8s-init.yml
```

## 검증 명령어

### 전체 접속 확인

```bash
ansible -m ping all
```

### Kubernetes 노드 확인

```bash
ansible k8s -m shell -a "containerd --version"
ansible k8s -m shell -a "kubeadm version"
ansible k8s -m shell -a "kubectl version --client"
ansible k8s -m shell -a "systemctl is-enabled containerd"
ansible k8s -m shell -a "systemctl is-enabled kubelet"
```

### Load Balancer 확인

```bash
ansible load_balancer -m shell -a "systemctl status haproxy --no-pager"
ansible load_balancer -m shell -a "systemctl status keepalived --no-pager"
ansible load_balancer -m shell -a "ip addr"
```

### Kubernetes 초기화 확인

`k8s-cp-1`에서 확인한다.

```bash
kubectl get nodes
kubectl get pods -A
```

주의:

CNI가 아직 설치되지 않았다면 노드는 `NotReady` 상태일 수 있다. 이는 정상적인 다음 단계 작업이다.

## 현재 Ansible에서 제외된 작업

현재 Ansible 파일 기준으로 아직 자동화되지 않은 작업은 아래와 같다.

- 추가 Control Plane join
- Worker node join
- CNI network plugin 설치
- MetalLB 배포
- Ingress Controller 배포
- ArgoCD Application 배포
- Monitoring stack 배포
- MariaDB 설정
- Kubernetes node label/taint 적용

## 다음 작업 제안

### 1. Join 자동화

`k8s-init.yml` 이후 아래 작업이 필요하다.

- `kubeadm token create --print-join-command`
- certificate key 재사용 또는 재생성 방식 결정
- `k8s-cp-2`, `k8s-cp-3` Control Plane join
- `k8s-worker-1`, `k8s-worker-2`, `k8s-worker-3` Worker join

### 2. CNI 설치

Kubernetes 클러스터는 CNI가 설치되기 전까지 정상적인 `Ready` 상태가 되지 않는다.

CNI는 Ansible로 직접 manifest를 관리하기보다 Helm 또는 ArgoCD로 관리하는 것을 권장한다. 다만 bootstrap 단계에서는 Ansible이 Helm install을 한 번 호출하는 방식도 가능하다.

후보:

- Cilium
- Calico
- Flannel

### 3. GitOps 분리

클러스터 내부 애플리케이션 성격의 리소스는 Helm 또는 ArgoCD로 관리한다.

대상:

- CNI
- MetalLB
- Ingress Controller
- ArgoCD Application
- Monitoring stack
- 애플리케이션 manifest

## 완료 기준

이 문서 기준으로 다음 상태가 확인되면 Day 4 Ansible bootstrap 문서화가 완료된 것으로 본다.

- Terraform VM 생성 이후 Ansible 실행 순서가 정리되어 있다.
- Kubernetes 노드 사전 설정 작업이 설명되어 있다.
- HAProxy/Keepalived 설치와 API VIP 구성 흐름이 설명되어 있다.
- 첫 번째 Control Plane `kubeadm init` 흐름이 설명되어 있다.
- 현재 Ansible이 담당하지 않는 작업이 명확히 분리되어 있다.
- 다음 단계인 join, CNI, Helm/GitOps 작업 범위가 정리되어 있다.

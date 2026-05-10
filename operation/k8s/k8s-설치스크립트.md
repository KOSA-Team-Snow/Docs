# Day 1 - 2.kubeadm 설치 스크립트 초안 작성

## 1. 작업 목적

온프레미스 Kubernetes 클러스터를 `kubeadm` 기반으로 설치하기 전에, 각 노드에서 반복 실행해야 하는 사전 설정과 패키지 설치 절차를 스크립트 초안으로 정리한다.

이번 단계의 목표는 최종 자동화가 아니라, Control Plane과 Worker Node에 공통으로 필요한 설치 흐름을 명확히 만들고 Day 4의 Ansible 자동화로 전환할 수 있는 기준을 마련하는 것이다.

## 2. 작업 범위

| 항목 | 내용 |
| --- | --- |
| 대상 환경 | Proxmox VM 기반 Ubuntu 노드 |
| 설치 방식 | kubeadm |
| 컨테이너 런타임 | containerd |
| 주요 도구 | kubeadm, kubelet, kubectl |
| 자동화 전환 대상 | Ansible playbook |

## 3. 스크립트 초안에 포함할 내용

### 3.1 OS 사전 설정

Kubernetes 설치 전 각 노드에서 필요한 기본 설정을 정리한다.

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

확인할 내용:

- swap 비활성화 여부
- 커널 모듈 로드 여부
- sysctl 네트워크 설정 적용 여부

### 3.2 커널 모듈 및 네트워크 설정

Kubernetes 네트워크와 containerd 동작에 필요한 설정을 적용한다.

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

### 3.3 containerd 설치 및 설정

Kubernetes의 컨테이너 런타임으로 `containerd`를 설치하고 systemd cgroup 설정을 적용한다.

```bash
sudo apt-get update
sudo apt-get install -y containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

### 3.4 kubeadm, kubelet, kubectl 설치

Kubernetes 클러스터 구성에 필요한 기본 패키지를 설치한다.

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## 4. Control Plane 초기화 초안

첫 번째 Control Plane 노드에서 클러스터를 초기화한다.

```bash
sudo kubeadm init \
  --control-plane-endpoint "<LOAD_BALANCER_DNS_OR_IP>:6443" \
  --upload-certs \
  --pod-network-cidr "10.244.0.0/16"
```

초기화 후 일반 사용자 계정에서 `kubectl`을 사용할 수 있도록 kubeconfig를 설정한다.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 5. Worker Node Join 초안

Control Plane 초기화 후 출력되는 join 명령어를 Worker Node에서 실행한다.

```bash
sudo kubeadm join <CONTROL_PLANE_ENDPOINT>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

HA Control Plane을 추가할 경우에는 `--control-plane`과 인증서 키 옵션을 포함한다.

```bash
sudo kubeadm join <CONTROL_PLANE_ENDPOINT>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --control-plane \
  --certificate-key <CERTIFICATE_KEY>
```

## 6. 검증 명령어

설치 후 다음 명령어로 노드와 핵심 구성 요소 상태를 확인한다.

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl cluster-info
systemctl status kubelet
systemctl status containerd
```

## 7. Ansible 전환 시 분리 기준

Day 4에서 Ansible 자동화로 전환할 때는 다음 역할로 나누는 것을 기준으로 한다.

| Role | 담당 내용 |
| --- | --- |
| common | swap 비활성화, 커널 모듈, sysctl 설정 |
| containerd | containerd 설치 및 cgroup 설정 |
| kubernetes | kubeadm, kubelet, kubectl 설치 |
| control-plane | kubeadm init 및 kubeconfig 설정 |
| worker | kubeadm join 실행 |

## 8. 완료 기준

- Kubernetes 설치 전 사전 설정 항목이 정리되어 있다.
- containerd 설치 및 systemd cgroup 설정이 포함되어 있다.
- kubeadm, kubelet, kubectl 설치 절차가 정리되어 있다.
- Control Plane 초기화와 Worker Node Join 명령어 초안이 포함되어 있다.
- Day 4 Ansible 자동화로 전환할 수 있는 분리 기준이 정리되어 있다.

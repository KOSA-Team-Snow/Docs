# Kubernetes-ready VM 템플릿 생성 및 Ceph RBD 저장 기준

## 1. 작업 목적

Proxmox VE 클러스터 환경에서 Ubuntu 24.04 기반 Kubernetes-ready VM 템플릿을 생성한다.

이 템플릿은 Kubernetes가 이미 설치된 템플릿이 아니라, Kubernetes 노드로 사용하기 위한 OS 공통 준비가 완료된 템플릿이다. 이후 Terraform에서 템플릿을 clone하면서 VM별 IP, hostname, SSH public key를 Cloud-Init으로 주입하고, kubeadm, kubelet, kubectl 설치 및 클러스터 구성 작업을 수행한다.

## 2. 기본 기준

| 항목                    | 기준                          |
| ----------------------- | ----------------------------- |
| OS                      | Ubuntu 24.04                  |
| 가상화 플랫폼           | Proxmox VE                    |
| VM 디스크 저장소        | Ceph RBD                      |
| ISO 저장소              | Proxmox local storage         |
| 네트워크                | VLAN 30 내부망                |
| 템플릿 용도             | Kubernetes 노드 공통 베이스   |
| Kubernetes 설치         | 템플릿에는 미포함             |
| IP / hostname / SSH key | Terraform Cloud-Init에서 주입 |

## 3. Ubuntu 24.04 ISO 업로드

작업 PC에 있는 Ubuntu 24.04 ISO 파일을 Proxmox GUI를 통해 업로드한다.

Proxmox GUI에서 다음 경로로 이동한다.

```text
pve-1 -> local (pve-1) -> ISO Images -> Upload
```

업로드할 ISO 예시는 다음과 같다.

```text
Ubuntu 24.04.3 Live Server.iso
```

ISO 파일은 설치 미디어로만 사용한다. VM 설치가 끝난 뒤에는 CD/DVD Drive에서 ISO 참조를 제거해야 한다.

## 4. 설치 VM 생성

Proxmox GUI에서 VM을 생성한다.

```text
Create VM
```

기본 설정 예시는 다음과 같다.

| 항목            | 값                             |
| --------------- | ------------------------------ |
| VM ID           | 9000                           |
| Name            | ubuntu-2404-k8s-ready-template |
| OS              | Ubuntu 24.04 ISO               |
| System          | QEMU Guest Agent 활성화        |
| SCSI Controller | VirtIO SCSI                    |
| Disk Storage    | Ceph RBD                       |
| Disk Size       | 40GB 이상                      |
| CPU             | 2 cores                        |
| Memory          | 4096MB                         |
| Network Bridge  | vmbr0                          |
| Network Model   | VirtIO                         |
| VLAN Tag        | 30                             |

VM 디스크는 반드시 Ceph RBD 스토리지에 생성한다.

예시:

```text
Hard Disk (scsi0): ceph-rbd:vm-9000-disk-0
```

반대로 다음과 같이 local storage에 생성되면 다른 Proxmox 노드에서 clone 또는 migration 시 제약이 생길 수 있다.

```text
local-lvm:vm-9000-disk-0
local:9000/vm-9000-disk-0.qcow2
```

## 5. Ubuntu 설치

VM을 시작한 뒤 Ubuntu 24.04를 설치한다.

설치 과정에서 네트워크는 VLAN 30 내부망 기준으로 설정한다. 설치 중 임시 IP를 사용할 수 있지만, 템플릿에는 최종 고정 IP를 남기지 않는다.

Kubernetes 노드의 최종 IP는 Terraform에서 VM별로 Cloud-Init을 통해 주입한다.

## 6. CloudInit Drive 추가

Ubuntu 설치가 끝난 뒤 Proxmox GUI에서 CloudInit Drive를 추가한다.

```text
VM -> Hardware -> Add -> CloudInit Drive
```

CloudInit Drive도 Ceph RBD에 생성한다.

```text
Storage: ceph-rbd
```

정상 예시는 다음과 같다.

```text
CloudInit Drive: ceph-rbd:vm-9000-cloudinit
```

CloudInit Drive는 이후 Terraform이 VM별 사용자, SSH public key, IP, hostname을 주입하는 데 사용한다.

## 7. CD/DVD Drive에서 ISO 제거

Ubuntu 설치가 끝나면 CD/DVD Drive에서 local ISO 참조를 제거한다.

Proxmox GUI에서 다음 항목을 확인한다.

```text
VM -> Hardware -> CD/DVD Drive
```

설치 ISO가 다음처럼 연결되어 있으면 제거한다.

```text
local:iso/Ubuntu 24.04.3 Live Server.iso
```

설치 후 ISO 참조를 제거해야 하는 이유는 다음과 같다.

- ISO는 pve-1의 local storage에만 존재할 수 있다.
- 템플릿이나 clone VM이 local ISO를 계속 참조하면 다른 노드에서 clone 또는 migration 시 문제가 생길 수 있다.
- VM의 실제 부팅 디스크는 Ceph RBD에 있어야 하며, ISO는 설치 후 더 이상 필요하지 않다.

설치 완료 후에는 CD/DVD Drive를 다음 상태로 둔다.

```text
Do not use any media
```

또는 CD/DVD Drive를 제거한다.

## 8. Kubernetes-ready 템플릿 공통 패키지

템플릿에는 Kubernetes 자체를 설치하지 않는다.

포함할 항목은 Kubernetes 설치 전 OS 공통 준비에 해당하는 패키지와 설정이다.

설치할 공통 패키지는 다음과 같다.

```bash
sudo apt update

sudo apt install -y \
  qemu-guest-agent \
  cloud-init \
  chrony \
  curl \
  wget \
  vim \
  git \
  net-tools \
  dnsutils \
  iputils-ping \
  traceroute \
  ca-certificates \
  gnupg \
  lsb-release \
  software-properties-common \
  apt-transport-https \
  containerd \
  socat \
  conntrack \
  ethtool \
  ipset \
  ipvsadm
```

각 주요 패키지의 용도는 다음과 같다.

| 패키지                           | 용도                                        |
| -------------------------------- | ------------------------------------------- |
| qemu-guest-agent                 | Proxmox에서 VM IP와 상태 확인               |
| cloud-init                       | clone 후 사용자, SSH key, IP, hostname 주입 |
| chrony                           | 시간 동기화                                 |
| containerd                       | Kubernetes에서 사용할 컨테이너 런타임       |
| socat, conntrack, ipset, ipvsadm | Kubernetes 네트워크 구성에 필요한 도구      |
| curl, wget, vim, git             | 기본 운영 도구                              |
| dnsutils, net-tools, traceroute  | 네트워크 진단 도구                          |

## 9. 서비스 활성화

설치 후 주요 서비스를 활성화한다.

```bash
sudo systemctl enable --now chrony
sudo systemctl enable --now containerd
```

qemu-guest-agent는 Proxmox VM 옵션에서도 활성화되어 있어야 한다.

Proxmox GUI에서 확인한다.

```text
VM -> Options -> QEMU Guest Agent -> Enabled
```

VM 내부에서는 상태를 확인한다.

```bash
systemctl status qemu-guest-agent --no-pager
```

qemu-guest-agent가 바로 active 상태가 아니더라도, Proxmox의 QEMU Guest Agent 옵션을 활성화하고 VM을 재부팅하면 정상 동작하는 경우가 많다.

## 10. Kubernetes 커널 모듈 설정

Kubernetes Pod 네트워크와 브릿지 트래픽 처리를 위해 다음 커널 모듈을 사용한다.

- overlay
- br_netfilter

부팅 시 자동으로 로드되도록 설정한다.

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

현재 세션에도 즉시 적용한다.

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

확인한다.

```bash
lsmod | grep overlay
lsmod | grep br_netfilter
cat /etc/modules-load.d/k8s.conf
```

## 11. Kubernetes 네트워크 sysctl 설정

Kubernetes 네트워크가 정상 동작하려면 브릿지 트래픽이 iptables에서 처리되어야 하며, 노드에서 IP forwarding이 가능해야 한다.

다음 설정 파일을 생성한다.

```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

설정을 적용한다.

```bash
sudo sysctl --system
```

확인한다.

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.ipv4.ip_forward
```

정상 값은 다음과 같다.

```text
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

## 12. swap 비활성화

Kubernetes 노드에서는 swap을 비활성화한다.

swap은 디스크 일부를 메모리처럼 사용하는 기능이다. 일반 서버에서는 도움이 될 수 있지만, Kubernetes에서는 Pod 메모리 제어와 kubelet 동작에 영향을 줄 수 있다.

기본 kubelet 설정에서는 swap이 켜져 있으면 kubelet이 정상 동작하지 않을 수 있다.

현재 swap을 끈다.

```bash
sudo swapoff -a
```

재부팅 후에도 swap이 켜지지 않도록 `/etc/fstab`에서 swap 항목을 주석 처리한다.

Ubuntu 설치 VM에서는 보통 `/swap.img`가 등록되어 있다.

```bash
sudo sed -i '/\/swap.img/ s/^/#/' /etc/fstab
```

확인한다.

```bash
free -h
swapon --show
grep swap /etc/fstab
```

정상 상태는 다음과 같다.

```text
Swap: 0B
```

그리고 `swapon --show`는 출력이 없어야 한다.

## 13. containerd SystemdCgroup 설정

containerd는 Kubernetes에서 사용할 컨테이너 런타임이다.

먼저 설정 파일을 생성한다.

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```

Kubernetes와 Ubuntu systemd 환경에 맞게 `SystemdCgroup` 값을 `true`로 변경한다.

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

containerd를 재시작한다.

```bash
sudo systemctl restart containerd
```

확인한다.

```bash
grep SystemdCgroup /etc/containerd/config.toml
systemctl is-active containerd
```

정상 값은 다음과 같다.

```text
SystemdCgroup = true
active
```

## 14. 템플릿에 고정하지 않는 항목

템플릿에는 다음 값을 고정하지 않는다.

- SSH public key
- 고정 IP
- hostname
- kubeadm init
- kubeadm join
- kubelet
- kubeadm
- kubectl
- CNI
- 노드별 인증서
- 애플리케이션 코드

이 값들은 VM별로 달라져야 하므로 Terraform Cloud-Init 또는 이후 자동화 도구에서 주입한다.

Terraform에서 VM을 clone할 때 다음과 같은 값을 설정한다.

```hcl
ciuser  = "kosa"
sshkeys = file("~/.ssh/id_ed25519.pub")

ipconfig0 = "ip=172.16.43.100/24,gw=172.16.43.1"

network {
  model  = "virtio"
  bridge = "vmbr0"
  tag    = 30
}
```

템플릿은 공통 OS 준비만 담당하고, Kubernetes 설치와 클러스터 구성은 Terraform 또는 Ansible 단계에서 수행한다.

## 15. 최종 검증

템플릿으로 변환하기 전 VM 내부에서 다음 항목을 확인한다.

```bash
systemctl is-active chrony
systemctl is-active containerd
grep SystemdCgroup /etc/containerd/config.toml

lsmod | grep overlay
lsmod | grep br_netfilter

sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.ipv4.ip_forward

free -h
swapon --show
```

정상 기준은 다음과 같다.

| 항목                     | 정상 기준 |
| ------------------------ | --------- |
| chrony                   | active    |
| containerd               | active    |
| SystemdCgroup            | true      |
| overlay                  | 로드됨    |
| br_netfilter             | 로드됨    |
| bridge-nf-call-iptables  | 1         |
| bridge-nf-call-ip6tables | 1         |
| ip_forward               | 1         |
| swap                     | 0B        |
| swapon --show            | 출력 없음 |

네트워크도 확인한다.

```bash
ip a
ip route
ping -c 3 172.16.43.1
ping -c 3 8.8.8.8
ping -c 3 archive.ubuntu.com
```

단, 최종 clone VM에서는 Terraform이 고정 IP를 주입하므로 템플릿 자체에는 최종 IP를 남기지 않는다.

## 16. 템플릿 변환 전 정리

템플릿 변환 직전에 cloud-init, machine-id, SSH host key를 정리한다.

cloud-init 상태를 초기화한다.

```bash
sudo cloud-init clean
```

machine-id를 비운다.

```bash
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id
```

SSH host key를 삭제한다.

```bash
sudo rm -f /etc/ssh/ssh_host_*
```

apt 캐시를 정리한다.

```bash
sudo apt clean
```

정리 후에는 바로 VM을 종료한다.

```bash
sudo shutdown -h now
```

정리 후 다시 부팅하면 일부 값이 다시 생성될 수 있으므로, 종료 후 바로 Proxmox GUI에서 템플릿으로 변환한다.

## 17. Proxmox 템플릿 변환

VM이 종료된 뒤 Proxmox GUI에서 템플릿으로 변환한다.

```text
VM 우클릭 -> Convert to template
```

템플릿 이름 예시는 다음과 같다.

```text
ubuntu-2404-k8s-ready-template
```

## 18. Ceph RBD 저장 상태 확인

템플릿의 디스크가 Ceph RBD에 저장되어 있는지 확인한다.

Proxmox GUI에서 템플릿 VM의 Hardware를 확인한다.

정상 예시는 다음과 같다.

```text
Hard Disk (scsi0): ceph-rbd:vm-9000-disk-0
CloudInit Drive: ceph-rbd:vm-9000-cloudinit
```

다음과 같이 local storage를 참조하면 안 된다.

```text
Hard Disk: local-lvm:...
CD/DVD Drive: local:iso/Ubuntu...
```

ISO는 설치 중에는 local storage에 있어도 되지만, 설치 후에는 VM에서 참조하지 않아야 한다.

## 19. 다른 Proxmox 노드로 Full Clone 검증

템플릿이 Ceph RBD에 저장되어 있으면 pve-1에 표시된 템플릿이라도 pve-2, pve-3, pve-4, pve-5로 clone할 수 있어야 한다.

검증 방법은 다음과 같다.

```text
템플릿 우클릭 -> Clone
```

Clone 설정 예시는 다음과 같다.

| 항목           | 값                                 |
| -------------- | ---------------------------------- |
| Mode           | Full Clone                         |
| Target node    | pve-2, pve-3, pve-4, pve-5 중 선택 |
| Target Storage | ceph-rbd                           |

Full Clone을 사용하는 이유는 템플릿 원본 디스크와 clone VM 디스크의 의존성을 줄이고, 각 VM을 독립적으로 운영하기 위해서이다.

clone이 되지 않는 경우 다음 항목을 확인한다.

- Hard Disk가 Ceph RBD에 있는가
- CloudInit Drive가 Ceph RBD에 있는가
- CD/DVD Drive가 local ISO를 참조하고 있지 않은가
- Datacenter Storage에서 Ceph RBD가 모든 노드에서 사용 가능하도록 설정되어 있는가
- Clone Mode가 Full Clone인가

## 20. 최종 운영 흐름

최종 흐름은 다음과 같다.

```text
Ubuntu 24.04 ISO local 업로드
-> 설치 VM 생성
-> VM 디스크를 Ceph RBD에 생성
-> Ubuntu 설치
-> CloudInit Drive를 Ceph RBD에 추가
-> 설치 후 local ISO 참조 제거
-> 공통 패키지 설치
-> qemu-guest-agent, chrony, containerd 준비
-> Kubernetes용 커널 모듈/sysctl/swap/containerd 설정
-> cloud-init, machine-id, SSH host key 정리
-> shutdown
-> Convert to template
-> Terraform으로 VM별 Full Clone
-> Terraform Cloud-Init으로 IP, hostname, SSH key 주입
-> Terraform/Ansible에서 Kubernetes 설치 및 클러스터 구성
```

## 21. 템플릿에 포함할 항목과 제외할 항목

### 포함할 항목

- Ubuntu 24.04
- qemu-guest-agent
- cloud-init
- chrony
- containerd
- 기본 운영 도구
- Kubernetes 네트워크용 커널 모듈 설정
- Kubernetes 네트워크용 sysctl 설정
- swap off
- containerd SystemdCgroup 설정

### 제외할 항목

- kubeadm
- kubelet
- kubectl
- kubeadm init
- kubeadm join
- CNI
- SSH public key
- 고정 IP
- hostname
- 노드별 인증서
- 애플리케이션 설정

## 22. 정리

이 템플릿은 Kubernetes가 설치된 템플릿이 아니라 Kubernetes 설치 준비가 완료된 VM 템플릿이다.

VM 디스크와 CloudInit Drive는 Ceph RBD에 저장하고, ISO는 설치 후 참조를 제거한다. 템플릿에는 공통 OS 설정만 포함하며, VM별 IP, hostname, SSH public key와 Kubernetes 설치 작업은 Terraform 또는 Ansible 단계에서 처리한다.

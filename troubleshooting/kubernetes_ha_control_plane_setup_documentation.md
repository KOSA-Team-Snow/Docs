# Kubernetes HA Control Plane 구축 과정 정리 & 발생 오류 정리

## 1. 목표 아키텍처

구성 목표:

- Kubernetes HA Control Plane 구성
- kube-vip 기반 API VIP 구성
- Control Plane 3대 구성
- kubeadm 기반 클러스터 초기화

구성 노드:

| 역할 | 호스트명 | IP |
|---|---|---|
| Control Plane 1 | k8s-cp-1 | 172.16.43.100 |
| Control Plane 2 | k8s-cp-2 | 172.16.43.101 |
| Control Plane 3 | k8s-cp-3 | 172.16.43.102 |
| Kubernetes API VIP | kube-vip | 172.16.43.99 |

---

# 2. kube-vip 선택 이유

초기에는 HAProxy + Keepalived 구조를 고려하였다.

하지만:

- 별도 LB VM 관리 필요
- Keepalived VIP 관리 복잡성
- 설계 변경 부담

등의 이유로 kube-vip를 사용하기로 결정하였다.

kube-vip는:

- Kubernetes Control Plane 내부에서 동작
- VIP를 자동 관리
- 별도 외부 LB 없이 API HA 구성 가능

하다는 장점이 있다.

---

# 3. kubeadm init 설정

`kubeadm-init-config.yaml`

```yaml
controlPlaneEndpoint: "172.16.43.99:6443"
```

설정을 통해 모든 Control Plane 노드가 API VIP를 통해 통신하도록 구성하였다.

Pod CIDR:

```yaml
10.244.0.0/16
```

Service CIDR:

```yaml
10.96.0.0/12
```

---

# 4. kube-vip Static Pod 구성

`/etc/kubernetes/manifests/kube-vip.yaml`

형태의 Static Pod로 kube-vip를 배포하였다.

핵심 설정:

```yaml
hostNetwork: true
```

```yaml
vip_interface: eth0
```

```yaml
address: 172.16.43.99
```

```yaml
cp_enable: true
```

---

# 5. 발생한 주요 오류 및 해결 과정

## (1) kube-vip manifest 미적용 문제

### 증상

```bash
curl https://172.16.43.99:6443
```

실패

### 원인

Ansible inventory 경로 오류로 인해:

- kube-vip playbook이 실행되지 않음
- `/etc/kubernetes/manifests/kube-vip.yaml` 생성 실패

### 해결

- inventory 경로 수정
- kube-vip manifest 직접 배포

---

## (2) kube-vip YAML 파싱 오류

### 증상

```text
couldn't parse as pod
```

### 원인

Ansible Playbook 전체 YAML을:

- Kubernetes Static Pod YAML로 잘못 복사함

### 해결

Static Pod 형태:

```yaml
apiVersion: v1
kind: Pod
```

로 수정하여 재배포하였다.

---

## (3) kube-vip CrashLoopBackOff

### 증상

```text
kube-vip Exited 반복
```

### 원인

kube-vip가 Kubernetes API 인증 파일을 찾지 못함.

```text
/.kube/config
```

경로를 찾으려 했음.

### 해결

`super-admin.conf`를 mount하고:

```yaml
--k8sConfigPath=/etc/kubernetes/super-admin.conf
```

옵션으로 kubeconfig 경로를 지정하였다.

---

## (4) kube-vip CIDR 오류

### 증상

```text
invalid CIDR address: 172.16.43.9932
```

### 원인

```yaml
vip_subnet: "32"
```

설정 오류.

### 해결

```yaml
vip_subnet: "/32"
```

로 수정하였다.

---

## (5) etcd learner 오류

### 증상

```text
too many learner members in cluster
```

### 원인

Control Plane Join 실패 후:

- etcd learner member가 클러스터에 남아있었음.

### 해결

- kubeadm reset 수행
- 실패한 etcd member 제거
- 순차적으로 cp join 수행

---

## (6) API Server 응답 지연 문제

### 증상

```text
failed to request cluster-info ConfigMap
```

### 원인

잘못된 token 사용.

예시 token:

```text
abcdef.1234567890abcdef
```

을 실제 token 대신 사용함.

### 해결

새 bootstrap token 생성:

```bash
kubeadm token create --print-join-command
```

후 재조인하였다.

---

# 6. Control Plane Join 과정

## cp2 Join

```bash
sudo kubeadm join 172.16.43.99:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --control-plane \
  --certificate-key <CERT_KEY> \
  --apiserver-advertise-address 172.16.43.101
```

## cp3 Join

```bash
sudo kubeadm join 172.16.43.99:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --control-plane \
  --certificate-key <CERT_KEY> \
  --apiserver-advertise-address 172.16.43.102
```

---

# 7. 추가 이슈 및 해결 과정

## \(7\) cp2/cp3에서 kube-vip 미동작 문제

### 증상

cp1 종료 시:

```text
VIP가 cp2/cp3로 이동하지 않음
```

또한 아래 명령어 실행 시:

```bash
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps -a | grep kube-vip
```

cp2/cp3에서 아무 출력도 나타나지 않았다.

---

### 원인 분석

초기에는:

```text
leader election 문제
```

를 의심하였으나, 실제 원인은:

```text
cp2/cp3에 kube-vip static pod가 정상 생성되지 않음
```

이었다.

추가 로그 분석 결과:

```text
hostPath type check failed: /etc/kubernetes/super-admin.conf is not a file
```

오류가 발생하였다.

즉:

- cp1의 kube-vip.yaml을 그대로 복사함
- kube-vip가 `/etc/kubernetes/super-admin.conf` 를 mount 하도록 설정되어 있었음
- 하지만 cp2/cp3에는 해당 파일이 존재하지 않았음
- kubelet이 volume mount에 실패하면서 kube-vip static pod 생성 실패

상태였다.

---

### 해결 과정

cp2/cp3의 kube-vip.yaml에서:

```yaml
/etc/kubernetes/super-admin.conf
```

를:

```yaml
/etc/kubernetes/admin.conf
```

로 수정하였다.

수정 명령:

```bash
sudo sed -i 's#/etc/kubernetes/super-admin.conf#/etc/kubernetes/admin.conf#g' /etc/kubernetes/manifests/kube-vip.yaml
```

이후 kubelet 재시작:

```bash
sudo systemctl restart kubelet
```

을 수행하였다.

그 결과:

```bash
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps -a | grep kube-vip
```

명령어에서 kube-vip가 정상 Running 상태로 생성되었고,

cp1 shutdown 시:

```text
172.16.43.99 VIP가 cp2/cp3로 정상 이동
```

하는 것을 확인하였다.

---

## \(8\) kubectl localhost:8080 오류

### 증상

cp2/cp3에서:

```bash
kubectl get nodes
```

실행 시:

```text
The connection to the server localhost:8080 was refused
```

오류 발생.

---

### 원인

현재 사용자 홈 디렉토리에:

```text
~/.kube/config
```

파일이 존재하지 않아 kubectl이 기본값인:

```text
localhost:8080
```

으로 접속을 시도함.

---

### 해결

아래 명령어를 통해 사용자 kubeconfig를 생성하였다.

```bash
mkdir -p ~/.kube
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
```

이후 kubectl 명령이 정상 동작하였다.

---

# 8. 현재 상태

현재:

- kube-vip 정상 동작
- API VIP 정상 응답
- Control Plane 3대 join 성공

확인 명령어:

```bash
kubectl get nodes
```

다음 단계:

- CNI(Flannel/Calico) 설치
- Worker Node Join
- MetalLB 설치
- ingress-nginx 구성
- 애플리케이션 배포


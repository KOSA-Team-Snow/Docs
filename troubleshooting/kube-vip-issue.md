# Kubernetes kube-vip 설정 중 발생한 문제 정리 

## etcd 확인

### etcd 컨테이너 ID 확인

`k8s-cp-1`에서:

```bash
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps | grep etcd
```

결과:

```text
9c405e451cb89 ... Running etcd ... etcd-k8s-cp-1
```

### etcd endpoint status

```bash
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock exec -it 9c405e451cb89 etcdctl \
  --endpoints=https://172.16.43.100:2379,https://172.16.43.101:2379,https://172.16.43.102:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=table
```

결과:

```text
172.16.43.100:2379 정상
172.16.43.101:2379 정상
172.16.43.102:2379 정상, leader
```

### etcd endpoint health

```bash
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock exec -it 9c405e451cb89 etcdctl \
  --endpoints=https://172.16.43.100:2379,https://172.16.43.101:2379,https://172.16.43.102:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health
```

결과:

```text
https://172.16.43.102:2379 is healthy
https://172.16.43.100:2379 is healthy
https://172.16.43.101:2379 is healthy
```

다만 `172.16.43.101` 응답 시간이 약 `4.7s`로 다른 노드보다 느렸다.

## 주의할 점

### etcd 컨테이너 ID는 노드마다 다르다

`9c405e451cb89`는 `k8s-cp-1`의 etcd 컨테이너 ID다.

`k8s-cp-2`에서 같은 ID로 실행하면 다음 에러가 난다.

```text
failed to find container "9c405e451cb89" in store: not found
```

각 노드에서는 먼저 해당 노드의 etcd 컨테이너 ID를 확인해야 한다.

```bash
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps | grep etcd
```

또는:

```bash
ETCD_ID=$(sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps --name etcd -q)
```

## 최종 결론

이번 문제의 핵심 원인은 kube-vip static pod가 Lease를 조회할 때 `https://kubernetes:6443` 경로에서 timeout이 발생한 것이다.

해결은 static pod manifest에 다음 설정을 추가하는 것이었다.

```yaml
hostAliases:
  - ip: 127.0.0.1
    hostnames:
      - kubernetes
```

해결 후:

```text
kube-vip Pod 3개 Running
VIP 172.16.43.99가 k8s-cp-3에 부착
API VIP로 /readyz 응답 확인
etcd 3개 endpoint 모두 healthy
```

남은 관찰 포인트:

```text
k8s-cp-2의 etcd 응답 시간이 다른 노드보다 느렸음
반복되면 cp-2의 네트워크, 디스크 IO, CPU 상태를 확인해야 함
```

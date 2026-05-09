# Day 1 - 3.Kubernetes 구성 요소 선정

## 1. 작업 목적

팀원 B의 Kubernetes Platform 역할에서 사용할 주요 구성 요소를 정리한다.

이번 프로젝트는 온프레미스 Kubernetes 위에 FlaskApp, MariaDB, ArgoCD, Prometheus/Grafana를 배포하고, 이후 AWS EKS 기반 DR 환경과 비교하는 구조이다. 따라서 Day 1에서는 Kubernetes 플랫폼을 구성하는 핵심 도구를 먼저 선정하고, 각 팀원이 참고할 수 있도록 역할과 선택 이유를 문서화한다.

## 2. 구성 요소 결정 요약

| 구분 | 선택 구성 요소 | 사용 목적 |
| --- | --- | --- |
| Kubernetes 설치 | kubeadm | 온프레미스 Kubernetes 클러스터 직접 구성 |
| 컨테이너 런타임 | containerd | Pod 컨테이너 실행 런타임 |
| CNI | Calico 우선 검토, Flannel 대안 | Pod 네트워크 구성 |
| LoadBalancer | MetalLB | 온프레미스 환경에서 LoadBalancer 타입 Service IP 할당 |
| Ingress | Nginx Ingress Controller | HTTP/HTTPS 외부 접근 경로 구성 |
| GitOps | ArgoCD | Git 기준 Kubernetes 배포 자동화 |
| 패키징 | Helm | FlaskApp, monitoring 등 배포 템플릿 관리 |
| 모니터링 | Prometheus + Grafana | Node, Pod, App 상태 수집 및 시각화 |
| 테스트 Pod | BusyBox | DNS, Service, DB 연결 테스트 |
| 운영 정책 | requests/limits, HPA, taints/tolerations | 리소스 관리, 자동 확장, 인프라 Pod 격리 |

## 3. 구성 요소별 선정 내용

### 3.1 containerd

컨테이너 런타임은 `containerd`로 선정한다.

`containerd`는 Kubernetes에서 널리 사용하는 표준 런타임이며, kubeadm 기반 설치 절차와 잘 맞는다. Docker 자체를 런타임으로 사용하는 방식보다 Kubernetes 구성 설명이 명확하고, 운영 환경에서도 일반적으로 사용된다.

확인할 항목:

- `SystemdCgroup = true` 설정
- `containerd` 서비스 enable 및 restart
- 모든 Kubernetes 노드에 동일하게 설치

### 3.2 CNI

CNI는 `Calico`를 우선 후보로 정리한다. 단, 네트워크 구성이 단순하고 빠른 구축이 더 중요할 경우 `Flannel`을 대안으로 둔다.

| 후보 | 장점 | 검토 결과 |
| --- | --- | --- |
| Calico | NetworkPolicy 지원, 운영 기능 풍부, EKS와 비교 설명에 유리 | 우선 후보 |
| Flannel | 설치가 단순하고 학습 부담이 낮음 | 대안 후보 |

Day 2에서 팀원 A의 IP/VLAN 계획과 Pod CIDR 계획을 확인한 뒤 최종 선택한다.

### 3.3 MetalLB

온프레미스 Kubernetes에는 AWS ELB 같은 클라우드 LoadBalancer가 없기 때문에 `MetalLB`를 사용한다.

MetalLB는 `LoadBalancer` 타입 Service에 외부 접근용 IP를 할당한다. 이번 프로젝트에서는 ArgoCD, Grafana, Ingress Controller 등의 외부 접근 경로를 구성할 때 필요하다.

팀원 A와 맞춰야 할 항목:

- MetalLB IP pool 범위
- IP pool이 기존 VLAN/IP 대역과 충돌하지 않는지 확인
- L2 모드 사용 여부

### 3.4 Nginx Ingress Controller

Ingress Controller는 `Nginx Ingress Controller`로 선정한다.

FlaskApp, ArgoCD, Grafana 등 HTTP 기반 서비스의 외부 접근 경로를 관리하기 위해 사용한다. MetalLB가 외부 IP를 제공하고, Ingress Controller가 도메인/경로 기반 라우팅을 담당하는 구조로 잡는다.

예상 접근 구조:

```text
User
  -> MetalLB External IP
  -> Nginx Ingress Controller
  -> Kubernetes Service
  -> Pod
```

### 3.5 ArgoCD

GitOps 도구는 `ArgoCD`로 선정한다.

팀원 C가 작성하는 FlaskApp Helm Chart와 Kubernetes manifest를 Git 기준으로 배포하고, 배포 상태를 UI에서 확인하기 위해 사용한다.

팀원 B가 챙길 항목:

- `argocd` namespace 생성
- ArgoCD 설치 지원
- ArgoCD가 필요한 RBAC 권한 검토
- root app 적용 가능 여부 확인

### 3.6 Helm

배포 템플릿 관리는 `Helm`을 사용한다.

FlaskApp, monitoring, DB demo 등 여러 Kubernetes 리소스를 반복적으로 배포해야 하므로 Helm chart로 구성하면 on-prem과 EKS의 values 파일을 분리하기 쉽다.

팀원 C/E와 맞춰야 할 항목:

- `values-onprem.yaml`
- `values-aws.yaml`
- Ingress class 차이
- Service type 차이
- DB endpoint 차이
- S3 bucket 설정 차이

### 3.7 Prometheus + Grafana

모니터링은 `Prometheus + Grafana`로 구성한다.

Prometheus는 Kubernetes node, pod, service 상태를 수집하고, Grafana는 이를 대시보드로 시각화한다. Day 15 데모에서 on-prem Kubernetes 상태를 보여주는 핵심 화면으로 사용한다.

팀원 D와 맞춰야 할 항목:

- `monitoring` namespace 생성
- Prometheus/Grafana Helm values
- node-exporter DaemonSet 확인
- Grafana 외부 접근용 Ingress 또는 LoadBalancer 설정

### 3.8 테스트 Pod

클러스터 설치 후 기본 네트워크와 서비스 연결을 확인하기 위해 `BusyBox` 또는 `nicolaka/netshoot` 기반 테스트 Pod를 사용한다.

확인할 항목:

- Pod 간 통신
- Service DNS 조회
- FlaskApp Service 접근
- MariaDB Service 접근
- Ingress 경로 접근

예시 명령어:

```bash
kubectl run test-busybox --image=busybox:1.36 -it --rm --restart=Never -- sh
nslookup kubernetes.default.svc.cluster.local
```

### 3.9 운영 정책

기본 운영 정책은 `requests/limits`, `HPA`, `taints/tolerations`를 중심으로 정리한다.

| 항목 | 적용 목적 |
| --- | --- |
| requests/limits | Pod 리소스 요청량과 최대 사용량 제한 |
| HPA | CPU 또는 메모리 기준 Pod 자동 확장 |
| taints/tolerations | 인프라성 Pod와 일반 App Pod 배치 분리 |
| namespace | app, db, monitoring, argocd 등 리소스 영역 분리 |
| RBAC | 서비스 계정과 권한 범위 제한 |

## 4. 팀원별 영향도

| 팀원 | 관련 구성 요소 | 협업 내용 |
| --- | --- | --- |
| A | MetalLB, Ingress, CNI | IP pool, VLAN, 외부 접근 경로 확인 |
| C | ArgoCD, Helm, Ingress | FlaskApp 배포 방식, Helm chart, route 설정 |
| D | Prometheus, Grafana, RBAC | monitoring namespace, exporter, dashboard 구성 |
| E | Helm, EKS, Ingress | on-prem/EKS values 분리, ALB Ingress와 비교 |

## 5. 미확정 사항

Day 1 기준으로 방향은 정리했지만, 아래 항목은 Day 2 이후 실제 인프라 계획과 함께 확정한다.

- CNI 최종 선택: Calico 또는 Flannel
- Pod CIDR 대역
- Service CIDR 대역
- MetalLB IP pool 대역
- Ingress 외부 접근 도메인 또는 hosts 설정
- ArgoCD/Grafana 외부 노출 방식
- infra node 분리 여부
- HPA 기준 지표: CPU 또는 memory

## 6. 다음 진행 작업

이슈 3번 완료 후 다음 작업은 실제 구축 전에 필요한 값을 확정하는 것이다.

1. 팀원 A에게 Pod CIDR, Service CIDR, MetalLB IP pool 후보 확인
2. CNI를 Calico로 갈지 Flannel로 갈지 최종 결정
3. ArgoCD, monitoring, app, db namespace 이름 확정
4. Ingress Controller 외부 노출 방식을 MetalLB 연동으로 정리
5. Day 4 Ansible 자동화 대상에 containerd, kubeadm, kubelet, kubectl 설치 항목 반영

## 7. 완료 기준

- 컨테이너 런타임이 `containerd`로 정리되어 있다.
- CNI 후보가 Calico/Flannel로 정리되어 있다.
- 외부 LoadBalancer 구성이 `MetalLB`로 정리되어 있다.
- Ingress 구성이 `Nginx Ingress Controller`로 정리되어 있다.
- GitOps 연동 구성이 `ArgoCD + Helm`으로 정리되어 있다.
- 모니터링 연동 구성이 `Prometheus + Grafana`로 정리되어 있다.
- 앱, DB, 모니터링, AWS DR 담당자가 참고할 수 있는 협업 포인트가 정리되어 있다.

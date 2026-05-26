# 현재 아키텍처 요약

확인 기준: 2026-05-26 실측 및 repo 코드 기준  
상세 검증 문서: [full-verification-assessment-2026-05-26.md](../current/full-verification-assessment-2026-05-26.md)

## 1. 프로젝트 아키텍처 한 줄

온프렘 Proxmox 위에 Kubernetes 운영 환경을 구성하고 FlaskApp을 서비스하며, AWS에는 VPN/RDS/DMS/S3/ECR 기반 Pilot Light DR 환경을 준비한 하이브리드 DR 아키텍처이다.

## 2. 정상 운영 흐름

```text
사용자
  -> Bind9 DNS 172.16.41.53
  -> HAProxy/Keepalived VIP 172.16.42.99
  -> ingress-nginx-controller NodePort 30080/30443
  -> NGINX Ingress
  -> flaskapp-service ClusterIP
  -> FlaskApp Pod 2개
  -> MariaDB VM 172.16.43.160
```

## 3. 네트워크 기준

| 대역 | 역할 | 현재 기준 |
| --- | --- | --- |
| `172.16.30.0/24` | Proxmox 물리 관리망 | pve 관리, pfSense WAN |
| `172.16.41.0/24` | Public/DNS | Bind9 DNS `172.16.41.53` |
| `172.16.42.0/24` | DMZ/LB | HAProxy/Keepalived `lb-1`, `lb-2`, VIP `172.16.42.99` |
| `172.16.43.0/24` | Internal | Kubernetes node, MariaDB |
| `172.16.44.0/24` | Admin | Bastion, 운영 접근 |
| `10.10.10.0/24` | Ceph storage | Proxmox/Ceph 전용 |

## 4. 현재 주요 VM

| VM | IP | VLAN | 역할 |
| --- | --- | --- | --- |
| `bastion` | `172.16.44.100` | 40 | 운영자 접속, kubectl/ansible |
| `dns` | `172.16.41.53` | 10 | Bind9 내부 DNS |
| `lb-1` | `172.16.42.100` | 20 | HAProxy/Keepalived, 현재 VIP 보유 확인 |
| `lb-2` | `172.16.42.101` | 20 | HAProxy/Keepalived 백업 |
| `mariadb-1` | `172.16.43.160` | 30 | On-prem Primary DB |
| `k8s-cp-1` | `172.16.43.100` | 30 | Control plane |
| `k8s-cp-2` | `172.16.43.101` | 30 | Control plane |
| `k8s-cp-3` | `172.16.43.102` | 30 | Control plane |
| `k8s-worker-1` | `172.16.43.110` | 30 | Worker |
| `k8s-worker-2` | `172.16.43.111` | 30 | Worker |
| `k8s-worker-3` | `172.16.43.112` | 30 | Worker |
| `k8s-worker-4` | `172.16.43.113` | 30 | Worker |
| `k8s-worker-5` | `172.16.43.114` | 30 | Worker, monitoring workload 집중 |

## 5. Kubernetes 기준

| 항목 | 현재 기준 |
| --- | --- |
| 구성 | control plane 3대, worker 5대 |
| API VIP | `172.16.43.99:6443` |
| CNI | Calico |
| Ingress | ingress-nginx |
| Ingress Service | NodePort `30080`, `30443` |
| FlaskApp Service | ClusterIP |
| FlaskApp Host | `flaskapp.team.snow.internal` |
| Grafana Host | `grafana.team.snow.internal` |

## 6. DNS / Bind9 기준

Bind9는 실습 환경에서 Route 53 역할을 모의한다.

| Host | 정상 운영 응답 |
| --- | --- |
| `flaskapp.team.snow.internal` | `172.16.42.99` |
| `grafana.team.snow.internal` | `172.16.42.99` |

주의:

- `db.team.snow.internal`은 현재 Bind9에 없다.
- DB는 애플리케이션 ConfigMap에서 IP `172.16.43.160`으로 직접 연결한다.
- DR 전환 시에는 `flaskapp.team.snow.internal`을 AWS ALB DNS로 바꾸는 구조를 모의한다.

## 7. AWS DR 기준

| 항목 | 현재 기준 |
| --- | --- |
| VPC | `flaskapp-dr-vpc`, `10.20.0.0/16` |
| VPN | tunnel 2개 UP |
| RDS | `flaskapp-dr-mariadb`, MariaDB 10.11, private |
| DMS | `flaskapp-dr-full-load-cdc`, running |
| S3 | `flaskapp-proddata-kosa-project-team3-snow-lai9z` |
| ECR | `080252689380.dkr.ecr.ap-northeast-2.amazonaws.com/flaskapp` |
| EKS/ALB | 현재 없음. `dr_active=false` 평시 상태 |

## 8. 설계 결정 요약

| 결정 | 채택 |
| --- | --- |
| 정상 운영 진입점 | On-prem |
| 장애 시 진입점 | AWS ALB/EKS |
| On-prem 진입 방식 | HAProxy + Keepalived VIP |
| Kubernetes 외부 노출 | ingress-nginx NodePort |
| MetalLB | 최종 진입점으로 미사용 |
| DNS 전환 데모 | 내부 Bind9 |
| DB | Kubernetes 외부 MariaDB VM |
| DB DR | AWS DMS Full Load + CDC -> RDS |

## 9. 현재 리스크

- `lb-2` 상세 상태는 추가 확인 필요.
- `k8s-worker-5`에 monitoring/logging/AIOps workload가 몰려 있어 pve-5 장애 시 관측 계층 영향이 크다.
- DMS validation은 `Not enabled`라 복제 정합성은 별도 검증이 필요하다.
- AWS EKS/ALB는 현재 꺼져 있으므로 DR 전환 리허설이 필요하다.

> 문서 상태: 모니터링 배포 방식 결정 문서이다. 최신 기준은 Kubernetes 내부 `monitoring`/`logging`/`aiops` namespace 운영이며, Grafana host는 `grafana.team.snow.internal`이다.
>
> 이 문서의 `grafana.onprem.local`, `flaskapp.onprem.local` 예시는 과거 테스트 host로 본다.

# Monitoring 배포 방식 검토

## 1. 검토 배경

초기 설계에는 Prometheus/Grafana를 위한 `monitoring` VM을 별도로 두는 안이 포함되어 있다.

하지만 Kubernetes 운영 환경에서는 Prometheus, Grafana, Alertmanager를 별도 VM에 직접 설치하기보다 Kubernetes 내부에 배포하는 구성이 일반적이다. 이 문서는 기존 `monitoring` VM 설계와 Kubernetes 내부 배포 방식을 비교하고, 프로젝트에서 어떤 방향으로 정리할지 검토하기 위한 초안이다.

## 2. 기존 설계

| 항목 | 내용 |
| --- | --- |
| VM 이름 | `monitoring` |
| IP | `172.16.44.101` |
| VLAN | VLAN 40 Admin |
| 역할 | Prometheus, Grafana |
| 배치 후보 | `pve-4` |

기존 설계는 모니터링 시스템을 Kubernetes 클러스터 외부의 관리 VM에 설치하는 구조이다.

## 3. Kubernetes 내부 배포안

| 구성 요소 | 배포 위치 | 역할 |
| --- | --- | --- |
| Prometheus | Kubernetes `monitoring` namespace | node, pod, service, app metric 수집 |
| Grafana | Kubernetes `monitoring` namespace | 대시보드 시각화 |
| Alertmanager | Kubernetes `monitoring` namespace | 장애 조건 알림 |
| node-exporter | 각 Kubernetes node DaemonSet | node CPU, memory, disk, network metric 수집 |
| kube-state-metrics | Kubernetes `monitoring` namespace | Deployment, Pod, Node 등 K8s object 상태 metric 제공 |

배포 방식은 Helm chart 또는 ArgoCD Application을 기준으로 한다.

## 4. 비교

| 기준 | 별도 Monitoring VM | Kubernetes 내부 배포 |
| --- | --- | --- |
| 설치 방식 | VM에 패키지 또는 Docker로 직접 설치 | Helm/ArgoCD로 배포 |
| Kubernetes metric 수집 | kubeconfig, ServiceAccount, endpoint 설정 필요 | cluster 내부 ServiceDiscovery 사용 가능 |
| GitOps 관리 | 별도 Ansible/script 관리 필요 | ArgoCD Application으로 관리 가능 |
| 확장/변경 | VM 설정 변경 필요 | values.yaml 변경 후 sync |
| 장애 영향 | K8s 장애와 분리됨 | K8s 전체 장애 시 같이 영향 |
| 프로젝트 데모 적합성 | VM 관리 포인트 증가 | K8s 운영 화면 시연에 적합 |

## 5. 권장 방향

이 프로젝트에서는 Prometheus/Grafana/Alertmanager를 Kubernetes 내부 `monitoring` namespace에 배포하는 방향을 권장한다.

이유:

- Kubernetes node, pod, service 상태를 수집하는 것이 주 목적이다.
- Helm chart와 ArgoCD Application으로 관리하기 쉽다.
- WBS와 Kubernetes 구성 문서에도 `monitoring` namespace, Helm values, ServiceMonitor 구성이 이미 포함되어 있다.
- Day 15 데모에서 Grafana로 Kubernetes node/pod 상태를 보여주기 좋다.

## 6. 기존 `monitoring` VM 처리안

기존 `monitoring` VM은 다음 중 하나로 정리할 수 있다.

| 선택지 | 설명 | 권장 |
| --- | --- | --- |
| 제거 | Terraform VM 목록에서 제외하고 리소스를 줄인다. | 보류 |
| 보조 Probe VM으로 축소 | Kubernetes 외부에서 Ingress VIP, Grafana, API server health check를 확인하는 용도로만 사용한다. | 권장 |
| 기존 유지 | Prometheus/Grafana를 VM에 직접 설치한다. | 비권장 |

최종 방향은 `monitoring` VM을 제거하지 않고, Prometheus/Grafana 설치 대상에서도 제외하는 것이다. `monitoring` VM은 Kubernetes 외부에서 On-prem 서비스 상태를 확인하는 DR 판단 보조 지점으로 사용한다.

## 7. 네트워크 및 외부 접근

Grafana 외부 접근은 Ingress로 제공한다. 이 프로젝트에서는 MetalLB를 사용하지 않고 HAProxy/Keepalived 기반 LB 노드를 외부 진입점으로 사용한다.

| 방식 | 설명 |
| --- | --- |
| Ingress | `grafana.onprem.local`을 NGINX Ingress Controller로 라우팅 |
| HAProxy/Keepalived | `172.16.42.99` VIP가 Ingress Controller NodePort로 요청 전달 |

관리자는 Bastion 또는 관리망을 통해 Grafana에 접근한다.

접속 흐름:

```text
Browser
→ grafana.onprem.local
→ /etc/hosts 또는 내부 DNS
→ 172.16.42.99 HAProxy/Keepalived VIP
→ NGINX Ingress Controller
→ monitoring-grafana Service
→ Grafana Pod
```

검증 결과:

```bash
curl -I http://172.16.42.99 -H "Host: grafana.onprem.local"
```

정상 응답:

```text
HTTP/1.1 302 Found
Location: /login
```

브라우저 접속 PC에는 다음 hosts 항목을 추가한다.

```text
172.16.42.99 grafana.onprem.local
```

## 8. 구현 작업 초안

1. `monitoring` namespace 생성
2. kube-prometheus-stack Helm values 작성
3. Prometheus, Grafana, Alertmanager 배포
4. node-exporter DaemonSet 확인
5. kube-state-metrics 확인
6. Grafana Service 또는 Ingress 노출
7. `grafana.onprem.local` DNS 또는 hosts 등록
8. Kubernetes node/pod 기본 dashboard import
9. FlaskApp `/metrics` 노출 여부 검토
10. ServiceMonitor/PrometheusRule 고도화

## 9. 실제 구현 작업 기록

### 9.1 Ansible: DB VM exporter 구성

DB는 Kubernetes 내부가 아니라 별도 VM(`mariadb-1`, `172.16.43.160`)에서 운영한다. 따라서 Kubernetes 내부의 Prometheus가 DB 상태를 수집하려면 DB VM이 metric endpoint를 노출해야 한다.

DB VM에는 Ansible로 다음 exporter를 설치했다.

| exporter | 포트 | 역할 |
| --- | ---: | --- |
| node_exporter | `9100` | DB VM CPU, memory, disk, network metric 수집 |
| mysqld_exporter | `9104` | MariaDB 접속 상태, connection, query, status metric 수집 |

작업 파일:

```text
infra/onprem/ansible/playbooks/db-exporters.yml
```

실행:

```bash
cd infra/onprem/ansible

ansible-playbook -i inventories/onprem/hosts.yml playbooks/db-exporters.yml \
  -e mysqld_exporter_password='별도-긴-비밀번호'
```

검증:

```bash
curl http://172.16.43.160:9100/metrics | head
curl http://172.16.43.160:9104/metrics | head
```

`mysqld_exporter_password`는 MariaDB root 비밀번호나 FlaskApp DB 비밀번호가 아니다. mysqld_exporter가 MariaDB에 접속하기 위한 전용 계정(`exporter`)의 비밀번호이다.

생성되는 DB 계정 개념:

```sql
CREATE USER IF NOT EXISTS 'exporter'@'localhost'
IDENTIFIED BY '<mysqld_exporter_password>';

GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
```

### 9.2 Ansible: monitoring VM external health check 구성

`monitoring` VM은 Prometheus/Grafana 설치 VM으로 사용하지 않는다. 대신 Kubernetes 바깥에서 On-prem 서비스가 사용자 관점에서 살아있는지 확인하는 external probe VM으로 사용한다.

작업 파일:

```text
infra/onprem/ansible/playbooks/monitoring-external-probe.yml
```

설치되는 구성:

| 파일/서비스 | 역할 |
| --- | --- |
| `/usr/local/bin/onprem-healthcheck.sh` | HTTP/TCP health check 수행 |
| `/etc/onprem-healthcheck.env` | timeout, log path 설정 |
| `/etc/systemd/system/onprem-healthcheck.service` | 1회 실행 systemd service |
| `/etc/systemd/system/onprem-healthcheck.timer` | 주기 실행 timer |
| `/var/log/onprem-healthcheck.log` | health check 결과 로그 |

체크 대상:

| 이름 | 대상 | 방식 |
| --- | --- | --- |
| `flaskapp_health` | `http://flaskapp.onprem.local/healthz` | HTTP |
| `grafana` | `http://grafana.onprem.local` | HTTP |
| `haproxy_vip_http` | `172.16.42.99:80` | TCP |
| `kubernetes_api_vip` | `172.16.43.99:6443` | TCP |
| `mariadb` | `172.16.43.160:3306` | TCP |

검증:

```bash
ansible -i inventories/onprem/hosts.yml monitoring -b -m shell \
  -a "systemctl status onprem-healthcheck.timer --no-pager"

ansible -i inventories/onprem/hosts.yml monitoring -b -m shell \
  -a "tail -n 20 /var/log/onprem-healthcheck.log"
```

정상 로그 예:

```text
kind=http name=flaskapp_health target=http://flaskapp.onprem.local/healthz status=ok
kind=http name=grafana target=http://grafana.onprem.local status=ok
kind=tcp name=haproxy_vip_http target=172.16.42.99:80 status=ok
kind=tcp name=kubernetes_api_vip target=172.16.43.99:6443 status=ok
kind=tcp name=mariadb target=172.16.43.160:3306 status=ok
```

### 9.3 ArgoCD/Helm: kube-prometheus-stack 배포

Kubernetes 내부 모니터링 스택은 ArgoCD와 Helm으로 배포한다.

작업 파일:

```text
infra/argocd/apps/monitoring.yaml
infra/helm/monitoring/Chart.yaml
infra/helm/monitoring/values.yaml
```

배포 구조:

```text
root-app
→ argocd/apps/monitoring.yaml
→ helm/monitoring
→ kube-prometheus-stack dependency
→ monitoring namespace
```

설치되는 주요 구성:

| 구성 요소 | 역할 |
| --- | --- |
| Prometheus | metric 수집/저장 |
| Grafana | 대시보드/Explore |
| Alertmanager | 알림 처리 |
| node-exporter | Kubernetes node metric |
| kube-state-metrics | Kubernetes object 상태 metric |
| Prometheus Operator | Prometheus CRD와 scrape 설정 관리 |

검증:

```bash
kubectl get app -n argocd monitoring
kubectl get pods -n monitoring -o wide
kubectl get ingress -n monitoring
```

정상 기준:

```text
monitoring   Synced   Healthy
```

### 9.4 DB exporter scrape 방식 변경: ServiceMonitor에서 static scrape로 전환

초기에는 외부 DB VM exporter를 Kubernetes 리소스로 표현하기 위해 다음 구조를 시도했다.

```text
ServiceMonitor
→ Service
→ Endpoints 또는 EndpointSlice
→ 172.16.43.160:9100/9104
```

하지만 현재 ArgoCD 설정에서 `Endpoints`와 `EndpointSlice` 리소스가 제외되어 있었다.

발생한 warning:

```text
Resource /Endpoints mariadb-vm-exporters is excluded in the settings
Resource discovery.k8s.io/EndpointSlice mariadb-vm-exporters is excluded in the settings
```

따라서 DB VM exporter 연동은 Service/Endpoint 방식이 아니라 Prometheus static scrape 방식으로 변경했다.

최종 구조:

```text
Prometheus
→ additionalScrapeConfigs
→ 172.16.43.160:9100
→ 172.16.43.160:9104
```

`helm/monitoring/values.yaml` 설정:

```yaml
kube-prometheus-stack:
  prometheus:
    prometheusSpec:
      additionalScrapeConfigs:
        - job_name: mariadb-vm-node-exporter
          static_configs:
            - targets:
                - 172.16.43.160:9100
              labels:
                instance: mariadb-1
                role: database
                exporter: node_exporter

        - job_name: mariadb-vm-mysqld-exporter
          static_configs:
            - targets:
                - 172.16.43.160:9104
              labels:
                instance: mariadb-1
                role: database
                exporter: mysqld_exporter
```

Service/Endpoint 방식에서 사용하던 다음 파일은 제거한다.

```text
infra/helm/monitoring/templates/db-exporter-service.yaml
infra/helm/monitoring/templates/db-exporter-servicemonitor.yaml
```

### 9.5 네트워크/방화벽 검증

DB exporter가 Bastion에서는 보이지만 Prometheus에서는 `DOWN`으로 보이는 문제가 있었다.

Prometheus target 상태:

```text
mariadb-vm-node-exporter      up = 0
mariadb-vm-mysqld-exporter    up = 0
lastError = context deadline exceeded
```

이는 Prometheus 설정 문제가 아니라 Prometheus Pod 또는 worker node에서 DB VM exporter 포트로 접근하지 못하는 네트워크/방화벽 문제였다.

검증한 경로:

```text
Bastion → 172.16.43.160:9100/9104       OK
worker-1 → 172.16.43.160:9100/9104      timeout
curl-test Pod → 172.16.43.160:9100/9104 timeout
```

DB VM에서 worker/Pod 대역이 exporter 포트에 접근할 수 있도록 방화벽을 허용한 뒤 정상 수집되었다.

검증:

```bash
kubectl exec -n monitoring curl-test -- \
  curl -s --max-time 5 http://172.16.43.160:9100/metrics | head
```

정상 응답:

```text
# HELP go_gc_duration_seconds ...
# TYPE go_gc_duration_seconds summary
```

Prometheus target 확인:

```bash
curl -sG "http://localhost:9090/api/v1/query" \
  --data-urlencode 'query=up{job=~"mariadb-vm-.*"}' | jq
```

정상 결과:

```json
{
  "metric": {
    "job": "mariadb-vm-mysqld-exporter",
    "instance": "mariadb-1",
    "exporter": "mysqld_exporter",
    "role": "database"
  },
  "value": [1779330133.906, "1"]
}
{
  "metric": {
    "job": "mariadb-vm-node-exporter",
    "instance": "mariadb-1",
    "exporter": "node_exporter",
    "role": "database"
  },
  "value": [1779330133.906, "1"]
}
```

MariaDB 접속 상태:

```promql
mysql_up
```

정상 결과:

```text
mysql_up = 1
```

## 10. Grafana에서 확인하는 방법

### 10.1 접속

브라우저에서 다음 주소로 접속한다.

```text
http://grafana.onprem.local
```

브라우저를 여는 PC에는 hosts 또는 내부 DNS가 필요하다.

```text
172.16.42.99 grafana.onprem.local
```

초기 로그인:

```text
ID: admin
PW: admin
```

### 10.2 Explore에서 외부 DB 확인

Grafana 왼쪽 메뉴에서 `Explore`로 이동한다. 상단 datasource는 `Prometheus`를 선택한다.

처음에는 `Builder`보다 `Code` 모드가 편하다. 쿼리 입력 후 `Run query`를 누른다.

외부 DB exporter 생존 확인:

```promql
up{job=~"mariadb-vm-.*"}
```

정상 결과:

```text
up{exporter="mysqld_exporter", instance="mariadb-1", job="mariadb-vm-mysqld-exporter", role="database"} 1
up{exporter="node_exporter", instance="mariadb-1", job="mariadb-vm-node-exporter", role="database"} 1
```

MariaDB 실제 접속 상태:

```promql
mysql_up
```

정상 결과:

```text
mysql_up{exporter="mysqld_exporter", instance="mariadb-1", job="mariadb-vm-mysqld-exporter", role="database"} 1
```

### 10.3 DB 운영 지표 예시

DB 연결 수:

```promql
mysql_global_status_threads_connected
```

DB 쿼리 처리량:

```promql
rate(mysql_global_status_queries[5m])
```

DB VM CPU 사용률:

```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{job="mariadb-vm-node-exporter",mode="idle"}[5m])) * 100)
```

DB VM 메모리 사용률:

```promql
(1 - (node_memory_MemAvailable_bytes{job="mariadb-vm-node-exporter"} / node_memory_MemTotal_bytes{job="mariadb-vm-node-exporter"})) * 100
```

DB VM 디스크 사용률:

```promql
(1 - (node_filesystem_avail_bytes{job="mariadb-vm-node-exporter",fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes{job="mariadb-vm-node-exporter",fstype!~"tmpfs|overlay"})) * 100
```

### 10.4 Kubernetes 상태 확인

Kubernetes 상태는 Grafana의 기본 dashboard에서 확인한다.

찾아볼 dashboard 후보:

```text
Kubernetes / Compute Resources / Cluster
Kubernetes / Compute Resources / Namespace
Kubernetes / Compute Resources / Pod
Node Exporter
Prometheus
```

Kubernetes 내부 리소스는 kube-prometheus-stack의 기본 ServiceMonitor/PodMonitor를 통해 수집된다. 외부 DB VM은 Kubernetes 리소스가 아니므로 기본 Kubernetes dashboard에 자동으로 포함되지 않을 수 있다. 외부 DB는 Explore에서 `mariadb-vm-*`, `mysql_*` metric으로 확인한다.

## 11. 남은 결정 사항

| 결정 항목 | 후보 |
| --- | --- |
| 설치 chart | `prometheus-community/kube-prometheus-stack` |
| 외부 노출 | Ingress + HAProxy/Keepalived VIP |
| Grafana 접속 | `grafana.onprem.local` → `172.16.42.99` |
| 배치 노드 | `k8s-worker-3` infra node 우선 여부 |
| 기존 monitoring VM | external probe 용도로 유지 |
| DB exporter target | 단일 DB는 static scrape, Galera 확장 시 target 목록 또는 DNS-SD 검토 |

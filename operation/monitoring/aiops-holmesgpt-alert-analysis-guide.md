# HolmesGPT 기반 AIOps Alert 분석 운영 가이드

## 1. 문서 목적

이 문서는 On-prem Kubernetes 환경에서 Alertmanager, Loki, HolmesGPT를 연계하여 장애 알림 발생 시 AIOps 방식으로 원인을 분석하는 운영 절차를 정리한다.

현재 구성은 Alertmanager가 이메일로 장애 알림을 전송하고, 운영자가 알림에 포함된 `alertname`, `namespace`, `service`, `description`, `holmes_query` 값을 바탕으로 HolmesGPT에 질문하여 Kubernetes 상태와 Loki 로그를 함께 분석하는 방식이다.

운영 목표는 다음과 같다.

- Alertmanager 이메일 알림을 받은 뒤 빠르게 원인 분석을 시작한다.
- Kubernetes deployment, pod, event, Loki 로그를 함께 확인한다.
- 단순 애플리케이션 오류, On-prem 노드/LAN 오류, DR 실행이 필요한 장애를 구분한다.
- 분석 범위를 제한하여 OpenAI API 비용을 통제한다.
- 향후 Alertmanager webhook 기반 자동 분석으로 확장할 수 있는 기반을 마련한다.

## 2. 전체 구성

```text
PrometheusRule
  -> Prometheus
    -> Alertmanager
      -> Email alert
        -> Operator copies alert fields
          -> holmes CLI wrapper
            -> HolmesGPT API
              -> Kubernetes toolset
              -> Loki toolset
              -> gpt-5-mini
```

현재 주요 컴포넌트는 다음과 같다.

| 구성 요소 | Namespace | 역할 |
| --- | --- | --- |
| Alertmanager | `monitoring` | Prometheus alert 이메일 전송 |
| Prometheus | `monitoring` | PrometheusRule 평가 및 alert 발생 |
| Loki | `logging` | Kubernetes 로그 저장 및 검색 |
| Alloy | `logging` | Kubernetes 로그 수집 후 Loki 전송 |
| HolmesGPT | `aiops` | Alert 기반 원인 분석 |
| OpenAI API | 외부 | HolmesGPT LLM 분석 모델 |
| Bastion | `172.16.44.100` | 운영자가 `holmes` 명령 실행 |

## 3. 현재 HolmesGPT 구성

HolmesGPT는 Kubernetes 내부에 다음과 같이 배포되어 있다.

```bash
kubectl -n aiops get pods -l app=holmes
kubectl -n aiops get svc aiops-holmes
```

예상 상태:

```text
aiops-holmes-...   1/1   Running
```

HolmesGPT 서비스:

```text
aiops-holmes.aiops.svc.cluster.local
```

HolmesGPT API:

```text
/readyz
/api/chat
```

현재 모델 설정은 비용 절감을 위해 `gpt-5-mini`를 사용한다.

주의할 점은 HolmesGPT 내부 model key는 `gpt-5`로 보일 수 있다는 것이다. 실제 backend model은 ConfigMap에서 `openai/gpt-5-mini`를 바라보도록 설정되어 있다.

확인 명령:

```bash
kubectl -n aiops get cm custom-toolsets-configmap \
  -o jsonpath='{.data.model_list\.yaml}'
```

예상 설정:

```yaml
gpt-5:
  api_key: '{{ env.OPENAI_API_KEY }}'
  model: openai/gpt-5-mini
  temperature: 0
```

## 4. 사용 전 확인

HolmesGPT를 사용하기 전 다음 상태를 확인한다.

```bash
kubectl get nodes
kubectl -n aiops get pods
kubectl -n logging get pods
kubectl -n monitoring get pods
```

특히 On-prem 환경에서는 물리 LAN 또는 Proxmox 노드 문제로 특정 worker가 `NotReady`가 될 수 있다.

다음 메시지가 보이면 HolmesGPT 문제가 아니라 물리 네트워크 또는 노드 접근 문제일 수 있다.

```text
NodeNotReady
node.kubernetes.io/unreachable
no route to host
FailedScheduling
```

예시:

```text
dial tcp 172.16.43.114:10250: connect: no route to host
```

이 경우 우선 Proxmox/LAN 케이블 상태를 확인한다.

## 5. Bastion에서 HolmesGPT 사용하기

### 5.1 접속 위치

HolmesGPT 질문은 Bastion 서버에서 실행한다.

```bash
ssh kosa@172.16.44.100
```

### 5.2 holmes wrapper 명령

Bastion에는 다음 wrapper 명령을 구성한다.

```text
/home/kosa/bin/holmes
```

명령이 잡히는지 확인한다.

```bash
which holmes
```

정상 출력:

```text
/home/kosa/bin/holmes
```

만약 `holmes: command not found`가 발생하면 PATH를 추가한다.

```bash
export PATH="$HOME/bin:$PATH"
```

영구 반영:

```bash
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### 5.3 기본 테스트

```bash
holmes "네 라고만 대답해"
```

정상 응답:

```text
네
```

### 5.4 port-forward 방식

HolmesGPT 서비스는 ClusterIP이므로 Bastion에서 port-forward를 사용한다.

```bash
kubectl -n aiops port-forward svc/aiops-holmes 18081:80
```

현재 wrapper는 port-forward가 없으면 자동으로 시도하도록 구성할 수 있다. 직접 확인하려면 다음 명령을 사용한다.

```bash
curl -fsS http://127.0.0.1:18081/readyz
```

예상 응답:

```json
{"status":"ready","models":["gpt-5","gpt-5.4"]}
```

여기서 `gpt-5`로 표시되어도 실제 backend는 `gpt-5-mini`일 수 있다.

## 6. Alertmanager 이메일 기반 질문 방법

Alertmanager 이메일 예시는 다음과 같다.

```text
1 alert for alertname=FlaskAppReplicasUnavailable namespace=flaskapp-prod

Labels
alertname = FlaskAppReplicasUnavailable
aiops_scope = application
namespace = flaskapp-prod
prometheus = monitoring/monitoring-prometheus
service = flaskapp
severity = critical

Annotations
description = Deployment in namespace flaskapp-prod has no available replicas for more than 2 minutes.
holmes_query = Investigate why FlaskApp has no available replicas in flaskapp-prod. Check deployment, pods, events, logs, and related Prometheus metrics.
summary = FlaskApp has no available replicas
```

이 알림을 받으면 다음 값을 질문에 포함한다.

| 필드 | 질문에 넣는 이유 |
| --- | --- |
| `alertname` | 어떤 종류의 장애인지 식별 |
| `namespace` | 조사 범위 제한 |
| `service` | 관련 workload 식별 |
| `severity` | 우선순위 판단 |
| `description` | 알림 발생 조건 설명 |
| `holmes_query` | 사전에 정의된 조사 방향 |

## 7. 대표 질문 템플릿

### 7.1 기본 Alert 분석

```bash
holmes "Alert 분석:
alertname=<alertname>
namespace=<namespace>
service=<service>
severity=<severity>
description='<description>'
최근 30분 기준으로 deployment, pod, event, Loki error 로그를 확인해줘.
결과는 1) 현재상태 2) 가능원인 3) 조치명령어 4) DR 필요 여부로 짧게 정리해줘."
```

### 7.2 FlaskAppReplicasUnavailable 전용

```bash
holmes "FlaskAppReplicasUnavailable alert 분석해줘.
namespace=flaskapp-prod
service=flaskapp
최근 30분 deployment, pod event, Loki logs 기준으로 원인과 조치 알려줘."
```

### 7.3 비용 절감용 빠른 테스트

```bash
holmes "FlaskAppReplicasUnavailable alert 테스트.
namespace=flaskapp-prod service=flaskapp.
최근 5분 pod/deployment 상태만 확인하고 3줄 이내로 답해줘."
```

### 7.4 Loki 로그만 짧게 확인

```bash
holmes "namespace=flaskapp-prod service=flaskapp 최근 10분 Loki 로그에서 error/fail/exception 포함 로그 최대 3줄만 찾고, 있으면 원인 1줄로 요약해줘."
```

### 7.5 DR 판단 포함 분석

```bash
holmes "DR 판단 포함 장애 분석:
alertname=<alertname>
namespace=<namespace>
service=<service>
severity=<severity>
description='<description>'
최근 30분 기준으로 deployment, pod, event, node 상태, Loki error 로그를 확인해줘.
DR 실행 기준:
- 단일 pod/app 설정/이미지/코드 오류면 DR 불필요
- 단일 worker 또는 물리 LAN 문제면 노드 격리/재스케줄링 우선, DR 대기
- 여러 노드/스토리지/네트워크/control-plane 장애로 서비스 복구 지연 가능성이 높으면 DR 검토 또는 실행 권고
답변 형식:
DR 판단: 실행/대기/불필요/판단보류
근거:
가능 원인:
즉시 조치 명령어:"
```

### 7.6 On-prem/LAN 문제 의심

```bash
holmes "온프렘 인프라 문제 여부 확인:
최근 30분 기준으로 node 상태, pod scheduling event, no route to host, NodeNotReady, unreachable 관련 event를 확인해줘.
문제가 특정 worker/pve/LAN 장애로 보이는지, 앱 장애로 보이는지 구분하고 DR 필요 여부를 알려줘."
```

### 7.7 DR 실행 전 최종 확인

```bash
holmes "DR 실행 전 최종 판단:
namespace=flaskapp-prod service=flaskapp
최근 30분 기준으로 앱 pod 상태, deployment available replica, node 상태, event, Loki error 로그를 확인해줘.
아래 형식으로 답해줘:
DR 판단: 실행/대기/불필요
근거:
즉시 조치:
추가 확인:"
```

## 8. DR 판단 기준

HolmesGPT는 최종 의사결정자가 아니라 판단 보조 도구다. DR 실행 여부는 운영자가 최종 판단한다.

다만 질문에 아래 기준을 포함하면 응답 품질이 좋아진다.

### 8.1 DR 불필요 가능성이 높은 경우

다음은 보통 DR 대상이 아니다.

- 단일 Pod crash
- 애플리케이션 코드 예외
- 잘못된 환경변수 또는 Secret
- ImagePullBackOff
- ECR pull secret 만료
- readiness/liveness probe 설정 오류
- 단일 Deployment rollout 실패
- 단일 worker에만 발생한 scheduling 문제

예시 판단:

```text
DR 판단: 불필요
근거: flaskapp pod만 비정상이며 node/control-plane/storage 전체 장애 증거 없음
```

### 8.2 DR 대기 또는 보류가 적절한 경우

다음은 즉시 DR보다 On-prem 복구를 먼저 시도한다.

- 단일 worker `NotReady`
- 단일 Proxmox 노드 LAN 불량
- 특정 worker kubelet 접근 실패
- `no route to host`가 특정 node에만 발생
- Pod가 다른 worker로 재스케줄링 가능

예시 판단:

```text
DR 판단: 대기
근거: k8s-worker-5만 NotReady이며 다른 worker는 Ready 상태
즉시 조치: LAN 케이블 확인, node cordon/drain 검토, pod 재스케줄링 확인
```

### 8.3 DR 검토 또는 실행이 필요한 경우

다음은 DR 검토 대상이다.

- 다수 worker가 동시에 `NotReady`
- control-plane 다수 장애
- Kubernetes API 접근 불가
- Ingress 전체 장애
- Ceph/RBD/S3 등 공통 storage 장애
- DB primary 접근 불가 및 On-prem 복구 지연
- 서비스 전체가 장시간 unavailable
- On-prem 네트워크 전체 단절

예시 판단:

```text
DR 판단: 검토
근거: 다수 node와 ingress 경로가 동시에 비정상이며 앱 단일 장애로 보기 어려움
```

## 9. 운영 시 기대 효과

HolmesGPT를 AIOps 분석 보조로 사용하면 다음 효과를 얻을 수 있다.

### 9.1 장애 대응 시간 단축

운영자가 Alertmanager 메일을 받은 뒤 수동으로 다음 명령을 하나씩 실행하지 않아도 된다.

```bash
kubectl get deploy
kubectl get pods
kubectl describe pod
kubectl get events
kubectl logs
```

HolmesGPT가 질문에 따라 Kubernetes 상태와 Loki 로그를 함께 조회하고 요약한다.

### 9.2 로그와 리소스 상태를 함께 분석

Pod가 `Running`이어도 애플리케이션 내부에서는 오류가 발생할 수 있다. HolmesGPT는 Kubernetes 상태와 Loki 로그를 같이 보도록 질문할 수 있어 단순 리소스 상태 확인보다 실제 장애 원인에 더 가까운 분석이 가능하다.

### 9.3 DR 판단 보조

DR은 비용과 운영 영향이 큰 작업이다. HolmesGPT를 사용하면 다음을 빠르게 구분할 수 있다.

- 애플리케이션 자체 문제
- 단일 Pod 문제
- 단일 node/LAN 문제
- 다중 node 또는 공통 인프라 문제
- DR 검토가 필요한 광범위 장애

### 9.4 운영 지식 표준화

질문 템플릿을 사용하면 담당자마다 다른 방식으로 조사하는 문제를 줄일 수 있다.

Alertmanager 메일의 `holmes_query` annotation을 함께 관리하면, alert마다 권장 조사 방향을 미리 정의할 수 있다.

### 9.5 비용 통제

`gpt-5-mini`를 사용하고 질문 범위를 제한하면 실습 및 운영 PoC 수준에서 비용 부담을 줄일 수 있다.

비용을 줄이는 질문 방식:

- 시간 범위를 `최근 5분`, `최근 10분`, `최근 30분`으로 제한한다.
- Loki 로그는 `최대 3줄`, `최대 5줄`처럼 제한한다.
- 처음부터 모든 namespace를 조사하지 않는다.
- 먼저 deployment/pod/event만 보고, 필요할 때 Loki를 추가한다.

## 10. 비용 절감용 질문 패턴

가장 저렴한 테스트:

```bash
holmes "namespace=flaskapp-prod pod 상태만 확인하고 3줄로 답해줘. Loki 보지마."
```

짧은 Loki 테스트:

```bash
holmes "namespace=flaskapp-prod 최근 5분 Loki error 로그 최대 1줄만 찾아줘. 없으면 없다고만 답해."
```

Alert 빠른 판별:

```bash
holmes "FlaskAppReplicasUnavailable alert. namespace=flaskapp-prod. deployment/pod 상태만 보고 5줄 이내로 원인과 조치 알려줘. Loki는 보지마."
```

실제 장애 분석:

```bash
holmes "FlaskAppReplicasUnavailable alert 분석.
namespace=flaskapp-prod service=flaskapp.
최근 30분 deployment, pod event, Loki error 로그 기준으로 원인과 조치, DR 필요 여부를 알려줘."
```

## 11. 응답이 느릴 때

HolmesGPT 응답은 일반 ChatGPT보다 느릴 수 있다.

이유:

- 질문 분석 후 조사 계획을 만든다.
- Kubernetes tool을 호출한다.
- Loki query를 실행한다.
- 결과를 다시 LLM에 넣어 요약한다.
- 필요하면 여러 번 반복한다.

일반적으로 다음 정도를 예상한다.

| 질문 유형 | 예상 시간 |
| --- | --- |
| 단순 응답 테스트 | 5초~20초 |
| Pod 상태만 확인 | 20초~60초 |
| Kubernetes + Loki 분석 | 1분~3분 |
| 다중 namespace 또는 복잡한 장애 분석 | 3분 이상 |

진행 상태 확인:

```bash
kubectl -n aiops logs deploy/aiops-holmes -f
```

로그에서 다음 문구가 보이면 실제로 `gpt-5-mini`를 사용 중이다.

```text
LiteLLM completion() model= gpt-5-mini; provider = openai
```

## 12. 문제 해결

### 12.1 `holmes: command not found`

원인: Bastion shell PATH에 `/home/kosa/bin`이 없다.

해결:

```bash
export PATH="$HOME/bin:$PATH"
which holmes
```

영구 반영:

```bash
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

절대 경로로도 실행 가능하다.

```bash
/home/kosa/bin/holmes "네 라고만 대답해"
```

### 12.2 port already in use

다음 에러가 발생할 수 있다.

```text
Unable to listen on port 18080
bind: address already in use
```

원인: 이미 port-forward가 실행 중이다.

해결 1: 다른 포트 사용

```bash
kubectl -n aiops port-forward svc/aiops-holmes 18081:80
```

해결 2: 기존 프로세스 확인 후 종료

```bash
lsof -i :18080
kill <PID>
```

### 12.3 HolmesGPT는 Running인데 응답이 이상함

확인:

```bash
kubectl -n aiops get pods
kubectl -n aiops logs deploy/aiops-holmes --tail=100
```

OpenAI quota 문제가 있으면 다음과 유사한 에러가 나온다.

```text
You exceeded your current quota
```

이 경우 OpenAI Platform billing과 project API key를 확인한다.

### 12.4 Prometheus toolset 실패

현재 HolmesGPT 로그에서 Prometheus toolset이 실패할 수 있다.

예시:

```text
Toolset prometheus/metrics failed
Connection refused
```

이 경우 Prometheus metric 기반 분석은 제한될 수 있다. 임시 운영 기준은 Kubernetes 상태와 Loki 로그 중심으로 분석하는 것이다.

Prometheus 연결이 필요하면 다음을 확인한다.

```bash
kubectl -n monitoring get svc monitoring-prometheus
kubectl -n monitoring get pods -l app.kubernetes.io/name=prometheus
```

### 12.5 On-prem LAN 또는 Proxmox 문제

다음 이벤트가 보이면 HolmesGPT 자체 문제가 아니라 On-prem 물리 네트워크 문제일 가능성이 있다.

```text
NodeNotReady
node.kubernetes.io/unreachable
no route to host
FailedScheduling
```

확인:

```bash
kubectl get nodes -o wide
kubectl get events -A --sort-by=.lastTimestamp
```

조치:

- 해당 Proxmox 노드의 LAN 케이블 확인
- 스위치 포트 link 확인
- 해당 Kubernetes node가 `Ready`로 복귀하는지 확인
- 필요 시 node cordon/drain 후 pod 재스케줄링 검토

## 13. 향후 자동화 방향

현재는 Alertmanager 이메일을 보고 운영자가 `holmes` 명령으로 수동 분석한다.

다음 단계는 Alertmanager webhook 기반 자동 분석이다.

목표 구조:

```text
Alertmanager
  -> email receiver
  -> webhook receiver
    -> aiops-analyzer service
      -> HolmesGPT /api/chat
      -> analysis result
      -> email or Slack notification
```

주의할 점:

- Alertmanager 기본 이메일에 HolmesGPT 분석 결과를 직접 합치는 방식은 어렵다.
- 현실적인 방식은 1차 Alertmanager 이메일과 2차 HolmesGPT 분석 이메일을 분리하는 것이다.
- webhook service는 alert payload에서 `holmes_query`를 추출해 HolmesGPT에 전달하면 된다.

Alertmanager receiver 예시:

```yaml
receivers:
  - name: email
    email_configs:
      - to: snowkwon420@gmail.com
        send_resolved: true

  - name: holmes-webhook
    webhook_configs:
      - url: http://aiops-analyzer.aiops.svc.cluster.local/alert
        send_resolved: false
```

분석 webhook service가 생성할 HolmesGPT 질문 예시:

```text
Alert 분석:
alertname={{ alertname }}
namespace={{ namespace }}
service={{ service }}
severity={{ severity }}
description='{{ description }}'
holmes_query='{{ holmes_query }}'
최근 30분 기준으로 Kubernetes 상태와 Loki error 로그를 확인하고
DR 판단, 가능 원인, 즉시 조치 명령어를 알려줘.
```

## 14. 운영 체크리스트

Alert 발생 시 운영자는 다음 순서로 확인한다.

- [ ] Alertmanager 이메일에서 `alertname`, `namespace`, `service`, `description` 확인
- [ ] Bastion 접속
- [ ] `holmes` 명령 사용 가능 여부 확인
- [ ] 비용 절감을 위해 시간 범위와 로그 개수 제한
- [ ] HolmesGPT에 Alert 분석 질문 실행
- [ ] 응답이 느리면 `kubectl -n aiops logs deploy/aiops-holmes -f` 확인
- [ ] `DR 판단` 결과가 `실행`, `대기`, `불필요`, `판단보류` 중 무엇인지 확인
- [ ] HolmesGPT 근거가 Kubernetes/Loki 상태와 맞는지 검토
- [ ] 최종 DR 실행 여부는 운영자가 판단

## 15. 핵심 요약

HolmesGPT는 Alertmanager 이메일을 대체하는 도구가 아니라, 이메일 알림 이후 원인 분석을 빠르게 시작하기 위한 AIOps 보조 도구다.

현재 운영 방식은 다음과 같다.

```text
Alertmanager 이메일 확인
-> alert 필드 복사
-> bastion에서 holmes 명령 실행
-> Kubernetes 상태와 Loki 로그 분석
-> DR 필요 여부 판단 보조
```

가장 많이 사용할 실전 명령:

```bash
holmes "DR 판단 포함 장애 분석:
alertname=FlaskAppReplicasUnavailable
namespace=flaskapp-prod
service=flaskapp
severity=critical
description='Deployment in namespace flaskapp-prod has no available replicas for more than 2 minutes.'
최근 30분 기준으로 deployment, pod, event, node 상태, Loki error 로그를 확인해줘.
DR 판단: 실행/대기/불필요/판단보류 중 하나로 답하고, 근거와 즉시 조치 명령어를 짧게 알려줘."
```

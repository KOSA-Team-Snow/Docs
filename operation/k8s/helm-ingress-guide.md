# FlaskApp Helm Chart 구조 및 Ingress 연결 가이드

> 담당: 팀원 C (@ireneminhee)  
> 관련 이슈: #15  
> 상태: Helm chart 문서화 완료 / Ingress 접속 확인은 클러스터 구성 후 추가 예정

---

## 1. Helm chart란?

Helm은 Kubernetes용 패키지 매니저다. 여러 manifest 파일(deployment, service, ingress 등)을 하나의 **chart**로 묶어 관리한다.

```
Helm 없이: deployment.yaml, service.yaml, ingress.yaml ... 파일을 각각 kubectl apply
Helm 있으면: helm upgrade --install flaskapp ./helm/flaskapp 한 줄로 전체 배포
```

환경별로 달라지는 값(DB 주소, 이미지 태그, Ingress 클래스 등)은 `values.yaml`로 분리해 관리한다.

---

## 2. Chart 디렉토리 구조

```
infra/helm/flaskapp/
├── Chart.yaml            # chart 메타정보 (이름, 버전)
├── values.yaml           # 기본 변수값
└── templates/            # 실제 K8s manifest 템플릿
    ├── _helpers.tpl      # 공통 헬퍼 함수 (배포되지 않음)
    ├── namespace.yaml
    ├── deployment.yaml
    ├── service.yaml
    ├── configmap.yaml
    ├── secret.yaml
    ├── ingress.yaml
    └── NOTES.txt         # helm install 완료 후 출력 메시지
```

---

## 3. Chart.yaml

chart 자체의 메타정보를 담는다.

```yaml
apiVersion: v2
name: flaskapp
description: FlaskApp Helm Chart — KOSA Snow Project
type: application
version: 0.1.0      # chart 버전 (변경 시 올린다)
appVersion: "1.0.0" # 앱 버전
```

---

## 4. values.yaml — 기본 변수

templates/ 파일에서 `{{ .Values.xxx }}` 로 참조하는 기본값이다.

```yaml
replicaCount: 2

image:
  repository: <ECR_URL>/flaskapp
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: nginx                  # On-prem Nginx Ingress Controller
  host: flaskapp.onprem.local

config:
  DATABASE_HOST: "172.16.43.160"    # mariadb-1 VM IP (VLAN 30)
  DATABASE_USER: "kosa"
  DATABASE_DB_NAME: "employees"
  AWS_DEFAULT_REGION: "ap-northeast-2"
  PHOTOS_BUCKET: "flaskapp-proddata-kosa-project-team3-snow-lai9z"

secret:
  DATABASE_PASSWORD: "<CHANGE_ME>"  # 실제 배포 시 교체
  FLASK_SECRET: "<CHANGE_ME>"

resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "200m"
```

---

## 5. Ingress 구조

외부 요청을 FlaskApp Service로 연결하는 역할을 한다.

```
사용자 요청 (flaskapp.onprem.local)
  ↓
Nginx Ingress Controller (MetalLB: 172.16.41.110)
  ↓
flaskapp-service (ClusterIP: 80)
  ↓
FlaskApp Pod
```

```yaml
# templates/ingress.yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}-ingress
  namespace: {{ .Release.Namespace }}
spec:
  ingressClassName: {{ .Values.ingress.className }}
  rules:
  - host: {{ .Values.ingress.host }}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: {{ .Release.Name }}-service
            port:
              number: {{ .Values.service.port }}
{{- end }}
```

---

## 6. 템플릿 변수 규칙

| 변수 | 의미 | 예시 결과 |
|------|------|-----------|
| `{{ .Release.Name }}` | helm install 시 지정한 이름 | `flaskapp` |
| `{{ .Release.Namespace }}` | 배포 namespace | `flaskapp-prod` |
| `{{ .Values.xxx }}` | values.yaml 값 참조 | `2`, `nginx`, ... |

---

## 7. 주요 명령어

### chart 생성 (초안)
```bash
helm create flaskapp
```

### 문법 검사
```bash
helm lint ./helm/flaskapp
```

### 렌더링 미리 보기 (배포 전 필수)
```bash
helm template flaskapp ./helm/flaskapp -n flaskapp-prod
```

### 배포 (On-prem)
```bash
helm upgrade --install flaskapp ./helm/flaskapp \
  -n flaskapp-prod \
  -f helm/flaskapp/values.yaml
```

### 배포 상태 확인
```bash
helm list -n flaskapp-prod
kubectl get all -n flaskapp-prod
```

### 삭제
```bash
helm uninstall flaskapp -n flaskapp-prod
```

---

## 8. Ingress 접속 확인 (클러스터 구성 후 추가 예정)

> 팀원 B의 Nginx Ingress Controller 설치 완료 후 아래 내용을 채운다.

```bash
# Ingress 상태 확인
kubectl get ingress -n flaskapp-prod

# 로컬 hosts 파일에 등록 후 접속 테스트
curl http://flaskapp.onprem.local/
```

---

## 9. 환경별 values 파일 (Day 11 예정)

On-prem / AWS DR 환경별로 달라지는 값만 별도 파일로 관리한다.

| 파일 | 용도 | 주요 차이 |
|------|------|-----------|
| `values.yaml` | 공통 기본값 | - |
| `values-onprem.yaml` | On-prem 오버라이드 | `ingressClassName: nginx`, MariaDB IP |
| `values-aws-dr.yaml` | AWS DR 오버라이드 | `ingressClassName: alb`, RDS endpoint |

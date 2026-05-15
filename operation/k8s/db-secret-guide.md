# FlaskApp K8s ConfigMap / Secret DB 연동 가이드

> 담당: 팀원 C (@ireneminhee)  
> 관련 이슈: #108

---

## 개요

FlaskApp이 MariaDB에 연결하려면 두 가지 K8s 리소스가 필요하다.

| 리소스 | 역할 | 담는 값 |
|--------|------|---------|
| ConfigMap | 비민감 설정 | DB 호스트, 유저명, DB명, AWS 리전, S3 버킷 |
| Secret | 민감 정보 | DB 비밀번호, Flask 세션 키 |

FlaskApp Pod는 이 두 리소스를 `envFrom`으로 읽어 환경변수로 사용한다.

---

## Helm values.yaml 설정 위치

`helm/flaskapp/values.yaml`에서 ConfigMap과 Secret 값을 관리한다.

```yaml
config:
  DATABASE_HOST: "172.16.43.160"      # MariaDB VM IP (VLAN 30)
  DATABASE_USER: "kosa"
  DATABASE_DB_NAME: "employees"
  AWS_DEFAULT_REGION: "ap-northeast-2"
  PHOTOS_BUCKET: "flaskapp-proddata-kosa-project-team3-snow-lai9z"

secret:
  DATABASE_PASSWORD: "<CHANGE_ME>"    # 실제 값은 kubectl로 직접 적용 (커밋 금지)
  FLASK_SECRET: "<CHANGE_ME>"         # 실제 값은 kubectl로 직접 적용 (커밋 금지)
```

> **보안 주의사항**: 이 레포지토리는 **Public**이므로 `values.yaml`에 실제 Secret 값을 절대 커밋하지 않는다.  
> Secret은 반드시 `kubectl` 명령어로 클러스터에 직접 적용한다.
>
> ```bash
> kubectl create secret generic flaskapp-secret \
>   -n flaskapp-prod \
>   --from-literal=DATABASE_PASSWORD=실제비번 \
>   --from-literal=FLASK_SECRET=임의문자열
> ```
>
> 향후 확장 시 Sealed Secrets 또는 AWS Secrets Manager + External Secrets Operator 도입을 권장한다.

---

## Helm 템플릿 구조

### ConfigMap (`helm/flaskapp/templates/configmap.yaml`)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
  namespace: {{ .Release.Namespace }}
data:
  DATABASE_HOST: {{ .Values.config.DATABASE_HOST | quote }}
  DATABASE_USER: {{ .Values.config.DATABASE_USER | quote }}
  DATABASE_DB_NAME: {{ .Values.config.DATABASE_DB_NAME | quote }}
  AWS_DEFAULT_REGION: {{ .Values.config.AWS_DEFAULT_REGION | quote }}
  PHOTOS_BUCKET: {{ .Values.config.PHOTOS_BUCKET | quote }}
```

### Secret (`helm/flaskapp/templates/secret.yaml`)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Release.Name }}-secret
  namespace: {{ .Release.Namespace }}
type: Opaque
stringData:
  DATABASE_PASSWORD: {{ .Values.secret.DATABASE_PASSWORD | quote }}
  FLASK_SECRET: {{ .Values.secret.FLASK_SECRET | quote }}
```

> `stringData`를 사용하면 base64 인코딩 없이 평문으로 작성할 수 있다.  
> K8s가 내부적으로 자동 인코딩해서 저장한다.

---

## Deployment envFrom 연동

`helm/flaskapp/templates/deployment.yaml`에서 `envFrom`으로 두 리소스를 한 번에 주입한다.

```yaml
envFrom:
  - configMapRef:
      name: {{ .Release.Name }}-config
  - secretRef:
      name: {{ .Release.Name }}-secret
```

Pod 안에서 환경변수로 읽힌다:

```bash
kubectl exec -n flaskapp-prod <pod명> -- env | grep DATABASE
# DATABASE_HOST=172.16.43.160
# DATABASE_USER=kosa
# DATABASE_DB_NAME=employees
# DATABASE_PASSWORD=***
```

---

## 네트워크 경로

MariaDB와 K8s 워커 노드는 동일한 VLAN 30 대역에 있어 직접 통신이 가능하다.

```
FlaskApp Pod (172.16.43.110~112, VLAN 30)
  ↓ DATABASE_HOST
MariaDB VM (172.16.43.160, VLAN 30)
```

---

## DB 사전 준비 사항 (팀원 D 담당)

FlaskApp이 정상 연결되려면 MariaDB 측에서 아래가 준비돼 있어야 한다.

- [ ] `employees` 데이터베이스 생성
- [ ] `employees` 테이블 스키마 생성
- [ ] `kosa` 유저 외부 접속 허용 (`GRANT ALL ON employees.* TO 'kosa'@'%'`)
- [ ] MariaDB binlog 활성화 (DMS CDC 복제용)

---

## 연동 확인 방법

ArgoCD sync 후 Pod가 Running 상태이면 DB 연결 성공이다.

```bash
# Pod 상태 확인
kubectl get pods -n flaskapp-prod

# DB 연결 로그 확인
kubectl logs -n flaskapp-prod <pod명>

# 브라우저에서 직원 목록 조회
http://flaskapp.onprem.local/
```

Pod가 `CrashLoopBackOff`면 DB 연결 실패 → 비번 또는 네트워크 확인.

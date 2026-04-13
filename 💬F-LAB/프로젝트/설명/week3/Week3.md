## 0. Helm 설계 구조

```
values.yaml                        (공통 기본값)
values.dev.yaml                    (dev override)
values.dev-secret.yaml             (dev 민감정보 — Git ignore)
     │
     ▼   helm install url-shortener ./url-shortener-chart \
         -f values.yaml -f values.dev.yaml -f values.dev-secret.yaml

┌─────────────────────────────────────────────────────────────┐
│                    Helm Template Engine                      │
│  templates/secret.yaml     →  {{ .Values.postgres.password | b64enc }}  │
│  templates/deployment.yaml →  {{ .Release.Name }}-api                   │
│  templates/ingress.yaml    →  {{- if .Values.ingress.enabled }}          │
└─────────────────────────────────────────────────────────────┘
     │
     ▼   kubectl apply (내부 자동 처리)

┌─────────────────────────────────────────────────────────────┐
│                  minikube 클러스터                             │
│   Ingress → Service → Deployment (API Pod x2)               │
│                  │                                           │
│   StatefulSet (postgres:15) → PVC 1Gi                       │
│   Secret: POSTGRES_PASSWORD, DATABASE_URL                   │
└─────────────────────────────────────────────────────────────┘
```

**폴더 구조**

```
url-shortener-chart/
├── Chart.yaml
├── values.yaml                  # 공통 기본값 (Git 커밋 가능)
├── values.dev.yaml              # dev 환경 override
├── values.prod.yaml             # prod 환경 override
├── values.dev-secret.yaml       # dev 민감정보 (Git ignore 필수)
├── values.prod-secret.yaml      # prod 민감정보 (Git ignore 필수)
├── .helmignore
├── charts/
└── templates/
    ├── _helpers.tpl
    ├── NOTES.txt
    ├── secret.yaml
    ├── postgresql-pvc.yaml
    ├── postgres-statefulset.yaml
    ├── postgres-svc.yaml
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

---

## 1. Helm이란?

kubectl apply의 문제점: 파일 7개를 순서대로 직접 실행, dev/prod 환경마다 YAML 복사, 롤백 불가.

Helm 방식: 단일 명령어로 전체 배포, `helm upgrade/rollback`으로 이력 관리.

`helm create url-shortener-chart` 로 스캐폴딩 생성 후 불필요 파일 삭제.

---

## 2. Chart.yaml

```yaml
apiVersion: v2
name: url-shortener-chart
description: URL Shortener EKS Platform Helm Chart
type: application
version: 0.1.0       # Chart 자체 버전 (구조 바뀔 때 올림)
appVersion: '1.0.0'  # 앱 버전 (Docker 이미지 태그 참고용)
```

---

## 3. values.yaml — 환경별 분리 전략

3계층 구조: `values.yaml` → `values.dev.yaml` → `values.dev-secret.yaml`

**values.yaml (공통)**

```yaml
api:
  replicas: 2
  port: 8000
  image:
    repository: url-shortener-api
    tag: v1
    pullPolicy: Never
  env:
    appEnv: production
    debug: 'false'
  readinessProbe:
    initialDelaySeconds: 5
    periodSeconds: 10
  livenessProbe:
    initialDelaySeconds: 15
    periodSeconds: 20

postgres:
  version: '15'
  user: postgres
  database: prod-db
  password: ""
  databaseUrl: ""
  storage: 1Gi
  storageClass: standard

ingress:
  enabled: true
  host: url-shortener.local
```

**values.dev.yaml**

```yaml
api:
  replicas: 1
  image:
    tag: v1
    pullPolicy: Never
  env:
    appEnv: development
    debug: 'true'
postgres:
  database: tempdb
  storage: 1Gi
ingress:
  enabled: true
  host: url-shortener.local
```

**values.dev-secret.yaml (Git ignore 필수)**

```yaml
postgres:
  password: "dpdlvldzm1@"
  databaseUrl: "postgresql://{{ .Values.postgres.user }}:{{ .Values.postgres.password | urlquery }}@{{ .Release.Name }}-postgres-svc:5432/{{ .Values.postgres.database }}"
```

---

## 4. templates/ — 주요 템플릿 함수

```
{{ .Release.Name }}                      # 릴리스 이름 (url-shortener)
{{ .Values.api.replicas }}               # values.yaml 값 참조
{{ .Values.api.env.debug | quote }}      # 문자열을 "따옴표"로 감쌈
{{ .Values.postgres.password | b64enc }} # Base64 인코딩
{{ tpl .Values.postgres.databaseUrl . }} # 값 안의 {{ }} 재평가
{{- if .Values.ingress.enabled }}        # 조건부 렌더링
```

**templates/secret.yaml**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Release.Name }}-secret
type: Opaque
data:
  POSTGRES_PASSWORD: {{ .Values.postgres.password | b64enc }}
  DATABASE_URL: {{ tpl .Values.postgres.databaseUrl . | b64enc }}
```

**templates/service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-svc
spec:
  selector:
    app: {{ .Release.Name }}-api
  ports:
    - protocol: TCP
      port: 80
      targetPort: {{ .Values.api.port }}
  type: ClusterIP
```

**templates/ingress.yaml**

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ .Release.Name }}-svc
                port:
                  number: 80
{{- end }}
```

---

## 5. helm 명령어 실습

**helm template (사전 검증)**

```bash
helm template url-shortener ./url-shortener-chart \
  -f ./url-shortener-chart/values.yaml \
  -f ./url-shortener-chart/values.dev.yaml \
  -f ./url-shortener-chart/values.dev-secret.yaml

# 특정 파일만
helm template url-shortener ./url-shortener-chart \
  -f ./url-shortener-chart/values.dev-secret.yaml \
  -s templates/secret.yaml
```

**helm install**

```bash
helm install url-shortener ./url-shortener-chart \
  -f ./url-shortener-chart/values.yaml \
  -f ./url-shortener-chart/values.dev.yaml \
  -f ./url-shortener-chart/values.dev-secret.yaml

helm list
kubectl get pods
```

**helm upgrade**

```bash
helm upgrade url-shortener ./url-shortener-chart \
  -f ./url-shortener-chart/values.yaml \
  -f ./url-shortener-chart/values.dev.yaml \
  -f ./url-shortener-chart/values.dev-secret.yaml \
  --set api.image.tag=v2

helm history url-shortener
```

**helm rollback**

```bash
helm rollback url-shortener 1
helm history url-shortener
```

**helm uninstall**

```bash
helm uninstall url-shortener
```

---

## 6. k8s/ → Helm 변환 대조표

|k8s/ 파일|Helm 파일|주요 변경사항|
|---|---|---|
|k8s/db/secret.yaml|templates/secret.yaml|이름 app-secret → {Release.Name}-secret, b64enc/tpl 사용|
|k8s/db/postgres-pvc.yaml|templates/postgresql-pvc.yaml|파일명 변경, storageClass/storage 값화|
|k8s/db/postgres-stateful.yaml|templates/postgres-statefulset.yaml|파일명 변경, version/user/database 값화|
|k8s/db/postgres-svc.yaml|templates/postgres-svc.yaml|이름 postgres-svc → {Release.Name}-postgres-svc|
|k8s/deployment.yaml|templates/deployment.yaml|replicas/image/env 모두 값화|
|k8s/service.yaml|templates/service.yaml|이름 url-shortener-svc → {Release.Name}-svc|
|k8s/ingress.yaml|templates/ingress.yaml|{{- if ingress.enabled }} 조건 추가|
|(없음)|values.yaml / values.dev.yaml / values.prod.yaml|환경별 값 분리|
|(없음)|values.dev-secret.yaml / values.prod-secret.yaml|민감정보 분리 (Git ignore)|
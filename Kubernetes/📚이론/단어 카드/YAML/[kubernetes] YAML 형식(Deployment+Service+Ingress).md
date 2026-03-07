> 흐름

```
사용자 → Ingress → Service → Pod(Deployment가 관리)
```

---

# 1. Deployment

```yaml
# ─────────────────────────────────────────
# Deployment: Pod를 관리하고 원하는 개수만큼 유지
# ─────────────────────────────────────────
apiVersion: apps/v1          # 사용할 쿠버네티스 API 버전
kind: Deployment             # 리소스 종류 (Deployment)

metadata:
  name: my-app-deployment    # Deployment 이름
  labels:
    app: my-app              # 리소스 식별용 라벨

spec:
  replicas: 3                # 실행할 Pod 개수 (3개 유지)

  selector:
    matchLabels:
      app: my-app            # 이 라벨을 가진 Pod들을 Deployment가 관리

  template:                  # 실제 생성될 Pod 템플릿
    metadata:
      labels:
        app: my-app          # Pod에 붙는 라벨 (Service가 찾기 위해 필요)

    spec:
      containers:
        - name: my-app-container    # 컨테이너 이름
          image: nginx:latest       # 사용할 컨테이너 이미지

          ports:
            - containerPort: 80     # 컨테이너 내부 포트
```

---

# 2. Service

```yaml
# ─────────────────────────────────────────
# Service: Pod에 접근하기 위한 내부 네트워크 엔드포인트
# ─────────────────────────────────────────
apiVersion: v1
kind: Service

metadata:
  name: my-app-service       # Service 이름

spec:
  type: ClusterIP            # 클러스터 내부에서만 접근 가능한 서비스

  selector:
    app: my-app              # 이 라벨을 가진 Pod로 트래픽 전달

  ports:
    - port: 80               # Service가 사용하는 포트
      targetPort: 80         # Pod 컨테이너 포트
      protocol: TCP
```

---

# 3. Ingress

```yaml
# ─────────────────────────────────────────
# Ingress: 외부 트래픽을 Service로 라우팅
# ─────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
  name: my-app-ingress       # Ingress 이름

spec:
  rules:
    - host: myapp.example.com     # 접속 도메인

      http:
        paths:
          - path: /               # 요청 경로
            pathType: Prefix

            backend:
              service:
                name: my-app-service   # 연결할 Service
                port:
                  number: 80
```

---

# 전체 구조

```
사용자 요청
     │
     ▼
Ingress
     │
     ▼
Service
     │
     ▼
Pod1   Pod2   Pod3
 (Deployment이 관리)
```

---

# 실제 요청 흐름

1️⃣ 사용자가 접속

```
http://myapp.example.com
```

2️⃣ **Ingress가 요청을 받음**

```
myapp.example.com → my-app-service
```

3️⃣ **Service가 Pod 선택**

```
my-app-service
 ├ Pod1
 ├ Pod2
 └ Pod3
```

4️⃣ **로드밸런싱 후 Pod로 전달**

```
Client → Ingress → Service → Pod
```

---

# 이 구조가 가장 많이 쓰이는 이유

### 1️⃣ Deployment

Pod 생성 / 유지 / 업데이트 관리

### 2️⃣ Service

Pod들을 묶어서 **고정 네트워크 제공**

### 3️⃣ Ingress

외부 트래픽을 **도메인 기반으로 라우팅**

---

# 면접에서 한 줄 정리

```
Deployment는 Pod를 관리하고  
Service는 Pod에 대한 내부 네트워크 엔드포인트를 제공하며  
Ingress는 외부 트래픽을 Service로 라우팅하는 역할을 한다.
```

---

원하면 **이 YAML에서 면접에서 진짜 많이 물어보는 것 7개**도 정리해줄게.

예를 들면

- 왜 Pod를 직접 만들지 않고 Deployment를 쓰는가
    
- Service 없이 Pod 접근하면 왜 문제인가
    
- [[Ingress 없이 LoadBalancer만 쓰면 어떻게 되는가]]
    
- [[selector와 label이 왜 중요한가]]
    
- [[containerPort vs port vs targetPort 차이]]
    

이건 **쿠버네티스 면접에서 거의 80% 확률로 나오는 질문**이다.
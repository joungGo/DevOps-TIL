**결론:**  
Ingress 없이 **Service type = LoadBalancer**만 사용하면 **외부 → Pod 접근은 가능하지만, 라우팅 기능이 거의 없다.**

---

# 동작 구조

```text
Client
  │
  ▼
LoadBalancer (외부 IP 생성)
  │
  ▼
Service
  │
  ▼
Pod
```

---

# 어떻게 동작하는가

1. Service를 `LoadBalancer` 타입으로 생성
    
2. 클라우드가 **외부 LoadBalancer 생성**
    
3. **외부 IP 할당**
    
4. 요청 흐름
    

```
Client
 ↓
LoadBalancer
 ↓
Service
 ↓
Pod
```

---

# 문제점 (Ingress 없이 LB만 쓸 때)

### 1️⃣ 서비스마다 LoadBalancer 필요

예

```
Service A → LoadBalancer A
Service B → LoadBalancer B
Service C → LoadBalancer C
```

→ **LoadBalancer가 계속 늘어남**

---

### 2️⃣ 비용 증가 (클라우드)

예:

- AWS ELB
    
- GCP LB
    
- Azure LB
    

**서비스 하나마다 비용 발생**

---

### 3️⃣ URL 기반 라우팅 불가

Ingress가 있으면

```
example.com/api → api-service
example.com/user → user-service
```

하지만 LoadBalancer만 있으면

```
api.example.com
user.example.com
```

처럼 **서비스마다 별도 LB 필요**

---

### 4️⃣ TLS 관리 어려움

Ingress 사용

```
Ingress Controller
 └ TLS 관리
 └ 인증서 관리
```

LB만 사용

```
각 서비스마다 TLS 설정 필요
```

---

# Ingress 사용하는 구조

```
Client
   │
   ▼
LoadBalancer (1개)
   │
   ▼
Ingress Controller
   │
 ┌─┴───────────────┐
 ▼                 ▼
Service A       Service B
 ▼                 ▼
Pod               Pod
```

---

# 핵심 비교

|방식|특징|
|---|---|
|LoadBalancer만 사용|서비스마다 LB 생성|
|Ingress 사용|**LB 1개로 여러 서비스 관리 가능**|

---

# 면접용 한 줄

```
LoadBalancer만 사용하면 서비스마다 외부 LoadBalancer가 생성되지만,
Ingress를 사용하면 하나의 LoadBalancer로 여러 서비스를 라우팅할 수 있다.
```
## 1. 왜 NodePort는 운영에 부적합한가

NodePort의 한계부터 짚고 가면 이해가 쉬워.

- 포트가 `30000~32767`으로 고정 → URL 보기 안 좋음
    
- 모든 노드에 포트가 열림 → 보안 취약
    
- 서비스가 많아질수록 포트 관리 지옥
    
- HTTPS, 도메인, 라우팅 처리 어려움
    

👉 그래서 **운영 환경에서는 거의 사용하지 않음**

---

## 2. LoadBalancer Service

### 개념

- **클라우드 로드밸런서(AWS ELB, GCP LB 등)** 를 자동 생성
    
- 외부에서 접근 가능한 **고정 공인 IP** 제공
    

### 특징

- 외부 트래픽을 여러 노드/파드로 분산
    
- 내부적으로는 NodePort + ClusterIP 포함
    
- 클라우드 환경에서만 실질적으로 의미 있음
    

### 흐름

```scss
외부 사용자    
	↓ 
Cloud LoadBalancer (Public IP)    
	↓ 
NodePort    
	↓ 
Service (ClusterIP)    
	↓ 
Pod
```

### Service 예시

```yaml
apiVersion: v1              # Kubernetes API 버전 (Service는 v1 사용)
kind: Service               # 생성할 리소스 종류 (Service)
metadata:
  name: nginx-lb            # Service 이름
spec:
  type: LoadBalancer        # 외부에서 접근 가능한 로드밸런서 타입
  selector:
    app: nginx              # app=nginx 라벨을 가진 Pod와 연결
  ports:
  - port: 80                # Service가 외부에 노출하는 포트
    targetPort: 80          # Pod(컨테이너) 내부에서 실제로 사용하는 포트
```

### 언제 쓰나

- **단일 서비스**를 외부에 바로 노출할 때
    
- Ingress 없이 간단히 운영할 경우
    

---

## 3. Ingress

### 개념

- **HTTP/HTTPS 트래픽 라우팅 규칙 모음**
    
- 실제 트래픽 처리는 **Ingress Controller**가 담당
    

> Ingress = 규칙  
> Ingress Controller = 실제 동작하는 컴포넌트 (nginx, traefik 등)

---

### 왜 Ingress가 필요한가

LoadBalancer만 쓰면 이런 문제가 생김:

- 서비스마다 LoadBalancer 생성 → 비용 증가
    
- 도메인/경로 기반 라우팅 불가
    
- TLS 인증서 관리 어려움
    

👉 Ingress로 해결

---

### Ingress가 하는 일

- 도메인 기반 라우팅
    
    - `api.example.com` → API 서비스
        
    - `www.example.com` → 웹 서비스
        
- 경로 기반 라우팅
    
    - `/api` → backend
        
    - `/` → frontend
        
- TLS(HTTPS) 처리
    

---

### Ingress 트래픽 흐름

```scss
외부 사용자
   ↓
LoadBalancer (고정 IP)
   ↓
Ingress Controller
   ↓
Service (ClusterIP)
   ↓
Pod
```

---

## 4. Ingress 예시

```yaml
apiVersion: networking.k8s.io/v1   # Ingress 리소스에서 사용하는 API 버전
kind: Ingress                     # 생성할 리소스 종류 (Ingress)
metadata:
  name: web-ingress               # Ingress 리소스 이름
spec:
  rules:
  - host: www.example.com         # 이 도메인으로 들어오는 요청에만 적용
    http:
      paths:
      - path: /                   # 요청 URL 경로 (/로 시작하는 모든 경로)
        pathType: Prefix          # /로 시작하는 모든 경로를 매칭
        backend:
          service:
            name: frontend-svc    # 트래픽을 전달할 Service 이름
            port:
              number: 80          # Service에서 사용할 포트
```

---

## 5. 왜 “LoadBalancer + Ingress” 조합인가

### 역할 분리

|구성요소|역할|
|---|---|
|LoadBalancer|외부 고정 IP, L4 트래픽 수신|
|Ingress Controller|L7(HTTP/HTTPS) 라우팅|
|Service|파드 집합 추상화|
|Pod|실제 애플리케이션|

---

## 6. 운영 환경 정석 구조

```scss
[ Internet ]
     ↓
[ Cloud LoadBalancer ]
     ↓
[ Ingress Controller ]
     ↓
[ ClusterIP Services ]
     ↓
[ Pods ]
```
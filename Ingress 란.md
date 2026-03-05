> [!note] 면접 답변
> Ingress는 외부에서 들어오는 HTTP/HTTPS 요청을 도메인이나 경로 기반으로 여러 Service에 라우팅하는 Kubernetes 리소스이며, ClusterIP는 클러스터 내부 Pod 간 통신을 위해 Service에 할당되는 가상 IP이다.

---
## 1. Ingress란 무엇인가

**Ingress는 외부에서 클러스터 내부 서비스로 들어오는 HTTP/HTTPS 트래픽을 관리하는 리소스이다.**

쉽게 말하면

```
외부 요청 → 어떤 Service로 보낼지 결정하는 라우터
```

예를 들어 이런 요청이 있다고 가정하자.

```
example.com/api
example.com/admin
```

Ingress는 이를 다음처럼 라우팅할 수 있다.

```
example.com/api   → api-service
example.com/admin → admin-service
```

즉 Ingress의 역할은

- HTTP/HTTPS 트래픽 라우팅
    
- 여러 서비스 하나의 도메인으로 제공
    
- TLS(HTTPS) 관리
    

---

### Ingress 동작 구조

중요한 점은 **Ingress 자체는 실제로 트래픽을 처리하지 않는다.**

실제 트래픽 처리는 **Ingress Controller**가 담당한다.

구조

```
Internet
   ↓
Ingress Controller
   ↓
Service
   ↓
Pod
```

대표적인 Ingress Controller

- NGINX Ingress Controller
    
- Traefik
    
- HAProxy
    
- Kong
    

---

### Ingress 예시

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

의미

```
example.com/api → api-service
```

---

## 2. Ingress 말고 다른 것도 있나

Ingress는 **L7(HTTP/HTTPS) 라우팅** 방식이다.

외부 트래픽을 노출하는 대표적인 방법은 3가지다.

|방식|특징|
|---|---|
|NodePort|Node 포트로 직접 접근|
|LoadBalancer|클라우드 로드밸런서 생성|
|Ingress|HTTP/HTTPS 라우팅|

구조 비교

### NodePort

```
Client
  ↓
NodeIP:Port
  ↓
Service
  ↓
Pod
```

---

### LoadBalancer

```
Client
  ↓
External Load Balancer
  ↓
Service
  ↓
Pod
```

---

### Ingress

```
Client
  ↓
Ingress Controller
  ↓
Service
  ↓
Pod
```


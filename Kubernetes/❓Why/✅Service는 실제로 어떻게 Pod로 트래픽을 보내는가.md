> [!note] 면접 답변
> Service는 실제로 트래픽을 전달하는 컴포넌트가 아니라 **ClusterIP라는 가상 IP를 제공하는 추상화 계층**입니다.  
실제 트래픽 전달은 **kube-proxy가 각 노드에 생성한 iptables 또는 IPVS 규칙을 통해 Service IP를 Pod IP로 NAT 변환하면서 이루어집니다.**

---
### Service는 실제로 어떻게 Pod로 트래픽을 보내는가

쿠버네티스에서 Service가 Pod로 트래픽을 보내는 과정에는 **kube-proxy**가 중요한 역할을 한다.

핵심 흐름

```text
Pod → Service(ClusterIP) → kube-proxy → Pod
```

Service 자체는 **실제 네트워크 장비가 아니다.**  
단순히 **논리적인 개념(Virtual IP)** 이다.

실제 트래픽 전달은 **각 Node에서 실행되는 kube-proxy가 처리한다.**

---

# 1. 전체 동작 흐름

예시

Service

```
payment-service
ClusterIP: 10.96.0.15
```

Pod

```
payment-pod-1 10.244.1.5
payment-pod-2 10.244.2.8
payment-pod-3 10.244.3.2
```

요청 흐름

```
Order Pod
   ↓
10.96.0.15 (Service)
   ↓
kube-proxy
   ↓
Pod 중 하나 선택
```

즉

```
ClusterIP는 가상의 IP
실제 라우팅은 kube-proxy가 수행
```

---

# 2. kube-proxy 역할

kube-proxy는 **각 Node에서 실행되는 네트워크 컴포넌트**다.

역할

1. Service 생성 감지
    
2. Endpoint 목록 확인
    
3. 네트워크 규칙 생성
    

즉

```
Service → Pod 연결 규칙 생성
```

---

# 3. iptables 방식 (가장 많이 사용)

kube-proxy는 **iptables 규칙**을 만들어 트래픽을 Pod로 보낸다.

iptables는 리눅스의 **패킷 라우팅 규칙 시스템**이다.

예

```
10.96.0.15:80 → Pod1
10.96.0.15:80 → Pod2
10.96.0.15:80 → Pod3
```

실제 규칙 예

```
ClusterIP: 10.96.0.15
↓
iptables rule
↓
10.244.1.5
10.244.2.8
10.244.3.2
```

요청이 오면

```
10.96.0.15 → iptables → Pod
```

---

# 4. 실제 트래픽 흐름

전체 흐름

```
Order Pod
   ↓
DNS 조회
   ↓
payment-service → 10.96.0.15
   ↓
Node network
   ↓
iptables (kube-proxy)
   ↓
Pod 선택
   ↓
Payment Pod
```

---

# 5. kube-proxy 동작 모드

kube-proxy는 3가지 방식이 있다.

|모드|설명|
|---|---|
|userspace|초기 방식 (거의 사용 안함)|
|iptables|가장 일반적|
|ipvs|대규모 환경에서 사용|

차이

### iptables

```
iptables rule 기반 라우팅
```

장점

- 단순
    
- 안정적
    

단점

- Pod가 많아지면 규칙이 많아짐
    

---

### ipvs

Linux의 **IP Virtual Server** 사용

특징

```
더 빠른 로드밸런싱
대규모 트래픽에 유리
```

---

# 6. 매우 중요한 개념 (Service는 실제 서버가 아니다)

많이 하는 오해

```
Service = 서버
```

실제

```
Service = 가상의 IP
```

트래픽 흐름

```
Client Pod
   ↓
ClusterIP
   ↓
kube-proxy
   ↓
Pod
```

즉

```
Service는 네트워크 규칙일 뿐이다.
```

---

# 7. 전체 구조 한 번에 보기

```
Pod (Order)
   ↓
DNS (CoreDNS)
   ↓
Service name
   ↓
ClusterIP
   ↓
kube-proxy
   ↓
iptables / ipvs
   ↓
Pod (Payment)
```

---

# 한 줄 정리

Service는 실제 서버가 아니라 **ClusterIP라는 가상 IP이고, kube-proxy가 iptables/ipvs 규칙을 이용해 Pod로 트래픽을 전달한다.**

---

쿠버네티스를 제대로 이해하려면 여기서 **아주 중요한 질문이 하나 더 나온다.**

"**[[✅노드가 다른 Pod간 통신이 어떻게 가능할까]]?**"


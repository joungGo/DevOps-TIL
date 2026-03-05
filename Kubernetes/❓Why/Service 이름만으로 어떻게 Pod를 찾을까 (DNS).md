> [!note] 면접 답변
> Service 이름으로 통신하면 Pod가 직접 찾아지는 것이 아니라, **CoreDNS가 Service 이름을 Service의 ClusterIP로 변환하고, 이후 kube-proxy가 해당 Service에 연결된 Pod들로 트래픽을 로드밸런싱하여 전달한다.**

---
### Service 이름만으로 Pod를 찾는 이유 (DNS)

쿠버네티스에는 **내부 DNS 서버**가 있다.

이 DNS는 **CoreDNS**라는 컴포넌트가 담당한다.

쿠버네티스 클러스터 내부에서는 다음과 같은 규칙이 있다.

```
서비스이름 → ClusterIP
```

예

Service

```
name: payment-service
ClusterIP: 10.96.0.15
```

그러면 DNS에 자동으로 등록된다.

```
payment-service → 10.96.0.15 (변환)
```

---

### 실제 동작 과정

Order Pod가 Payment 서비스를 호출한다고 가정한다.

코드

```
http://payment-service/pay
```

동작 과정

```
1. Order Pod가 payment-service DNS 조회
2. CoreDNS가 ClusterIP 반환
3. 요청이 Service로 전달
4. Service가 Pod 중 하나로 전달
```

흐름

```
Order Pod
   ↓
DNS 조회 (CoreDNS)
   ↓
payment-service → 10.96.0.15
   ↓
Service
   ↓
Payment Pod
```

---

### 실제 DNS 구조

쿠버네티스 DNS는 보통 이런 형태다.

```
서비스이름.네임스페이스.svc.cluster.local
```

예

```
payment-service.default.svc.cluster.local
```

하지만 같은 네임스페이스라면 **서비스 이름만 써도 된다.**

```
http://payment-service
```

---

### 한 줄 정리

Service 이름으로 Pod를 찾는 이유는 **쿠버네티스 내부 DNS(CoreDNS)가 서비스 이름을 ClusterIP로 변환해주기 때문**이다.

---

쿠버네티스를 처음 배울 때 여기서 **많이 헷갈리는 중요한 질문**이 있다.

"[[Service는 Pod를 어떻게 찾을까]]?"

"[[Pod는 CoreDNS의 IP를 어떻게 알고 있을까]]"


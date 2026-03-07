### 1. ClusterIP란

**ClusterIP**는 쿠버네티스에서 가장 기본적인 **Service 타입**이다.

Pod들은 IP가 계속 바뀌기 때문에 직접 접근하기 어렵다.  
그래서 **Pod 앞에 Service를 두고 고정된 IP로 접근하도록 만든다.**

이때 생성되는 **클러스터 내부 전용 IP**가 **ClusterIP**다.

핵심 특징

- 클러스터 **내부에서만 접근 가능**
    
- 외부 인터넷에서는 접근 불가능
    
- 여러 Pod에 **로드밸런싱** 수행
    

---

### 2. 왜 필요한가

Pod의 문제

- Pod는 재시작되면 **IP가 바뀐다**
    
- 여러 Pod가 있을 수 있다
    
- 직접 접근하면 안정적이지 않다
    

그래서

```
Client Pod → Service(ClusterIP) → 여러 Pod
```

Service가 **고정된 주소 역할**을 한다.

---

### 3. 동작 구조

예시

```
Pod A   10.244.1.3
Pod B   10.244.1.4
Pod C   10.244.1.5
```

Service 생성

```
Service: my-service
ClusterIP: 10.96.0.10
```

접근 흐름

```
요청
  ↓
10.96.0.10 (ClusterIP)
  ↓
kube-proxy
  ↓
Pod A / Pod B / Pod C 중 하나
```

kube-proxy가 **[[라운드 로빈 방식]]으로 Pod에 전달**한다.

---

### 4. YAML 예시

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

의미

|항목|설명|
|---|---|
|type|Service 타입|
|selector|어떤 Pod와 연결할지|
|port|Service 포트|
|targetPort|Pod 포트|

---

### 5. 실제 접근 예

같은 클러스터 내부 Pod에서

```
curl http://my-service
```

또는

```
curl http://10.96.0.10
```

---

### 6. 언제 사용하나

ClusterIP는 보통 **마이크로서비스 내부 통신**에 사용된다.

예

```
frontend → backend
backend → database
backend → auth-service
```

외부 사용자 접근이 필요하면 다음을 사용한다.

|Service 타입|용도|
|---|---|
|ClusterIP|내부 통신|
|NodePort|외부 접근|
|LoadBalancer|클라우드 로드밸런서|
> [!info] 마이크로서비스 아키텍처 (Microservice Architecture)
> **마이크로서비스 아키텍처(MSA)**에서는 하나의 큰 애플리케이션을 **여러 개의 작은 서비스로 나누어** 구성한다.  
> 각 서비스는 **독립적으로 동작하며 개별 기능을 담당**한다.
>
> **예시 (쇼핑몰 시스템)**
>
> - 사용자 서비스
> - 주문 서비스
> - 결제 서비스
> - 상품 서비스
>
> 각 기능이 **독립적인 서비스**로 실행된다.
>
> **전체 구조 예**
>
> ```
> Frontend
>    ↓
> API Gateway
>    ↓
> Order Service → Payment Service
>             → Product Service
>             → User Service
> ```
>
> 이때 **서비스끼리는 API를 통해 서로 통신**한다.
>
> **예시**
>
> ```
> Order Service → Payment Service
> POST /payments
> ```
>
> **쿠버네티스에서의 서비스 간 통신**
>
> ```
> Order Pod
>    ↓
> payment-service (ClusterIP)
>    ↓
> Payment Pod
> ```
>
> 즉 **서비스 이름(Service DNS)을 사용하여 내부 API 호출**을 수행한다.
>
> **예시 호출**
>
> ```
> http://payment-service/pay
> ```
>
> 이것을 **마이크로서비스 간 내부 통신(Internal Service Communication)**이라고 한다.


---

### 7. 한 줄 정리

ClusterIP는 **쿠버네티스 클러스터 내부에서 Pod들을 안정적으로 접근하기 위한 내부 서비스 IP**다.

---

원하면 다음도 이어서 설명해 줄 수 있다.

- NodePort
    
- LoadBalancer
    
- Service 내부 동작 (kube-proxy, iptables)
    
- ClusterIP가 실제로 어떻게 라우팅되는지 (네트워크 레벨)
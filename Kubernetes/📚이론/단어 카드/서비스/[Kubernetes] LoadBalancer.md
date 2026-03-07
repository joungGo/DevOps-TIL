## LoadBalancer

**LoadBalancer는 클라우드 환경에서 외부 로드밸런서를 자동으로 생성하여 클러스터 외부에서 서비스에 접근할 수 있도록 하는 Service 타입이다.**

즉 사용자는 다음과 같은 방식으로 접근한다.

```
External IP:Port
```

---

# 1. 동작 구조

LoadBalancer Service를 생성하면 쿠버네티스는 **클라우드 제공자의 로드밸런서를 자동으로 생성한다.**

예

```
AWS → ELB
GCP → Cloud Load Balancer
Azure → Azure Load Balancer
```

요청 흐름

```
Client
  ↓
External Load Balancer
  ↓
NodePort
  ↓
Service (ClusterIP)
  ↓
Pod
```

즉 내부적으로는 다음 구조다.

```
External LB
   ↓
NodePort
   ↓
ClusterIP Service
   ↓
Pod
```

---

# 2. LoadBalancer 생성 예시

YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  type: LoadBalancer
  selector:
    app: payment
  ports:
    - port: 80
      targetPort: 8080
```

Service 생성 후 확인

```
kubectl get svc
```

예

```
NAME              TYPE           CLUSTER-IP     EXTERNAL-IP       PORT
payment-service   LoadBalancer   10.96.0.15     34.120.12.55      80
```

외부 접근

```
http://34.120.12.55
```

---

# 3. 내부 동작 과정

LoadBalancer Service 생성 시 발생하는 일

1. Service 생성
    

```
type: LoadBalancer
```

2. Kubernetes Cloud Controller Manager가 감지
    
3. 클라우드 API 호출
    
4. 외부 Load Balancer 생성
    
5. NodePort와 연결
    

전체 흐름

```
Service 생성
   ↓
Cloud Controller Manager
   ↓
Cloud Load Balancer 생성
   ↓
NodePort 연결
   ↓
Pod
```

---

# 4. LoadBalancer 특징

|특징|설명|
|---|---|
|외부 접근 가능|External IP 제공|
|클라우드 LB 사용|AWS ELB 등|
|자동 생성|Service 생성 시|
|내부적으로 NodePort 사용|트래픽 전달|

---

# 5. NodePort와 차이

|구분|NodePort|LoadBalancer|
|---|---|---|
|접근 방식|NodeIP:Port|External IP|
|로드밸런서|없음|있음|
|사용자 접근성|낮음|높음|
|운영 환경|테스트|실제 서비스|

---

# 6. 실제 운영 구조

대부분의 클라우드 환경은 다음 구조를 사용한다.

```
Internet
   ↓
LoadBalancer
   ↓
Ingress Controller
   ↓
Service (ClusterIP)
   ↓
Pod
```

이 구조를 사용하는 이유

- 여러 서비스 관리 가능
    
- path 기반 라우팅
    
- TLS 관리
    

"**[[Ingress 란]]**"

---

# 한 줄 면접 정리

**LoadBalancer Service는 클라우드 환경에서 외부 로드밸런서를 자동으로 생성하여 External IP를 통해 클러스터 외부에서 Pod에 접근할 수 있도록 하는 Service 타입이다.**
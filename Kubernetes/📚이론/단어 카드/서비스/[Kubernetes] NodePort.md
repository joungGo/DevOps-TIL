### NodePort

**NodePort는 Service 타입 중 하나로, 클러스터 외부에서 Node의 IP와 특정 포트를 통해 Pod에 접근할 수 있도록 하는 방식이다.**

즉 외부 사용자가 다음과 같은 방식으로 서비스에 접근할 수 있다.

```
NodeIP:NodePort
```

---

## 1. 동작 구조

NodePort Service를 생성하면 쿠버네티스는 **모든 Node에 동일한 포트를 열어준다.**

예

```
Node IP     : 192.168.0.10
NodePort    : 30080
Service     : payment-service
Pod         : 10.244.1.5
```

요청 흐름

```
Client
  ↓
192.168.0.10:30080
  ↓
NodePort
  ↓
Service (ClusterIP)
  ↓
kube-proxy
  ↓
Pod
```

즉 실제 내부 흐름은

```
외부 요청
 → NodePort
 → ClusterIP Service
 → Pod
```

---

## 2. NodePort 특징

|특징|설명|
|---|---|
|외부 접근 가능|Node IP로 접근|
|포트 범위|기본 30000~32767|
|모든 Node에 포트 개방|어느 Node로 접근해도 서비스 가능|
|내부적으로 ClusterIP 사용|NodePort는 ClusterIP 위에 동작|

---

## 3. 예시 YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  type: NodePort
  selector:
    app: payment
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080
```

의미

```
Service Port : 80
Pod Port     : 8080
Node Port    : 30080
```

외부 접근

```
http://NodeIP:30080
```

---

## 4. 왜 모든 Node에서 접근 가능할까

kube-proxy가 **각 Node에 iptables/ipvs 규칙을 생성하기 때문이다.**

따라서 요청이 들어오면

```
Node
 ↓
kube-proxy
 ↓
Pod 선택
```

Pod가 다른 Node에 있어도 자동으로 전달된다.


---

#### 왜 사용할까?



---

## 5. NodePort의 한계

실제 운영에서는 NodePort만으로 서비스를 공개하는 경우는 많지 않다.

이유

1. 포트 번호가 높다
    

```
30080
31234
```

2. 로드밸런서 기능이 제한적이다
    
3. Node IP를 직접 사용해야 한다
    

그래서 보통 다음 구조를 사용한다.

```
Internet
 ↓
LoadBalancer / Ingress
 ↓
NodePort
 ↓
Service
 ↓
Pod
```

---

### 한 줄 면접 정리

**NodePort는 Node의 특정 포트를 외부에 개방하여 NodeIP:NodePort 방식으로 클러스터 외부에서 Pod에 접근할 수 있도록 하는 Service 타입이다.**
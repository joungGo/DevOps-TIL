## ExternalName

**ExternalName은 Kubernetes Service 타입 중 하나로, 클러스터 내부 서비스 이름을 외부 도메인 이름에 매핑하기 위한 Service이다.**

즉 Kubernetes 내부에서 **외부 서비스에 접근할 때 DNS 별칭(alias)을 제공하는 역할**을 한다.

---

## 1. 동작 방식

ExternalName Service는 **IP를 생성하지 않는다.**

대신 DNS에서 다음과 같이 변환된다.

예

```text
Service 이름        → 실제 외부 도메인
payment-service     → api.payment.com
```

Pod에서 요청

```text
http://payment-service
```

DNS 변환

```text
payment-service → api.payment.com
```

즉

```text
Pod
 ↓
Kubernetes DNS
 ↓
External Domain
```

---

## 2. 예시 YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-payment
spec:
  type: ExternalName
  externalName: api.payment.com
```

의미

```text
external-payment → api.payment.com
```

Pod에서 접근

```text
http://external-payment
```

실제로는

```text
http://api.payment.com
```

으로 연결된다.

---

## 3. 특징

|특징|설명|
|---|---|
|IP 없음|ClusterIP 생성 안함|
|DNS 별칭|CNAME 레코드 생성|
|외부 서비스 연결|클러스터 외부 서비스 사용|
|kube-proxy 사용 안함|단순 DNS 변환|

---

## 4. 사용 예

예를 들어 회사 시스템이 다음 구조라고 가정한다.

```text
Kubernetes
   |
   | → 내부 서비스
   |
   | → 외부 DB
   | → 외부 API
```

ExternalName 사용

```text
db-service → company-db.example.com
payment-service → payment-api.example.com
```

Pod에서는 내부 서비스처럼 사용 가능하다.

```text
http://payment-service
```

---

## 5. 다른 Service 타입과 차이

|Service 타입|역할|
|---|---|
|ClusterIP|클러스터 내부 통신|
|NodePort|Node 포트로 외부 접근|
|LoadBalancer|외부 로드밸런서 생성|
|ExternalName|외부 도메인 연결|

---

## 면접용 한 줄 정리

**ExternalName Service는 Kubernetes 내부 서비스 이름을 외부 도메인 이름에 매핑하여 Pod가 외부 서비스를 내부 서비스처럼 사용할 수 있도록 하는 DNS 기반 Service 타입이다.**
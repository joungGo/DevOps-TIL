## ExternalName Service

**ExternalName Service**는 Kubernetes 클러스터 내부에서 **외부 서비스의 DNS 이름을 사용하도록 연결해 주는 Service 타입**이다.  
즉, **클러스터 밖에 있는 서비스를 내부 서비스처럼 접근할 수 있게 해준다.**

---

# 1. ExternalName Service 의미

보통 Service는 **Pod에 연결**된다.

하지만 **ExternalName Service는 Pod와 연결되지 않고 외부 도메인으로 연결된다.**

구조

```
Pod → Service → 외부 DNS
```

예

```
Pod → external-service → google.com
```

즉 Pod에서 `external-service`로 접속하면 **실제로는 google.com으로 연결된다.**

---

# 2. Service 타입 중 하나

Kubernetes Service에는 여러 타입이 있다.

|Service 타입|설명|
|---|---|
|ClusterIP|클러스터 내부 접근|
|NodePort|노드 IP로 외부 접근|
|LoadBalancer|외부 로드밸런서|
|ExternalName|외부 DNS 연결|

---

# 3. ExternalName Service 예시

YAML 예

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  type: ExternalName
  externalName: google.com
```

설명

|항목|의미|
|---|---|
|type|ExternalName Service|
|externalName|연결할 외부 도메인|

---

# 4. 동작 방식

Pod에서

```
http://external-service
```

접속하면

DNS가 다음처럼 변환된다.

```
external-service → google.com
```

즉 실제 연결

```
Pod → google.com
```

---

# 5. 특징

ExternalName Service 특징

- Pod와 연결되지 않음
    
- DNS 이름만 매핑
    
- ClusterIP 없음
    
- 외부 서비스 연결용
    

---

# 6. 사용 예

ExternalName은 다음 상황에서 사용된다.

예

- 외부 데이터베이스
    
- 외부 API 서버
    
- 다른 클라우드 서비스
    

예

```
Pod → mysql-service → 외부 MySQL 서버
```

---

# 핵심 정리

ExternalName Service는 Kubernetes 클러스터 내부에서 외부 서비스의 DNS 이름을 사용할 수 있도록 연결해 주는 Service 타입이다. Pod와 직접 연결되지 않으며 DNS 이름을 외부 도메인으로 매핑하여 외부 서비스에 접근할 수 있도록 한다.
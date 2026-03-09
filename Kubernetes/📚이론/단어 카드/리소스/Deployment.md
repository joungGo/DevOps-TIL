## Deployment

Deployment는 **애플리케이션 Pod를 생성하고 관리하는 Kubernetes 컨트롤러 객체**이다.  
ReplicaSet을 이용하여 **Pod의 생성, 업데이트, 롤백 등을 관리한다.**

즉 Deployment는 **Pod의 배포와 업데이트를 관리하는 객체**이다.

---

## 1. Deployment의 역할

Deployment는 다음 기능을 제공한다.

- Pod 생성 및 관리
    
- Pod 개수 유지
    
- 롤링 업데이트
    
- 이전 버전으로 롤백
    

구조

```text
Deployment
   ↓
ReplicaSet
   ↓
Pod
```

Deployment가 ReplicaSet을 만들고, ReplicaSet이 Pod를 관리한다.

---

## 2. Deployment가 필요한 이유

Pod를 직접 생성하면 다음과 같은 문제가 있다.

- Pod가 삭제되면 복구되지 않을 수 있음
    
- 애플리케이션 업데이트 관리가 어려움
    
- 여러 Pod 관리가 어려움
    

Deployment는 이러한 문제를 해결한다.

예

```text
웹 서버 3개 실행
```

Deployment 설정

```text
replicas = 3
```

Pod 하나가 삭제되면

```text
3 → 2
```

Deployment가 ReplicaSet을 통해 새로운 Pod를 생성한다.

---

## 3. Deployment 예시

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
```

설명

|항목|의미|
|---|---|
|replicas|실행할 Pod 개수|
|selector|관리할 Pod 선택|
|template|생성할 Pod 정의|

결과

```text
nginx Pod 3개 실행
```

---

## 4. 롤링 업데이트

Deployment는 **애플리케이션을 무중단으로 업데이트할 수 있다.**

예

```text
nginx:1.20 → nginx:1.21
```

Deployment 동작

```text
기존 Pod 유지
↓
새 Pod 생성
↓
기존 Pod 점진적 종료
```

즉 서비스 중단 없이 업데이트된다.

---

## 5. 롤백 기능

업데이트에 문제가 생기면 이전 버전으로 돌아갈 수 있다.

예

```bash
kubectl rollout undo deployment nginx-deployment
```

결과

```text
이전 버전 Pod로 복구
```

---

## 6. Deployment 특징

|특징|설명|
|---|---|
|Pod 관리|ReplicaSet을 통해 관리|
|자동 복구|Pod 삭제 시 새 Pod 생성|
|롤링 업데이트|무중단 배포|
|롤백|이전 버전 복구 가능|

---

## 핵심 정리

Deployment는 Kubernetes에서 애플리케이션 Pod를 배포하고 관리하는 컨트롤러 객체이다. ReplicaSet을 통해 Pod의 개수를 유지하며, 롤링 업데이트와 롤백 기능을 제공하여 애플리케이션을 안정적으로 배포하고 관리할 수 있다.
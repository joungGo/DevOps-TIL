## StatefulSet

StatefulSet은 **상태(State)를 가지는 애플리케이션을 관리하기 위한 Kubernetes 컨트롤러 객체**이다.

Pod에 **고유한 이름, 고정된 네트워크, 지속적인 저장소**를 제공한다.

주로 다음과 같은 서비스에 사용된다.

- 데이터베이스
    
- 분산 시스템
    
- 메시지 큐
    

예  
MySQL, MongoDB, Kafka, Redis 등

---

## 1. StatefulSet이 필요한 이유

Deployment는 **모든 Pod가 동일하고 서로 구분되지 않는다.**

예

```text
web-pod-1
web-pod-2
web-pod-3
```

Pod가 삭제되면 **새로운 Pod가 임의 이름으로 생성된다.**

하지만 데이터베이스 같은 시스템은 다음이 필요하다.

- Pod마다 **고유한 이름**
    
- **고정된 네트워크 주소**
    
- **데이터 유지**
    

이를 해결하는 것이 StatefulSet이다.

---

## 2. StatefulSet 특징

### 1) 고유한 Pod 이름

Pod 이름이 **순서 번호로 고정된다.**

예

```text
mysql-0
mysql-1
mysql-2
```

Pod가 재시작되어도 같은 이름을 유지한다.

---

### 2) 고정된 네트워크

각 Pod는 **고정된 DNS 이름**을 가진다.

예

```text
mysql-0.mysql
mysql-1.mysql
mysql-2.mysql
```

따라서 다른 서비스가 **특정 Pod에 항상 접근 가능**하다.

---

### 3) 영구 스토리지

Pod마다 **개별 Persistent Volume**이 연결된다.

예

```text
mysql-0 → volume-0
mysql-1 → volume-1
mysql-2 → volume-2
```

Pod가 삭제되어도 **데이터는 유지된다.**

---

### 4) 순차적 생성 및 종료

StatefulSet은 Pod를 **순서대로 생성한다.**

예

```text
mysql-0 생성
↓
mysql-1 생성
↓
mysql-2 생성
```

삭제도 **역순으로 진행된다.**

```text
mysql-2 삭제
↓
mysql-1 삭제
↓
mysql-0 삭제
```

---

## 3. StatefulSet 예시

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql
```

결과

```text
mysql-0
mysql-1
mysql-2
```

각 Pod는 **고유한 이름과 저장소를 가진다.**

---

## 4. Deployment vs StatefulSet

|항목|Deployment|StatefulSet|
|---|---|---|
|Pod 식별|없음|고유 이름|
|Pod 순서|없음|순차 생성|
|스토리지|공유 가능|Pod별 고정|
|사용 용도|stateless 앱|stateful 앱|

---

## 핵심 정리

StatefulSet은 상태를 가지는 애플리케이션을 관리하기 위한 Kubernetes 컨트롤러이다. Pod마다 고유한 이름과 고정된 네트워크 주소, 영구 저장소를 제공하며, Pod를 순서대로 생성하고 종료한다. 주로 데이터베이스나 분산 시스템과 같이 상태가 중요한 서비스에 사용된다.
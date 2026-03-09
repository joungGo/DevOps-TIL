## ReplicaSet

**ReplicaSet은 지정한 개수의 Pod가 항상 유지되도록 관리하는 Kubernetes 리소스이다.**

---

## 역할

ReplicaSet의 핵심 역할은 하나이다.

```text
원하는 개수의 Pod 유지
```

예

```yaml
replicas: 3
```

의미

```text
Pod가 항상 3개 유지되도록 관리
```

Pod 하나가 죽으면

```text
ReplicaSet → 새로운 Pod 생성
```

---

## 동작 구조

```text
Deployment
   ↓
ReplicaSet
   ↓
Pod
```

역할

|구성|역할|
|---|---|
|Deployment|배포 관리|
|ReplicaSet|Pod 개수 유지|
|Pod|컨테이너 실행|

---

## Pod 선택 방식

ReplicaSet은 **label selector**로 Pod를 찾는다.

예

```yaml
selector:
  matchLabels:
    app: my-app
```

의미

```text
app=my-app label을 가진 Pod 관리
```

---

## 왜 직접 사용하지 않을까

ReplicaSet은 보통 **직접 사용하지 않는다.**

이유

Deployment가 **ReplicaSet을 자동으로 생성하고 관리**하기 때문이다.

즉

```text
Deployment → ReplicaSet 생성 → Pod 관리
```

---

## 면접용 정리

**ReplicaSet은 지정한 개수의 Pod가 항상 실행되도록 유지하는 Kubernetes 리소스이며, 일반적으로 Deployment에 의해 자동으로 생성되고 관리된다.**
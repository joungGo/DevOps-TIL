## DaemonSet

DaemonSet은 **모든 노드(Node) 또는 특정 노드에 하나씩 Pod가 실행되도록 보장하는 Kubernetes 컨트롤러 객체**이다.

즉, **각 노드마다 동일한 Pod를 하나씩 자동으로 배치한다.**

---

## 1. DaemonSet이 필요한 이유

일반적인 Pod나 Deployment는 **Pod가 특정 노드에만 실행될 수 있다.**

예

```text
Node1 → Pod 실행
Node2 → 없음
Node3 → 없음
```

하지만 다음과 같은 프로그램은 **모든 노드에서 실행되어야 한다.**

예

- 로그 수집
    
- 모니터링
    
- 네트워크 관리
    

이때 사용하는 것이 **DaemonSet**이다.

---

## 2. DaemonSet 동작 방식

DaemonSet은 **각 노드에 Pod 하나씩 실행한다.**

예

```text
Cluster
 ├ Node1 → Pod1
 ├ Node2 → Pod2
 ├ Node3 → Pod3
```

특징

- 노드가 추가되면 Pod도 자동 생성
    
- 노드가 제거되면 Pod도 함께 제거
    

---

## 3. DaemonSet 사용 예

DaemonSet은 보통 **클러스터 관리용 프로그램**에 사용된다.

대표적인 예

|용도|예|
|---|---|
|로그 수집|Fluentd|
|모니터링|Prometheus Node Exporter|
|네트워크|CNI 플러그인|
|보안|보안 에이전트|

예

```text
모든 노드에서 로그 수집
```

```text
Node1 → Fluentd
Node2 → Fluentd
Node3 → Fluentd
```

---

## 4. DaemonSet 예시

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluentd
```

결과

```text
Node1 → fluentd Pod
Node2 → fluentd Pod
Node3 → fluentd Pod
```

각 노드마다 Pod가 하나씩 실행된다.

---

## 5. Deployment vs DaemonSet

|항목|Deployment|DaemonSet|
|---|---|---|
|Pod 개수|replicas로 설정|노드 수만큼|
|Pod 위치|아무 노드|모든 노드|
|사용 목적|일반 애플리케이션|노드 관리 프로그램|

---

## 핵심 정리

DaemonSet은 Kubernetes에서 모든 노드 또는 특정 노드마다 하나의 Pod가 실행되도록 보장하는 컨트롤러이다. 노드가 추가되면 자동으로 Pod가 생성되며, 주로 로그 수집, 모니터링, 네트워크 관리와 같은 클러스터 관리 프로그램에 사용된다.
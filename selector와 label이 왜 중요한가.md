## selector와 label이 중요한 이유

**Kubernetes는 리소스 간 연결을 label과 selector로 한다.**

즉

```text
label = 리소스에 붙는 식별 정보
selector = label을 기준으로 리소스를 찾는 조건
```

---

## 왜 필요한가

Kubernetes는 **리소스 이름으로 연결하지 않는다.**

대신

```text
label → selector 매칭
```

으로 연결한다.

---

## 예시

### Pod

```yaml
metadata:
  labels:
    app: my-app
```

### Service

```yaml
spec:
  selector:
    app: my-app
```

의미

```text
Service는 app=my-app label을 가진 Pod로 트래픽 전달
```

---

## 실제 연결 구조

```text
Service
  selector: app=my-app
        │
        ▼
Pod
  label: app=my-app
```

---

## selector / label이 중요한 이유

### 1. Pod를 자동으로 찾기 위해

Pod IP는 계속 바뀐다.

그래서

```text
label로 Pod 그룹을 식별
```

한다.

---

### 2. 로드밸런싱을 위해

Service는 selector로 Pod들을 찾고

```text
Pod1
Pod2
Pod3
```

여기로 트래픽을 분산한다.

---

### 3. Deployment가 Pod 관리

Deployment도 selector를 사용한다.

```yaml
selector:
  matchLabels:
    app: my-app
```

이 label을 가진 Pod들을 **관리 대상으로 삼는다.**

---

## selector가 틀리면 생기는 문제

예

Pod

```yaml
labels:
  app: nginx
```

Service

```yaml
selector:
  app: my-app
```

결과

```text
Service → Pod 못 찾음
```

즉

```text
트래픽 전달 실패
```

---

## 면접용 핵심 정리

**Kubernetes는 리소스 간 연결을 label과 selector로 한다.  
label은 리소스 식별 정보이고 selector는 해당 label을 가진 리소스를 찾기 위한 조건이다.  
Service, Deployment 등이 Pod를 찾고 관리할 때 사용된다.**
## 레이블 (Label)

레이블(Label)은 Kubernetes 객체에 붙는 **key-value 형태의 메타데이터**이다.  
Pod, Service, Deployment 같은 리소스를 **구분하고 그룹화하기 위해 사용된다.**

형식

```
key: value
```

예

```yaml
labels:
  app: nginx
  tier: frontend
  env: production
```

---

## 1. 레이블을 사용하는 이유

Kubernetes 환경에서는 많은 Pod가 실행되기 때문에 **Pod를 구분하고 관리할 방법이 필요하다.**

예를 들어 다음과 같은 애플리케이션 구조가 있을 수 있다.

```
frontend Pod
backend Pod
database Pod
```

이때 각 Pod에 레이블을 붙여서 구분한다.

예

```yaml
labels:
  app: web
  tier: frontend
```

---

## 2. 레이블 예시

Pod 생성 예

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    tier: frontend
spec:
  containers:
  - name: nginx
    image: nginx
```

이 Pod에는 다음 레이블이 붙는다.

```
app=nginx
tier=frontend
```

---

## 3. 레이블을 이용한 조회

레이블을 기준으로 특정 Pod를 조회할 수 있다.

예

```
kubectl get pods -l app=nginx
```

의미

```
app=nginx 레이블을 가진 Pod 조회
```

---

## 4. 레이블과 Selector

레이블은 **Selector와 함께 사용된다.**

Selector는 **특정 레이블을 가진 리소스를 선택하는 조건**이다.

구조

```
Service / Deployment
        ↓
     Selector
        ↓
       Label
        ↓
        Pod
```

예

Service 설정

```yaml
selector:
  app: nginx
```

Pod 설정

```yaml
labels:
  app: nginx
```

이 경우 Service는 **app=nginx 레이블을 가진 Pod와 연결된다.**

---

## 5. 동작 예시

Pod가 3개 있다고 가정한다.

```
Pod A
labels: app=web

Pod B
labels: app=web

Pod C
labels: app=db
```

Service 설정

```yaml
selector:
  app: web
```

결과

```
Service
 ├ Pod A
 └ Pod B
```

Pod C는 Service와 연결되지 않는다.

---

## 6. 레이블 특징

|특징|설명|
|---|---|
|key-value 구조|key=value 형태|
|리소스 식별|Pod 구분 가능|
|그룹화|여러 Pod를 하나의 그룹으로 관리|
|Selector와 사용|Service, Deployment 등이 Pod를 선택|

---

## 핵심 정리

레이블(Label)은 Kubernetes 리소스에 붙는 key-value 형태의 메타데이터이며, Pod와 같은 객체를 식별하고 그룹화하는 데 사용된다. Service, Deployment 같은 객체는 Selector를 사용해 특정 레이블을 가진 Pod를 선택하고 관리한다.
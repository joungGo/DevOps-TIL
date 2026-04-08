## ServiceAccount란? 🔐

👉 **Pod가 Kubernetes API에 접근할 때 사용하는 "계정"**

즉,

- 사용자 계정(User) → 사람이 쓰는 것
    
- **ServiceAccount** → **Pod가 쓰는 것**
    

---

## 왜 필요해?

Pod가 이런 작업할 때 필요:

- 다른 Pod 조회
    
- Secret 읽기
    
- ConfigMap 읽기
    
- Kubernetes API 호출
    

권한 없으면 ❌ 접근 못 함

---

## 기본 구조

```id="w5q3kk"
Pod → ServiceAccount → Role → RoleBinding → 권한
```

---

## 기본 ServiceAccount

Pod는 아무 설정 없으면 자동으로:

```id="f49a3j"
default
```

ServiceAccount 사용함

확인:

```bash
kubectl get sa
```

> [!summary] ServiceAccount: default vs 별도 사용 핵심
> 
> - **default ServiceAccount**: 모든 Pod가 동일 권한 공유 → 권한 분리 불가, 보안 취약
>     
> - **별도 ServiceAccount**: Pod별 권한 분리 → 최소 권한 원칙 적용, 보안 강화
>     
> - **default 사용**: 단순 앱, Kubernetes API 접근 없음
>     
> - **별도 사용**: Secret/ConfigMap 접근, 리소스 조회, 운영 환경
>     
> - **핵심**: _운영 환경에서는 Pod별 ServiceAccount를 분리하는 것이 권장됨_
>

---

## ServiceAccount 생성

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-sa
```

---

## Pod에 연결

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test
spec:
  serviceAccountName: api-sa
  containers:
  - name: app
    image: nginx
```

```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      serviceAccountName: api-sa
      containers:
        - name: api
          image: my-api:latest
```

---

## 언제 쓰나? (실무)

- Pod가 Secret 읽어야 할 때
    
- CI/CD Pod가 배포할 때
    
- operator / controller 만들 때
    
- external API 대신 k8s API 호출할 때
    

---

## 면접 한 줄 🎤

👉 **ServiceAccount는 Pod가 Kubernetes API에 접근할 때 사용하는 계정이며 RBAC과 함께 권한을 제어한다.**

---

## 핵심 요약 ⚡

- Pod용 계정
    
- RBAC 권한 연결
    
- 기본값 = `default`
    
- 필요 시 별도 생성해서 사용
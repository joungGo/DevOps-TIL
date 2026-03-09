> [!note] 면접 답변
> Service는 Pod를 직접 찾는 것이 아니라 **Label Selector를 통해 대상 Pod를 선택하고, Kubernetes의 Endpoint Controller가 해당 Pod들의 IP를 Endpoint 리소스로 관리합니다. 이후 kube-proxy가 Endpoint 정보를 기반으로 Service IP에서 Pod IP로 트래픽을 전달하는 규칙을 생성합니다.**

---
### Service는 Pod를 어떻게 찾을까

**“spec의 selector + label”을 통해 Pod를 찾는다.**

핵심 구조

```text
Service selector  →  Pod label
```

예시

Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: payment-pod
  labels:
    app: payment
```

Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  selector:
    app: payment
  ports:
    - port: 80
      targetPort: 8080
```

동작 과정

```text
1. Service 생성
2. selector(app=payment) 확인
3. 해당 label을 가진 Pod 검색
4. Endpoint 목록 생성
5. Service가 Endpoint Pod로 트래픽 전달
```

구조

```text
Service
 selector: app=payment
      │
      │
      ▼
Pod1 (label: app=payment)
Pod2 (label: app=payment)
Pod3 (label: app=payment)
```

이 Pod 목록은 **Endpoint 또는 EndpointSlice**라는 리소스로 관리된다.

즉 실제 흐름은 다음과 같다.

```text
Service
   ↓
Endpoint / EndpointSlice
   ↓
Pod
```

요약

|구성요소|역할|
|---|---|
|label|Pod의 식별 태그|
|selector|Service가 Pod를 찾는 조건|
|Endpoint|실제 Pod IP 목록|

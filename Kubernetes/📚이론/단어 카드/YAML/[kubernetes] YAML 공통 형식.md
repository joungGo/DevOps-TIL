쿠버네티스 YAML은 거의 **모든 리소스가 동일한 4단 구조**를 가진다.

면접에서도 보통 이 구조로 설명한다.

```
apiVersion
kind
metadata
spec
```

---

# 1. Kubernetes YAML 공통 구조

```yaml
apiVersion: apps/v1      # 사용할 Kubernetes API 버전
kind: Deployment         # 생성할 리소스 종류

metadata:                # 리소스 기본 정보
  name: my-app           # 리소스 이름
  labels:                # 리소스 식별용 라벨
    app: my-app

spec:                    # 리소스의 실제 설정
  replicas: 3
  selector:
    matchLabels:
      app: my-app

  template:              # Pod 템플릿
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: nginx
          ports:
            - containerPort: 80
```

---

# 2. 각 영역의 역할

## 1) apiVersion

**어떤 Kubernetes API 버전을 사용할지 정의**

예

```
apps/v1
v1
networking.k8s.io/v1
```

예시

|리소스|apiVersion|
|---|---|
|Pod|v1|
|Service|v1|
|Deployment|apps/v1|
|Ingress|networking.k8s.io/v1|

---

## 2) kind

**생성할 리소스 타입**

예

```markdown
Pod         # 하나 이상의 컨테이너를 실행하는 Kubernetes의 최소 배포 단위
Deployment  # Pod를 생성하고 개수 유지, 업데이트, 롤백 등을 관리하는 리소스
Service     # 여러 Pod에 안정적으로 접근하기 위한 고정 네트워크 엔드포인트 제공
ConfigMap   # 애플리케이션 설정 정보를 Key-Value 형태로 저장하는 리소스
Secret      # 비밀번호, 토큰 같은 민감 정보를 저장하는 리소스
Ingress     # 외부 HTTP/HTTPS 트래픽을 내부 Service로 라우팅하는 리소스
```

즉 Kubernetes에게

```
"어떤 종류의 리소스를 만들지"
```

알려준다.

---

## 3) metadata

**리소스의 기본 정보**

예

```yaml
metadata:
  name: my-app
  namespace: default
  labels:
    app: my-app
```

주요 필드

|필드|역할|
|---|---|
|name|리소스 이름|
|namespace|리소스가 속한 네임스페이스|
|labels|리소스 식별용 태그|
>Namespace는 Kubernetes 클러스터 내에서 리소스를 논리적으로 구분하고 관리하기 위한 격리된 공간이다.

labels는 **Service가 Pod를 찾을 때 사용된다.**

> [!info] Kubernetes 기본 Namespace
>
> | namespace       | 설명 |
> |-----------------|------------------------------|
> | default         | 기본 namespace (명시하지 않으면 여기에 생성) |
> | kube-system     | Kubernetes 시스템 컴포넌트 |
> | kube-public     | 모든 사용자가 읽을 수 있는 namespace |
> | kube-node-lease | 노드 상태 정보 관리 |
> | (사용자 namespace) | (사용자가 만든 namespace) |

---

## 4) spec

**리소스의 실제 동작 정의**

예

```yaml
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
```

예시

Deployment spec

```
replicas
update 전략
Pod template
```

Service spec

```
type
selector
port
```

즉

```
이 리소스를 어떻게 동작시킬지 정의
```

---

# 3. 핵심 구조 요약

면접에서는 보통 이렇게 설명한다.

```
Kubernetes YAML은

apiVersion
kind
metadata
spec

4가지 구조로 구성됩니다.

apiVersion은 사용할 API 버전,
kind는 생성할 리소스 타입,
metadata는 리소스 정보,
spec은 리소스의 실제 동작을 정의합니다.
```

---

# 4. 한 가지 더 중요한 구조

Deployment 같은 리소스에서는 **Pod Template 구조가 추가된다.**

```
Deployment
   └── Pod Template
          └── Container
```

YAML 구조

```
Deployment
 └ spec
    └ template
       └ spec
          └ containers
```

즉

```
Deployment
 → Pod 생성
 → Pod 안에 Container 실행
```

---

# 5. 실무에서 가장 중요한 흐름

쿠버네티스 리소스 관계는 보통 이렇게 연결된다.

```
Deployment
     ↓
    Pod
     ↓
  Service
     ↓
  Ingress
```

의미

```
Deployment → Pod 생성
Service → Pod 접근
Ingress → 외부 접근
```

---

원하면 다음도 정리해줄게.

1. **Kubernetes YAML 10개 필수 필드 (실무에서 90% 사용)**
    
2. **Deployment → Pod → Service → Ingress 전체 흐름**
    
3. **쿠버네티스 네트워크 흐름 (외부 → Ingress → Service → Pod)**
    

이 3개는 **쿠버네티스 면접에서 거의 반드시 나오는 내용이다.**
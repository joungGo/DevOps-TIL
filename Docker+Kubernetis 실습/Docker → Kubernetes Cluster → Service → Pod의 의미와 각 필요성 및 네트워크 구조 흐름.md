# Docker → Kubernetes 총 정리본

## 1️⃣ 왜 Docker만으로는 부족한가?

### Docker의 역할

Docker는 **컨테이너를 실행하는 엔진**이다.

```bash
docker run -d -p 80:80 nginx
```

위 명령은 다음을 의미한다.

- nginx 이미지를 기반으로 컨테이너 생성
    
- Host의 80번 포트를 컨테이너 80번 포트에 직접 연결
    
- nginx 프로세스 실행
    

### Docker 단독 환경의 한계

- 서버가 1대라는 전제
    
- 컨테이너가 죽으면 서비스도 종료
    
- 자동 복구 불가
    
- 여러 서버에서의 확장 관리 불가
    

👉 **"컨테이너 실행"은 가능하지만, "서비스 운영"은 어렵다**

---

## 2️⃣ Kubernetes는 무엇을 해결하려는가?

### Kubernetes 한 문장 정의

> Kubernetes는 여러 대의 서버를 하나의 시스템처럼 묶어  
> 컨테이너 기반 서비스를 자동으로 배포·복구·확장·노출하는 플랫폼이다.

### Kubernetes 클러스터 구조

```text
[Kubernetes Cluster]
 ├─ Control Plane (Master)
 │   ├─ API Server
 │   ├─ Scheduler
 │   ├─ Controller Manager
 │   └─ etcd
 │
 └─ Worker Nodes
     ├─ kubelet
     ├─ container runtime (Docker)
     └─ Pods
```

- **Master**: 전체 클러스터의 뇌
    
- **Worker**: 실제 컨테이너가 실행되는 서버
    

---

## 3️⃣ Pod란 무엇인가?

### Pod 정의

> Pod는 Kubernetes에서 **컨테이너를 실행하는 최소 단위**이다.

Kubernetes는 컨테이너를 직접 다루지 않고 **항상 Pod 안에서 실행**한다.

### Pod의 핵심 특징

- Pod마다 고유한 IP를 가짐
    
- Pod 내부의 모든 컨테이너는 **같은 IP / 네트워크 네임스페이스 공유**
    
- Pod는 언제든지 죽고 새로 생성될 수 있음 (IP 변경)
    

### Pod 구조

```text
[Pod]
 ├─ Container (nginx)
 ├─ (선택) Sidecar Container
 └─ Pod IP (공유)
```

### nginx Pod 예시 (Pod 단독)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```

⚠️ 이 Pod는 **자동 복구되지 않는다**.

---

## 4️⃣ Deployment란 무엇인가? (자동 복구의 핵심)

### Deployment 정의

> Deployment는 Pod를 선언적으로 관리하는 컨트롤러로,  
> Pod의 개수 유지, 자동 재생성, 롤링 업데이트를 담당한다.

### nginx Deployment 예시

```yaml
apiVersion: apps/v1          # 사용할 쿠버네티스 API 버전 (Deployment는 apps/v1)
kind: Deployment             # 리소스 종류: Deployment
metadata:
  name: nginx-deploy         # Deployment 이름
spec:
  replicas: 3                # 생성할 파드(Pod) 개수
  selector:
    matchLabels:
      app: nginx             # 이 Deployment가 관리할 파드를 고르는 조건
  template:                  # 실제로 생성될 파드의 설계도
    metadata:
      labels:
        app: nginx           # 파드에 붙일 라벨 (selector와 반드시 일치해야 함)
    spec:
      containers:            # 파드 안에서 실행될 컨테이너 목록
      - name: nginx          # 컨테이너 이름
        image: nginx         # 사용할 도커 이미지
        ports:
        - containerPort: 80  # 컨테이너 내부에서 열 포트 -> 문서 상(실제 구동 Port가 아닌 동료에게 보여주기 위한 문서상의 기재 요소임)
```

이 의미는 다음과 같다.

- nginx Pod를 항상 3개 유지
    
- Pod가 죽으면 자동으로 새 Pod 생성
    
- Pod IP는 바뀌어도 상관 없음
    

---

## 5️⃣ Service란 무엇인가?

### Service 한 문장 정의

> Service는 **죽고 살아나는 Pod들을 대신해  
> 항상 고정된 접근 지점을 제공하는 네트워크 객체**이다.

### Service의 핵심 설계 철학

- Service는 Pod를 직접 기억하지 않는다
    
- **selector(라벨 조건)**만 기억한다
    
- 실제 Pod IP 목록은 **Endpoints 객체가 자동 관리**
    

---

## 6️⃣ Service와 Pod의 관계

### 라벨 기반 연결 구조

```text
[Service]
  selector: app=nginx
        ↓
[Pod A] app=nginx
[Pod B] app=nginx
[Pod C] app=nginx
```

Pod가 늘어나거나 줄어도 Service는 변경되지 않는다.

---

## 7️⃣ Service 유형과 nginx 예시

### 1️⃣ ClusterIP (기본)

- 클러스터 내부 전용
    

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

**개념**

- 쿠버네티스 **클러스터 내부에서만 접근 가능한 Service**
    
- 기본 Service 타입
    

**접근 범위**

- 클러스터 내부 파드/서비스 → 접근 가능
    
- 외부(브라우저, PC) → ❌ 접근 불가
    

**특징**

- 고정된 **가상 IP(Cluster IP)** 를 가짐
    
- 파드가 죽고 새로 생겨도 Service IP는 유지됨
    
- 내부 마이크로서비스 통신에 사용
    

**사용 예**

- Backend ↔ Database
    
- Frontend ↔ Backend (둘 다 클러스터 내부일 때)
    

**흐름**

`Pod → Service(ClusterIP) → Pod`

**언제 쓰나**

- 외부에 노출할 필요 없는 내부 전용 서비스일 때

---

### 2️⃣ NodePort (외부 접근 가능)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

접속 방식:

```text
http://<ANY_NODE_IP>:30080
```

**개념**

- **모든 노드의 특정 포트**를 열어서 외부 접근을 허용하는 Service
    

**접근 범위**

- 클러스터 내부 → 접근 가능
    
- 클러스터 외부 → ⭕ 접근 가능
    

**특징**

- 노드 포트 범위: `30000 ~ 32767`
    
- 모든 노드 IP + NodePort 로 접속 가능
    
- 내부적으로는 **ClusterIP를 포함**하고 있음
    

**사용 예**

- 테스트 환경에서 빠르게 외부 노출
    
- LoadBalancer 없이 외부 접근이 필요할 때
    

**흐름**

`외부 → NodeIP:NodePort → Service → Pod`

**언제 쓰나**

- 로컬 / 실습 / 간단한 외부 접근이 필요할 때

---

[[운영환경에서는 LoadBalancer Service를 사용하는 이유. NodePort는 왜 안될까]]

---

## 8️⃣ 전체 흐름 한 번에 보기

### 실행 관점 흐름

```text
kubectl apply
   ↓
Deployment
   ↓
ReplicaSet
   ↓
Pod
   ↓
Container
   ↓
nginx process
```

### 네트워크 접근 흐름

```text
Browser
   ↓
NodePort Service
   ↓
Pod IP
   ↓
Container
   ↓
nginx
```

---

## 9️⃣ Docker vs Kubernetes 핵심 비교

|항목|Docker 단독|Kubernetes|
|---|---|---|
|실행 단위|Container|Pod|
|복구|없음|자동|
|확장|수동|자동|
|네트워크|직접 바인딩|Service 기반|
|운영 적합성|낮음|높음|

---

## 🔟 이 문서를 통해 반드시 이해해야 할 핵심 문장

- nginx는 **Pod가 아니라 컨테이너에서 실행된다**
    
- Pod는 언제든지 죽을 수 있다
    
- Deployment가 Pod를 다시 만든다
    
- Service는 Pod를 직접 관리하지 않고 selector로 연결한다
    
- 사용자는 Pod가 아닌 **Service에 접속한다**
    

---

## ✅ 최종 한 줄 요약

> Kubernetes는 컨테이너를 실행하는 기술이 아니라,  
> **컨테이너 기반 서비스를 끊기지 않게 운영하기 위한 시스템이다.**
### 0. 쿠버네티스 설계 구조

- 아키텍처
```md
외부 (브라우저 / curl)
           │
           │ HTTP 요청 (url-shortener.local)
           ▼
┌──────────────────────────────────────────────────────────────┐
│                      minikube 클러스터                         │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │                  Ingress (NGINX)                       │  │
│  │              k8s/ingress.yaml                          │  │
│  └───────────────────────┬────────────────────────────────┘  │
│                          │ :80                               │
│                          ▼                                   │
│  ┌────────────────────────────────────────────────────────┐  │
│  │          Service: url-shortener-svc (ClusterIP)        │  │
│  │               port 80 → targetPort 8000                │  │
│  │                    k8s/service.yaml                    │  │
│  └──────────────┬───────────────────┬──────────────────────┘  │
│                 │                   │  로드밸런싱              │
│                 ▼                   ▼                         │
│  ┌──────────────────┐   ┌──────────────────┐                 │
│  │   API Pod #1     │   │   API Pod #2     │                 │
│  │url-shortener-api │   │url-shortener-api │  Deployment     │
│  │      :v1         │   │      :v1         │  replicas: 2    │
│  │     :8000        │   │     :8000        │  k8s/           │
│  │  APP_ENV=prod    │   │  APP_ENV=prod    │  deployment.yaml│
│  │  DEBUG=false     │   │  DEBUG=false     │                 │
│  │  readinessProbe  │   │  readinessProbe  │                 │
│  │  livenessProbe   │   │  livenessProbe   │                 │
│  └────────┬─────────┘   └────────┬─────────┘                 │
│           │                     │                            │
│           └──────────┬──────────┘                            │
│                      │ DATABASE_URL (app-secret 참조)         │
│                      ▼ :5432                                 │
│  ┌────────────────────────────────────────────────────────┐  │
│  │          Service: postgres-svc (ClusterIP)             │  │
│  │               port 5432 → targetPort 5432              │  │
│  │                  k8s/db/postgres-svc.yaml              │  │
│  └───────────────────────┬────────────────────────────────┘  │
│                          │ :5432                             │
│                          ▼                                   │
│  ┌────────────────────────────────────────────────────────┐  │
│  │           StatefulSet: postgres-0                      │  │
│  │           image: postgres:15  replicas: 1              │  │
│  │           POSTGRES_USER=postgres                       │  │
│  │           POSTGRES_DB=tempdb                           │  │
│  │           POSTGRES_PASSWORD (app-secret 참조)          │  │
│  │              k8s/db/postgres-stateful.yaml             │  │
│  └───────────────────────┬────────────────────────────────┘  │
│                          │ 마운트                             │
│                          ▼ /var/lib/postgresql/data          │
│  ┌────────────────────────────────────────────────────────┐  │
│  │           PVC: postgres-pvc                            │  │
│  │           1Gi / ReadWriteOnce / standard               │  │
│  │              k8s/db/postgres-pvc.yaml                  │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │              Secret: app-secret                        │  │
│  │   POSTGRES_PASSWORD ──────────────► StatefulSet        │  │
│  │   DATABASE_URL      ──────────────► Deployment         │  │
│  │              k8s/db/secret.yaml                        │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

- 폴더 구조
```md
k8s/
├── secret.yaml              # POSTGRES_PASSWORD, DATABASE_URL 보관
├── postgres-pvc.yaml        # PostgreSQL 데이터 저장 공간 (1Gi)
├── postgres-statefulset.yaml # PostgreSQL Pod (데이터 영속성 보장)
├── postgres-svc.yaml        # API Pod → PostgreSQL 연결 통로
├── deployment.yaml          # API Pod x2 (무중단 배포, 자동 재시작)
├── service.yaml             # Ingress → API Pod 연결 통로
└── ingress.yaml             # 외부 트래픽 진입점, 도메인 라우팅
```

---
### 1. MiniKube 사용

#### 문제점
Minikube는 로컬에서 격리된 Kubernetes 노드 환경(VM 또는 컨테이너)을 만들어 실행하는 도구이다.
따라서, **로컬 PC에서 빌드한 도커 이미지는 Minikube에서 볼 수 없다.**

```md
내 PC
 ├─ Local Docker
 │   └─ my-app:latest
 │
 └─ Minikube
     └─ Kubernetes Node
         └─ Docker (또는 containerd)
             └─ (이미지 없음)
```

#### 해결 방법
위 문제를 해결하기 위해 로컬의 이미지를 minikube로 넣어줘야 하는데 나는 이 방법 대신 **로컬 CLI의 Docker 명령이 Minikube 내부 Docker를 가리키게 전환한 뒤 빌드**하도록 하였다.

>MinuiKube는 자신만의 Docker Engine을  가지기 때문에 로컬의 Docker 명령어를 MiniKube Docker와 연결해야 한다.

```Linux
eval $(minikube docker-env) # Mac/Linux

docker info | grep -E "Name|Server Version" # 확인 명령어
```

```powershell
# 1. 터미널을 minikube Docker 환경으로 전환
& minikube -p minikube docker-env --shell powershell | Invoke-Expression

# 2. 전환됐는지 확인 (DOCKER_HOST가 minikube 주소로 바뀌어 있어야 함)
docker info | Select-String "Name"

# 3. url-shortener 프로젝트 폴더에서 이미지 빌드
docker build -t url-shortener:v1 .

# 4. 이미지 확인
docker images | Select-String "url-shortener"

# 5. Deployment 재시작
kubectl rollout restart deployment url-shortener

# 6. 상태 확인
kubectl get pods
```
- 단점: 터미널 세션마다 재설정 필요
- `Minukube 클러스터가 시작(start) 된 후에 실행해야 한다.`

> [!info] 다른 방법 
> - 방법A - 로컬에서 빌드 후 minukube 안으로 전송. 이미지가 클수록 느림 
> `minikube image load <이미지명>`
> 
> - 방법B - 외부 레지스트리(Docker Hub, ECR) 사용 ✔️(실제 프로덕션 방식에서 주로 사용, 간단한 테스트용으로는 너무 과하다고 판단)

---
### 2. Minikube 클러스터 띄우기

#### 2-1. 클러스터 시작

```powershell
minikube start --driver=docker
```
![[Pasted image 20260310174908.png]]
#### 2-2. ingress 애드온 활성화

- ingress 애드온 활성화

```powershell
minikube addons enable ingress
```
![[Pasted image 20260310174923.png]]

- Minikube 에서 사용 가능한 애드온 목록을 출력

```linux
minikube addons list | grep ingress
```

```powershell
minikube addons list | Select-String ingress
```
![[Pasted image 20260310174848.png]]

---
### 3. Secret으로 보안 정보 별도 관리

- `./gitignore`에 `sercret.yaml` 을 추가
- `secret.yaml`의 `password`와 `db_url`은 Base64로 인코딩 후 기입

---
### 4. PostgreSql StatefulSet + PVC 배포

여러개의 Pod 존재하는 구조에서 이 Pod들은 DB가 올라가 있는 하나의 Pod를 공유해야 한다.
따라서..
- Pod 마다 고정된 이름과 고정된 스토리지를 부여할 수 있는 방법이 필요

1. pod가 재시작돼도 같은 디스크를 다시 붙임
2. 순서 있는 배포/삭제로 마스터/레플리카 구성도 가능

![[Pasted image 20260312121151.png]]

---
### 5.  deployment.yaml로 메인 API 서버 올리기

![[Pasted image 20260312135705.png]]

---
### 6. Service(ClusterIP)로 내부 연결

외부 노출은 Ingress가 담당하고 Service는 내부 라우팅만 맡음.
ClusterIP + Ingress 조합으로 진행

- Ingress 적용 전 임시 port-forward 사용하여 테스트 진행 결과
![[Pasted image 20260312170938.png]]

---
### 7. Ingress로 외부 접속 라우팅 진행

> [!why] 로드밸런서를 왜 사용하지 않았을까?
> 로드밸런서를 외부 트래픽 요청에 대해 최초 진입점 역할을 한다. 이 진입점 역할을 하는 서비스는 `LoadBalancer`, `NodePort`, `Ingress` 가 있다.
> 현재 구조의 경우 **라우팅 대상이 되는 Service가 1개**라는 점, **Minikube는 Ingress 애드온을 enable할 경우 NordPort로 외부 트래픽을 받아 Ingress 규칙대로 라우팅 진행**을 한다는 점에서 LoadBalance의 필요성이 낮다고 판단했다.
> 
> 그리고, 어차피 AWS EKS 구조에서는 우리가 Ingress만 설정하면 앞단에 ALB가 자동으로 붙기 때문에  수동으로 만들 필요는 없지 않을까 생각한다.

- DB까지 데이터 잘 들어가는 것 확인 완료
![[Pasted image 20260312204810.png]]

- health 체크도 완료
![[Pasted image 20260312204858.png]]

---
### 문제 및 해결 모음

#### 문제 1: 이미지 풀 실패 (`ErrImageNeverPull`)
- **문제**  
  - Deployment 는 `image: url-shortener-api:v1` 를 사용 중이었는데,  
    로컬에서 빌드한 이미지는 `url-shortener:v1` 이라서 이름이 달랐음.  
  - `imagePullPolicy: Never` 라 외부 레지스트리도 못 쓰고, 노드에도 `url-shortener-api:v1` 태그가 없어서 `ErrImageNeverPull` 발생.

- **해결 방법**  
  - 기존 이미지에 Deployment 와 같은 태그를 추가:
    ```bash
    docker tag url-shortener:v1 url-shortener-api:v1
    ```
  - 이후 `kubectl rollout restart deployment url-shortener` 로 재시작하여 정상적으로 이미지를 사용.

---

#### 문제 2: DB 스키마 없음 (`schema "temp_schema" does not exist`)
- **문제**  
  - `Item` 모델이 다음과 같이 정의되어 있었음:
    ```python
    __table_args__ = {"schema": "temp_schema"}
    ```
  - 컨테이너가 접속하는 DB(`postgresql://postgres:...@postgres-svc:5432/tempdb`) 안에는 `temp_schema` 가 없었음.
  - 앱 시작 시 `Base.metadata.create_all()` 호출에서 `schema "temp_schema" does not exist` 발생 → 애플리케이션이 바로 종료되고 `CrashLoopBackOff`.

- **해결 방법**  
  - 실제 접속하는 DB (`postgres-svc` 의 `tempdb`) 안에서 스키마 생성:
    ```bash
    kubectl exec -it postgres-0 -- bash
    psql -U postgres -d tempdb
    CREATE SCHEMA IF NOT EXISTS temp_schema;
    ```
  - 이후 Deployment 재시작 시 앱이 정상 기동됨.

---

#### 문제 3: Ingress `rewrite-target` 으로 경로가 `/` 로 바뀌는 위험
- **문제**  
  - 기존 `ingress.yaml`:
    ```yaml
    metadata:
      annotations:
        nginx.ingress.kubernetes.io/rewrite-target: /
    ```
  - 이 설정은 Ingress로 들어오는 모든 경로(`/healthz`, 사용자 API 등)를 백엔드로 전달할 때 `/` 로 재작성할 수 있어, 경로 기반 라우팅이 깨질 위험이 있었음.

- **해결 방법**  
  - `rewrite-target` 어노테이션을 제거해서, 요청 경로를 그대로 백엔드로 전달하도록 수정:
    ```yaml
    metadata:
      name: url-shortener-ingress
      # annotations 제거
    ```
  - 이후 `/healthz`, 사용자 API들이 원래 경로 그대로 동작하도록 만듦.

---

#### 📌 문제 4: NodePort(80:31003)와 Windows Docker 드라이버 환경에서의 직접 접속 실패

- **문제**  
  - `ingress-nginx-controller` 서비스:
    ```text
    TYPE: NodePort, PORT(S): 80:31003/TCP
    ```
  - 의미: **클러스터 내부 서비스 포트 80 ↔ 노드 외부 포트 31003**.  
  - 하지만 Windows + Docker 드라이버 조합에서는 `192.168.49.2:31003` 로 **호스트에서 직접 접속이 되지 않음** (VM 내부에서만 의미 있는 주소).
  - 그래서 `http://192.168.49.2:31003/healthz`, `http://url-shortener.local:31003/healthz` 요청들이 계속 실패.

정리하면 현재 ingress가 외부 요청을 받아 LB 타입의 서비스로 라우팅하고 이 서비스가 하위의 svc, pod에게 라우팅을 하는 구조야. 이걸 만족하려면 ingress-nginx가 현재 LB 타입이어야 하는데 현재 NordePort로 되어 있어서 호스트로 외부에서 요청이 불가능해. (Nortport는 포트가 있어야 함). 그래서 ingres-nginx 타입을 LB로 변경하거나 NortPort를 사용한 외부 요청방식을 사용해야 함. 
만약, ingress-nginx를 LB타입으로 변경하지 않는다면 url-shortener-api 서비스에 직접 접근해야 해. 그런데 이 서비스의 타입이 ClusterIp로 되어 있어서 또 외부에서 접근이 불가능함. 여기서 할 수 있는건 타입을 NortPort로 변경하거나 다른 방법을 찾는 것임. 만약 NortPort로 변경하면 Port를 사용해 직접 접속할 수 있음. 다른 방법은 **minikube service -n ingress-nginx ingress-nginx-controller** 이걸 사용하는 것임. 이 명령어는 내부적으로 Minikube가 **NodePort**를 찾아서 외부에서 접속할 수 있는 브라우저 URL를 출력함. 중요한건 외부 요청이 ingress-nginx 즉, ingress를 통해 들어올 수 있는 경로를 만들어준다는 것임. 또한 타입이 ClusterIP라면 minikube가 **임시 NodePort 터널** 만들어서 접속 가능한 URL를 출력함. 이걸 사용하면 접속 가능.
(로컬에서 host 도메인으로 요청을 하기 위해서는 로컬과 도메인간 맵핑 관계가 /etc/hosts 파일에 기입이 되어 있어야 함. 기입 후 테스트 하면 성공함.)
![[Pasted image 20260320152000.png]]

![[Pasted image 20260320152454.png]]

이 둘의 ConentType이 다른 이유는 ingress 규칙에 맞으면 200ok를 반환하고 그렇지 않으면 빈화면을 반환하기 때문이다.



- **해결 방법**  
  - NodePort 를 직접 쓰는 대신, minikube 가 제공하는 터널/포트포워딩 기능 사용:
    ```bash
    minikube service -n ingress-nginx ingress-nginx-controller
    ```
  - 이 명령이 실행된 터미널을 열어둔 상태에서, `127.0.0.1:60143` 같은 **로컬 포트**로 Ingress 에 접속:
    ```bash
    curl.exe http://127.0.0.1:60143/healthz
    ```
  - 이렇게 해서 Windows 환경에서도 안전하게 Ingress 에 접근 가능해짐.

> [!NOTE] 왜 `192.168.49.2:31003`은 안 되는가
>
> - **minikube**는 Docker 안에서 **작은 Linux VM 하나를 실행**한다고 보면 된다.
> - `192.168.49.2:31003` 주소는 **그 Linux VM 내부에서만 의미 있는 주소**다.
> - **Windows 호스트에서는 VM 내부 포트로 직접 접근할 수 없다.**
>   >윈도우 호스트 입장에서는 그 네트워크에 직접 연결된 인터페이스가 없음
> - 그래서 **minikube가 Windows 쪽 `127.0.0.1:50914` 같은 포트로 터널을 열어 접근할 수 있게 해준다.**
>   
>   >그래서 어떻게 우회를 해주나?
>   >Docker/WSL2 환경에서 호스트 → 클러스터 트래픽을 열어주기 위해  
>   >`minikube service` 나 `minikube tunnel` 이 중간에 프록시/포트포워딩 역할을 합니다.

---

#### 문제 5: PowerShell 의 `curl` 이 실제 curl 이 아니어서 옵션 에러 발생
- **문제**  
  - PowerShell 에서 `curl` 은 리눅스 `curl` 이 아니라 `Invoke-WebRequest` 의 별칭.  
  - `curl -v ... -H "Host: url-shortener.local"` 같은 리눅스 스타일 옵션 사용 시:
    - `-H` 옵션이 `Headers` 파라미터로 해석되다가 타입 불일치 에러 발생.

- **해결 방법**  
  - PowerShell에서 **실제 curl.exe** 를 명시적으로 호출:
    ```powershell
    curl.exe -H "Host: url-shortener.local" http://127.0.0.1:60143/healthz
    ```
  - 이렇게 해서 `-H`, `-v` 등 표준 curl 옵션을 정상적으로 사용할 수 있게 함.

---

#### 문제 6: 호스트 이름(`url-shortener.local`)을 쓰고 싶은데 어디로 붙여야 할지 혼란
- **문제**  
  - 처음에는 `url-shortener.local` 을 minikube IP(`192.168.49.2`) 에 매핑하고, NodePort(31003)로 접근하려 했으나:
    - 위에서 설명한 것처럼 NodePort가 호스트에서 직접 열려 있지 않아 실패.
  - 어떤 IP/포트 조합을 `hosts` 에 써야 하는지, 어떤 URL을 브라우저/`curl` 에서 써야 하는지 혼란.

- **해결 방법**  
  - `hosts` 를 **터널 주소 기준**으로 단순화:
    ```text
    127.0.0.1 url-shortener.local
    ```
  - 터널은 `minikube service -n ingress-nginx ingress-nginx-controller` 가 `127.0.0.1:60143` 같은 포트를 열어 줌.
  - 실제 호출은:
    ```powershell
    curl.exe http://url-shortener.local:60143/healthz
    ```
  - 즉,
    - 도메인 → `127.0.0.1` (hosts)
    - 포트 → minikube 가 열어 준 로컬 포트  
    조합으로, “호스트 이름을 사용하는” 형태를 완성.
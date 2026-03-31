## 한 줄 정의

**Docker Hub** — 인터넷에 공개된 공용 컨테이너 이미지 저장소. 누구나 이미지를 올리고 받을 수 있습니다.

**ECR (Elastic Container Registry)** — AWS가 제공하는 프라이빗 컨테이너 이미지 저장소. AWS 계정 안에 격리되어 있습니다.

---

## 핵심 차이

**접근 제어**

```
Docker Hub
  - 기본이 퍼블릭 (누구나 pull 가능)
  - 프라이빗은 유료
  - 별도 계정/비밀번호 관리 필요

ECR
  - 기본이 프라이빗 (IAM으로만 접근)
  - 별도 계정 없음, IAM 권한으로 제어
  - AWS 계정 외부에서는 접근 불가
```

**인증 방식**

```
Docker Hub
  docker login -u username -p password

ECR
  aws ecr get-login-password | docker login
  → IAM 자격증명 기반, 12시간짜리 임시 토큰
  → 비밀번호를 코드에 저장할 필요 없음
```

**EKS와의 연동**

```
Docker Hub → EKS
  - imagePullSecrets 설정 필요
  - K8s Secret에 Docker 로그인 정보 저장
  - 인터넷을 거쳐서 pull (NAT GW 통과)
  - Rate Limit 있음 (익명: 100회/6시간)

ECR → EKS
  - NodeGroup IAM Role에 ECRReadOnly 정책만 있으면 됨
  - imagePullSecrets 불필요
  - VPC Endpoint 설정 시 NAT GW 없이 내부 통신
  - Rate Limit 없음
```

---

## Week4 구조에서 왜 ECR인가

지금 Worker Node가 Private Subnet에 있습니다. Docker Hub에서 이미지를 받으려면 반드시 NAT GW를 거쳐야 합니다.

```
Private Subnet Worker Node
  → NAT GW → IGW → 인터넷 → Docker Hub
  → 데이터 전송 요금 발생
  → Rate Limit 걸릴 수 있음

Private Subnet Worker Node
  → VPC Endpoint → ECR (AWS 내부망)
  → NAT GW 불필요
  → 요금 절감, 빠름, 보안
```

또한 Week4에서 NodeGroup IAM Role에 `AmazonEC2ContainerRegistryReadOnly` 정책을 붙인 이유가 바로 이겁니다. 이 정책 하나로 별도 로그인 없이 ECR에서 이미지를 pull할 수 있습니다.

---

## 언제 뭘 쓰나

```
Docker Hub 사용
  - 오픈소스 공개 이미지 배포 (nginx, redis 등 공식 이미지)
  - 개인 프로젝트, 빠른 테스트
  - AWS 외 환경 (GCP, 온프레미스 등)

ECR 사용
  - 회사 내부 프라이빗 이미지
  - EKS에 배포하는 모든 앱 이미지
  - 보안 정책상 이미지가 외부 노출되면 안 될 때
  - 이미지 취약점 스캔이 필요할 때 (ECR 기본 제공)
```

---

## 실무 패턴

실무에서는 둘을 함께 씁니다.

```
베이스 이미지    → Docker Hub에서 pull  (node:18-alpine, python:3.11 등)
     ↓
  빌드 (CI/CD)
     ↓
앱 이미지 완성  → ECR에 push
     ↓
EKS Worker Node → ECR에서 pull
```

Docker Hub의 공식 이미지를 기반으로 앱을 빌드한 뒤, 완성된 이미지는 ECR에 올려서 EKS가 가져가는 흐름입니다.

### 실무 전체 흐름

```
개발자가 코드 작성
    ↓
Git push → GitHub
    ↓
CI/CD (GitHub Actions) 자동 실행
    ↓
① Docker Hub에서 베이스 이미지 pull
    ↓
② 앱 코드와 합쳐서 빌드
    ↓
③ 완성된 이미지를 ECR에 push
    ↓
④ EKS Worker Node가 ECR에서 pull
    ↓
Pod 실행
```

> [!NOTE] ECR만 사용하면 안되나? 베이스 이미지를 ECR에 미리 넣어 놓으면 되잖아.
> 결론부터 말하면 **ECR만 써도 됩니다.** Docker Hub를 쓰는 건 선택이지 필수가 아닙니다. 📦
>
> ---
>
> ## Docker Hub를 쓰는 실질적 이유
>
> **단순히 공식 이미지가 거기 있기 때문입니다.**
>
> `node:18-alpine`, `python:3.11`, `nginx:latest` 같은 공식 이미지들은 원래부터 Docker Hub에 올라와 있습니다. 이걸 ECR로 옮기려면 누군가 직접 pull → ECR push 작업을 해야 합니다. 그냥 Docker Hub에서 바로 가져다 쓰는 게 편하니까 쓰는 겁니다.
>
> ```
> Docker Hub 사용 시:
> FROM node:18-alpine   ← 그냥 바로 됨
>
> ECR만 사용 시:
> ① docker pull node:18-alpine          (Docker Hub에서)
> ② docker tag node:18-alpine \
>    123456789.dkr.ecr.../node:18-alpine (ECR 주소로 태그)
> ③ docker push 123456789.dkr.ecr.../node:18-alpine  (ECR에 올림)
> ④ FROM 123456789.dkr.ecr.../node:18-alpine  ← 이제 ECR에서 가능
> ```
>
> 추가 작업이 생기긴 하지만 불가능하지는 않습니다.
>
> ---
>
> ## 그럼 언제 ECR로 베이스 이미지까지 옮기나
>
> 실제로 이걸 하는 경우가 있습니다. 🔐
>
> **Docker Hub Rate Limit 문제**  
> CI/CD가 많이 돌아가는 큰 팀에서는 Docker Hub의 무료 Rate Limit(익명 100회/6시간)에 걸립니다. 이미지 pull이 실패하면 빌드가 깨집니다. 이를 피하려고 자주 쓰는 베이스 이미지를 ECR에 미러링해두는 패턴을 씁니다.
>
> **완전한 보안 격리가 필요할 때**  
> 인터넷 접근이 전혀 안 되는 폐쇄망 환경에서는 Docker Hub 자체에 접근이 불가합니다. 이 경우 모든 이미지를 ECR에 올려두고 내부에서만 사용합니다.
>
> **이미지 출처를 통제하고 싶을 때**  
> 보안 정책상 검증된 이미지만 사용해야 하는 경우, Docker Hub 이미지를 그대로 쓰는 게 아니라 ECR로 가져와서 취약점 스캔 후 사용합니다.
>
> ---
>
> ## 결론
>
> ```
> 일반적인 경우 (스타트업, 소규모 팀)
>   베이스 이미지  →  Docker Hub (공식 이미지 그대로 사용)
>   앱 이미지      →  ECR (회사 코드 보호)
>
> 대규모 팀 / 보안 강화 / 폐쇄망
>   베이스 이미지  →  ECR에 미러링 후 사용
>   앱 이미지      →  ECR
> ```
>
> Week4 수준에서는 Docker Hub에서 베이스 이미지를 받고 ECR에 앱 이미지를 올리는 패턴이 가장 일반적이고 충분합니다. ✅


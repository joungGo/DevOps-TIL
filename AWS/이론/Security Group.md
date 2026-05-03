AWS에서 SG는 Amazon Web Services 의 **Security Group** 을 의미해.

쉽게 말하면:

> EC2, RDS 같은 리소스에 붙는 “가상 방화벽”

이야.

---

# 핵심 역할

SG는:

- 어떤 트래픽을 허용할지
    
- 어떤 포트로 접근 가능한지
    
- 누가 접근 가능한지
    

를 제어해.

즉,

```text
"이 서버에 누가 들어올 수 있나?"
```

를 관리하는 보안 장치라고 보면 돼.

---

# 예시

EC2 서버 하나가 있다고 가정하면:

|설정|의미|
|---|---|
|TCP 22 허용|SSH 접속 가능|
|TCP 80 허용|HTTP 웹 접속 가능|
|TCP 443 허용|HTTPS 접속 가능|

예를 들어:

```text
Inbound Rule
- 22 port : 내 IP만 허용
- 80 port : 전체 허용
```

이면:

- 개발자만 SSH 가능
    
- 모든 사용자는 웹 접속 가능
    

이라는 의미야.

---

# 주요 특징

## 1. 상태 저장(Stateful)

가장 중요함.

예를 들어:

- Inbound 허용만 해도
    
- 응답 패킷은 자동 허용됨
    

즉:

```text
요청 허용 → 응답 자동 허용
```

이라서 Outbound를 따로 열 필요가 없는 경우가 많아.

---

# 2. Allow만 가능

SG는:

- 허용(Allow)만 가능
    
- 차단(Deny) 불가능
    

즉:

```text
허용 안 한 건 자동 차단
```

방식이야.

---

# 3. 리소스 단위로 적용

SG는 보통:

- EC2
    
- RDS
    
- EKS Node
    
- Load Balancer
    

같은 리소스에 붙어.

---

# NACL과 차이 (면접 단골)

|항목|SG|NACL|
|---|---|---|
|적용 위치|인스턴스 수준|서브넷 수준|
|상태 저장|O|X|
|Allow/Deny|Allow만|Allow + Deny|
|보통 사용|일반적인 접근 제어|네트워크 레벨 제어|

---

# DevOps / Kubernetes 관점

EKS에서 많이 나오는 예시:

```text
ALB SG
  ↓
Node SG
  ↓
Pod
```

- ALB는 443 허용
    
- Node SG는 ALB에서 오는 트래픽만 허용
    
- DB SG는 Backend SG만 허용
    

이런 식으로 계층적으로 보안을 구성해.

---

# 면접용 답변

“Security Group은 AWS 리소스에 적용되는 상태 저장(Stateful) 기반의 가상 방화벽입니다. 인바운드와 아웃바운드 트래픽을 제어하며, 특정 포트와 IP 대역에 대해 접근을 허용할 수 있습니다. 주로 EC2, RDS, EKS 등에 적용되며, 허용하지 않은 트래픽은 기본적으로 차단됩니다.”
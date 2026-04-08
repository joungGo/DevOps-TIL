## ARN (Amazon Resource Name)

AWS의 **모든 리소스를 전 세계에서 유일하게 식별**하는 주소입니다.

### 형식

```
arn:aws:<서비스>:<리전>:<계정ID>:<리소스>
```

### 예시

```
arn:aws:iam::123456789012:role/urlShortener-alb-controller-role
     │    │              │   └─ 리소스 (IAM Role 이름)
     │    │              └───── AWS 계정 ID
     │    └────────────────────  서비스 (IAM은 글로벌이라 리전 없음)
     └─────────────────────────  서비스명
```

```
arn:aws:eks:ap-northeast-2:123456789012:cluster/urlShortener
                │
                └── EKS는 리전별 서비스라 리전 포함
```

### 왜 필요한가?

같은 이름의 리소스가 **다른 계정/리전**에 존재할 수 있기 때문에 ARN으로 구분합니다.

```hcl
# Terraform에서 ARN 사용 예시
provider_arn = module.eks.oidc_provider_arn
# → "arn:aws:iam::123456789012:oidc-provider/oidc.eks.ap-northeast-2..."

policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
#                         ^^^ aws = AWS 공식 관리형 정책 (계정ID 없음)
```

### 정리

|상황|예시|
|---|---|
|IAM Role에 Policy 붙일 때|`policy_arn = "arn:aws:iam::aws:policy/..."`|
|IRSA에 OIDC 연결할 때|`provider_arn = module.eks.oidc_provider_arn`|
|다른 계정 리소스 참조할 때|Cross-account 설정|

한마디로 **AWS 리소스의 주민등록번호**입니다.
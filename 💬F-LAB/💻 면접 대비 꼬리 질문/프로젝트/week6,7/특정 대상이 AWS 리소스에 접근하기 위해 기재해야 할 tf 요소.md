크게 4가지 요소가 필요합니다. `github_actions.tf`를 예시로 설명합니다.

---

### 1. 신뢰 관계 (Trust Policy)

**"누가" 이 권한을 사용할 수 있는가**

```hcl
data "aws_iam_policy_document" "github_actions_assume_role" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]  # 또는 sts:AssumeRole

    # 신뢰할 대상 지정
    principals {
      type        = "Federated"                  # OIDC 방식
      identifiers = [aws_iam_openid_connect_provider.github.arn]
    }

    # 추가 조건 (더 좁게 제한)
    condition {
      variable = "token.actions.githubusercontent.com:sub"
      values   = ["repo:f-lab-edu/...main"]
    }
  }
}
```

`principals`의 `type`은 대상 종류에 따라 달라집니다.

|type|대상|
|---|---|
|`AWS`|IAM User, IAM Role, AWS 계정|
|`Service`|AWS 서비스 (EC2, Lambda 등)|
|`Federated`|OIDC, SAML (GitHub, Google 등 외부)|

---

### 2. IAM Role

**권한을 담는 그릇**

```hcl
resource "aws_iam_role" "github_actions" {
  name               = "urlshortener-github-actions-role"
  assume_role_policy = data.aws_iam_policy_document.github_actions_assume_role.json
}
```

신뢰 정책을 붙여서 "이 조건을 만족하면 이 Role을 사용할 수 있다"고 정의합니다.

---

### 3. IAM Policy

**"무엇을" 할 수 있는가**

```hcl
resource "aws_iam_policy" "github_actions" {
  name = "urlshortener-github-actions-policy"

  policy = jsonencode({
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["ecr:PutImage", "ecr:GetAuthorizationToken", ...]
        Resource = "arn:aws:ecr:...:repository/urlshortener"
      }
    ]
  })
}
```

`Action`과 `Resource`를 최소한으로 지정하는 것이 원칙입니다. (Least Privilege)

---

### 4. Role에 Policy 연결

**그릇에 권한을 담는 것**

```hcl
resource "aws_iam_role_policy_attachment" "github_actions" {
  role       = aws_iam_role.github_actions.name
  policy_arn = aws_iam_policy.github_actions.arn
}
```

---

### 전체 구조 요약

```
신뢰 정책 (Trust Policy)
    └ "누가" assume 할 수 있는지 정의
        ↓
IAM Role
    └ 신뢰 정책을 붙인 권한 그릇
        ↓
IAM Policy
    └ "무엇을" 할 수 있는지 정의
        ↓
Role ↔ Policy 연결 (attachment)
```

---

### 대상별 작성 패턴 차이

|대상|신뢰 정책 type|assume action|
|---|---|---|
|GitHub Actions|`Federated` (OIDC)|`AssumeRoleWithWebIdentity`|
|EC2/EKS Pod (IRSA)|`Federated` (OIDC)|`AssumeRoleWithWebIdentity`|
|Lambda|`Service`|`AssumeRole`|
|다른 AWS 계정|`AWS`|`AssumeRole`|
|IAM User|`AWS`|`AssumeRole`|

대상이 바뀌어도 이 4가지 구조는 동일합니다.
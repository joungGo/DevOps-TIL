```bash
# GitHub OIDC Provider 등록 (AWS가 GitHub Actions 토큰을 신뢰하게 됨)  
module "github_oidc_provider" {  
  source  = "terraform-aws-modules/iam/aws//modules/iam-github-oidc-provider"  
  version = "~> 5.0"  
  
  tags = {  
    Name = "${var.project_name}-github-oidc"  
  }  
}  
  
# GitHub Actions가 assume할 IAM Rolemodule "github_actions_role" {  
  source  = "terraform-aws-modules/iam/aws//modules/iam-github-oidc-role" # GitHub Actions용 IAM Role을 생성하는 모듈  
  version = "~> 5.0"  
  
  name = "${var.project_name}-github-actions-role"  
  
  # 보안: 특정 레포지토리의 main 브랜치만 허용  
  # pull_request 이벤트까지 허용하려면 "repo:org/repo:*"으로 변경  
  # TODO: dev, prod 환경 구분하기  
  subjects = [  
    # "repo:f-lab-edu/Url-Shortener-EKS-Platform:ref:refs/heads/main",  
    "repo:f-lab-edu/Url-Shortener-EKS-Platform:ref:refs/heads/feat/week6to7",  
    "repo:f-lab-edu/Url-Shortener-EKS-Platform:pull_request"  
  ]  
  
  policies = {  
    ECRPush = aws_iam_policy.github_actions_ecr.arn # 이 Role에 부여할 권한을 설정해야 함  
  }  
  
  tags = {  
    Name = "${var.project_name}-github-actions-role"  
  }  
}  
  
# ECR push 권한 (모듈이 기본 제공하지 않으므로 직접 정의)  
resource "aws_iam_policy" "github_actions_ecr" {  
  name        = "${var.project_name}-github-actions-ecr-policy"  
  description = "GitHub Actions CI/CD Pipeline - ECR push 권한 설정"  
  
  policy = jsonencode({  
    Version = "2012-10-17" # IAM 정책 문법 버전 -> TODO: 현재 유일한 유효 버전이라는데 재점검 필요  
    Statement = [  
      # ECR login 시 발급되는 권한에 대한 설정  
      {  
        Sid      = "AllowECRAuth"  
        Effect   = "Allow" # 아래 Action의 권한이 허용/거부에 대한 내용인지 결정  
        Action   = ["ecr:GetAuthorizationToken"]  
        Resource = "*" # 권한 적용 대상 - ECR 인증은 레포지토리 단위가 아닌 계정 단위로 동작함!  
      },  
      # 실제 이미지 push 시 필요한 권한에 대해 명세  
      {  
        Sid    = "AllowECRPush"  
        Effect = "Allow"  
        Action = [  
          "ecr:BatchCheckLayerAvailability", # 레이어 존재 여부 확인 (중복 업로드 방지)  
          "ecr:GetDownloadUrlForLayer",      # 레이어 다운로드 URL 조회  
          "ecr:BatchGetImage",               # 이미지 메타데이터 조회  
          "ecr:PutImage",                    # 이미지 push          "ecr:InitiateLayerUpload",         # 레이어 업로드 시작  
          "ecr:UploadLayerPart",             # 레이어 분할 업로드  
          "ecr:CompleteLayerUpload",         # 레이어 업로드 완료  
          "ecr:DescribeRepositories",        # 레포지토리 정보 조회  
          "ecr:ListImages",                  # 이미지 목록 조회  
          "ecr:DescribeImages"               # 이미지 상세 정보 조회  
        ]  
        Resource = "arn:aws:ecr:${var.aws_region}:*:repository/${var.project_name}"  
      }  
    ]  
  })  
}# GitHub OIDC Provider 등록 (AWS가 GitHub Actions 토큰을 신뢰하게 됨)  
module "github_oidc_provider" {  
  source  = "terraform-aws-modules/iam/aws//modules/iam-github-oidc-provider"  
  version = "~> 5.0"  
  
  tags = {  
    Name = "${var.project_name}-github-oidc"  
  }  
}  
  
# GitHub Actions가 assume할 IAM Rolemodule "github_actions_role" {  
  source  = "terraform-aws-modules/iam/aws//modules/iam-github-oidc-role" # GitHub Actions용 IAM Role을 생성하는 모듈  
  version = "~> 5.0"  
  
  name = "${var.project_name}-github-actions-role"  
  
  # 보안: 특정 레포지토리의 main 브랜치만 허용  
  # pull_request 이벤트까지 허용하려면 "repo:org/repo:*"으로 변경  
  # TODO: dev, prod 환경 구분하기  
  subjects = [  
    # "repo:f-lab-edu/Url-Shortener-EKS-Platform:ref:refs/heads/main",  
    "repo:f-lab-edu/Url-Shortener-EKS-Platform:ref:refs/heads/feat/week6to7",  
    "repo:f-lab-edu/Url-Shortener-EKS-Platform:pull_request"  
  ]  
  
  policies = {  
    ECRPush = aws_iam_policy.github_actions_ecr.arn # 이 Role에 부여할 권한을 설정해야 함  
  }  
  
  tags = {  
    Name = "${var.project_name}-github-actions-role"  
  }  
}  
  
# ECR push 권한 (모듈이 기본 제공하지 않으므로 직접 정의)  
resource "aws_iam_policy" "github_actions_ecr" {  
  name        = "${var.project_name}-github-actions-ecr-policy"  
  description = "GitHub Actions CI/CD Pipeline - ECR push 권한 설정"  
  
  policy = jsonencode({  
    Version = "2012-10-17" # IAM 정책 문법 버전 -> TODO: 현재 유일한 유효 버전이라는데 재점검 필요  
    Statement = [  
      # ECR login 시 발급되는 권한에 대한 설정  
      {  
        Sid      = "AllowECRAuth"  
        Effect   = "Allow" # 아래 Action의 권한이 허용/거부에 대한 내용인지 결정  
        Action   = ["ecr:GetAuthorizationToken"]  
        Resource = "*" # 권한 적용 대상 - ECR 인증은 레포지토리 단위가 아닌 계정 단위로 동작함!  
      },  
      # 실제 이미지 push 시 필요한 권한에 대해 명세  
      {  
        Sid    = "AllowECRPush"  
        Effect = "Allow"  
        Action = [  
          "ecr:BatchCheckLayerAvailability", # 레이어 존재 여부 확인 (중복 업로드 방지)  
          "ecr:GetDownloadUrlForLayer",      # 레이어 다운로드 URL 조회  
          "ecr:BatchGetImage",               # 이미지 메타데이터 조회  
          "ecr:PutImage",                    # 이미지 push          "ecr:InitiateLayerUpload",         # 레이어 업로드 시작  
          "ecr:UploadLayerPart",             # 레이어 분할 업로드  
          "ecr:CompleteLayerUpload",         # 레이어 업로드 완료  
          "ecr:DescribeRepositories",        # 레포지토리 정보 조회  
          "ecr:ListImages",                  # 이미지 목록 조회  
          "ecr:DescribeImages"               # 이미지 상세 정보 조회  
        ]  
        Resource = "arn:aws:ecr:${var.aws_region}:*:repository/${var.project_name}"  
      }  
    ]  
  })  
}
```

---

[[특정 대상이 AWS 리소스에 접근하기 위해 기재해야 할 tf 요소]]

---
### 1. OIDC Provider

**OIDC(OpenID Connect)** 는 "나는 누구입니다"를 증명하는 토큰 기반 인증 표준입니다.

```hcl
resource "aws_iam_openid_connect_provider" "github" {
  url            = "https://token.actions.githubusercontent.com"
  client_id_list = ["sts.amazonaws.com"]
}
```

GitHub Actions가 실행되면 GitHub는 자동으로 **JWT 토큰**을 발급합니다. 이 토큰 안에는 아래 정보가 담겨 있습니다.

```
{
  "sub": "repo:f-lab-edu/Url-Shortener-EKS-Platform:ref:refs/heads/main",
  "aud": "sts.amazonaws.com",
  "iss": "https://token.actions.githubusercontent.com"
}
```

이 Terraform 코드는 **"GitHub가 발급한 토큰을 AWS가 신뢰하겠다"** 고 AWS IAM에 등록하는 것입니다. 이게 없으면 AWS는 GitHub 토큰을 모르는 토큰으로 취급해서 거부합니다.

---

### 2. IAM Role은 계정이 아닙니다

IAM에는 두 가지 개념이 있습니다.

|구분|설명|비유|
|---|---|---|
|**IAM User**|고정된 사람/계정. Access Key 발급 가능|사원증|
|**IAM Role**|누군가가 **일시적으로 빌려 쓰는** 권한 묶음|방문증|

```hcl
resource "aws_iam_role" "github_actions" {
  name               = "urlshortener-github-actions-role"
  assume_role_policy = ...  # 누가 이 Role을 사용할 수 있는지 정의
}
```

Role은 계정이 아닙니다. **"이 조건을 만족하면 이 권한을 임시로 빌려줄게"** 라는 정의입니다.

```hcl
# 신뢰 정책: 이 Role을 누가 assume(사용) 할 수 있는가
condition {
  variable = "token.actions.githubusercontent.com:sub"
  values   = ["repo:f-lab-edu/Url-Shortener-EKS-Platform:ref:refs/heads/main"]
}
```

**main 브랜치의 이 레포에서 실행된 GitHub Actions만** 이 Role을 사용할 수 있습니다. PR이나 다른 레포는 사용 불가입니다.

---

### 3. Terraform 코드가 GitHub Actions에 권한으로 연결되는 과정

```
① terraform apply
   └ AWS에 OIDC Provider 등록
   └ IAM Role 생성 (신뢰 정책 포함)
   └ IAM Policy 생성 (ECR push 권한)
   └ Role에 Policy 연결

② GitHub Actions 실행 (.github/workflows/ci-cd.yml)
   └ GitHub가 JWT 토큰 자동 발급

③ AWS OIDC 인증 step
   role-to-assume: arn:aws:iam::716174522908:role/urlshortener-github-actions-role
   └ JWT 토큰을 AWS STS에 제출
   └ AWS가 토큰 검증 (OIDC Provider로 등록된 GitHub 발급 토큰인지 확인)
   └ 신뢰 정책 조건 확인 (main 브랜치인지, 이 레포인지)
   └ 조건 통과 → 임시 자격증명(Access Key + Secret + Session Token) 발급 (1시간 유효)

④ 이후 ECR push 등 AWS API 호출 시
   └ 발급받은 임시 자격증명으로 인증
   └ IAM Policy에 정의된 권한 범위 내에서만 허용
```

**Access Key를 GitHub에 저장하지 않아도 되는 이유**가 여기 있습니다. 토큰은 실행할 때마다 GitHub가 새로 발급하고, AWS가 검증한 뒤 1시간짜리 임시 자격증명을 주기 때문에 유출되어도 만료됩니다.

---

`.github/workflows/ci-cd.yml`과 `github_actions.tf`를 함께 보면서 설명합니다.

---

### 사전 준비 (terraform apply 시 완료된 것들)

```hcl
# AWS에 "GitHub 토큰 발급자를 신뢰한다"고 등록
resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"
}

# "이 조건이면 이 Role을 써도 된다"는 규칙 생성
resource "aws_iam_role" "github_actions" {
  assume_role_policy = ...  # main 브랜치, 이 레포만 허용
}

# Role에 ECR 권한 부여
resource "aws_iam_role_policy_attachment" ...
```

이 3가지가 AWS에 이미 등록된 상태입니다.

---

### Step 1: GitHub Actions 워크플로우 트리거

```yaml
on:
  push:
    branches: [main, feat/week6to7]
```

main 브랜치에 push가 발생하면 GitHub가 Runner(실행 환경)를 띄웁니다.

---

### Step 2: GitHub가 JWT 토큰 자동 발급

Runner가 시작되면 GitHub는 **이 실행에 대한 신원 증명서(JWT 토큰)** 를 자동으로 생성합니다.

```
JWT 토큰 내부 내용:
{
  "iss": "https://token.actions.githubusercontent.com",  # 발급자: GitHub
  "aud": "sts.amazonaws.com",                            # 사용 대상: AWS STS
  "sub": "repo:f-lab-edu/Url-Shortener-EKS-Platform:ref:refs/heads/main",
                                                          # 주체: 이 레포, main 브랜치
  "exp": 1234567890                                       # 만료 시간
}
```

이 토큰은 Runner의 환경변수에 자동으로 주입됩니다. 워크플로우에서 아래 권한이 있어야 발급됩니다.

```yaml
permissions:
  id-token: write   ← 이게 없으면 JWT 토큰 발급 안 됨
  contents: write
```

---

### Step 3: AWS OIDC 인증 Step 실행

```yaml
- name: AWS OIDC 인증
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::716174522908:role/urlshortener-github-actions-role
    aws-region: ap-northeast-2
```

이 step이 실행되면 내부적으로 아래 과정이 진행됩니다.

```
① Runner가 GitHub에 JWT 토큰 요청
        ↓
② GitHub가 JWT 토큰 발급 (Step 2의 토큰)
        ↓
③ Runner가 AWS STS에 아래 요청 전송
   POST https://sts.amazonaws.com
   {
     "Action": "AssumeRoleWithWebIdentity",
     "RoleArn": "arn:aws:iam::716174522908:role/urlshortener-github-actions-role",
     "WebIdentityToken": "eyJ..." (JWT 토큰)
   }
```

---

### Step 4: AWS STS가 토큰 검증

AWS STS는 JWT 토큰을 받아서 아래 3가지를 순서대로 검증합니다.

```
검증 1: 발급자(iss) 신뢰 여부
  "https://token.actions.githubusercontent.com"
  → terraform으로 등록한 OIDC Provider와 일치? ✅

검증 2: 대상(aud) 확인
  "sts.amazonaws.com"
  → 신뢰 정책의 condition과 일치? ✅
  condition {
    variable = "token.actions.githubusercontent.com:aud"
    values   = ["sts.amazonaws.com"]
  }

검증 3: 주체(sub) 확인
  "repo:f-lab-edu/Url-Shortener-EKS-Platform:ref:refs/heads/main"
  → 신뢰 정책의 condition과 일치? ✅
  condition {
    variable = "token.actions.githubusercontent.com:sub"
    values   = ["repo:f-lab-edu/...:ref:refs/heads/main"]
  }
```

3가지 모두 통과하면 임시 자격증명을 발급합니다.

```
AWS STS 응답:
{
  "AccessKeyId":     "ASIA...",   # 임시 Access Key
  "SecretAccessKey": "xxxx...",   # 임시 Secret Key
  "SessionToken":    "yyyy...",   # 세션 토큰
  "Expiration":      "1시간 후"   # 만료 시간
}
```

---

### Step 5: 임시 자격증명으로 AWS 리소스 접근

발급받은 임시 자격증명이 Runner의 환경변수에 자동으로 세팅됩니다.

```
AWS_ACCESS_KEY_ID=ASIA...
AWS_SECRET_ACCESS_KEY=xxxx...
AWS_SESSION_TOKEN=yyyy...
```

이후 ECR push 등 모든 AWS API 호출은 이 자격증명으로 인증됩니다.

```yaml
- name: ECR 로그인
  uses: aws-actions/amazon-ecr-login@v2
  # 환경변수의 임시 자격증명으로 자동 인증

- name: 이미지 빌드 및 ECR 푸시
  run: docker push ...
  # IAM Policy에 정의된 ecr:PutImage 권한으로 push 가능
```

---

### 전체 흐름 요약

```
push 이벤트 발생
    ↓
GitHub: Runner 실행
    ↓
GitHub: JWT 토큰 자동 발급 (id-token: write 권한 필요)
    ↓
Runner → AWS STS: "이 JWT로 이 Role 사용하고 싶어"
    ↓
AWS STS: JWT 검증
    ├ OIDC Provider 등록 여부 확인  (terraform: aws_iam_openid_connect_provider)
    ├ aud 조건 확인                 (terraform: condition aud)
    └ sub 조건 확인                 (terraform: condition sub 레포/브랜치)
    ↓
AWS STS: 임시 자격증명 발급 (유효기간 1시간)
    ↓
Runner: 임시 자격증명으로 ECR push 등 AWS API 호출
    └ IAM Policy 범위 내에서만 허용 (terraform: aws_iam_policy)
```

**핵심**: Access Key를 GitHub에 저장하지 않아도 됩니다. JWT 토큰은 실행할 때마다 새로 발급되고, 임시 자격증명은 1시간 후 자동 만료됩니다.
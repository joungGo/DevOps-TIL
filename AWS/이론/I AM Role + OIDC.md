### 핵심

- **AWS IAM Role + OpenID Connect 방식**  
    → **외부 서비스가 AWS에 “임시 권한”으로 접근하는 인증 방식**
    

> [[OIDC]] 는 인증 프로토콜이다.
> 보통, ServiceAccount 토큰을 생성하면 이 토큰을 가지고 OIDC 프로토콜을 사용해 OIDC Provider가 검증을 진행한다.

---

동작 흐름

1. 외부 시스템 (예: GitHub Actions, Kubernetes)
    
2. OIDC 토큰 발급
    
3. AWS IAM Role에 전달
    
4. AWS가 토큰 검증
    
5. **임시 권한(STS)** 발급
    

---

왜 쓰냐

- Access Key 필요 없음
    
- 보안 좋음 (임시 권한)
    
- 자동화에 적합
    

---

대표 사용 예

- GitHub Actions → AWS 배포
    
- EKS ServiceAccount → AWS 리소스 접근
    

특히 EKS에서 많이 쓰는 형태:

- Pod
    
- ServiceAccount
    
- OIDC 연결
    
- IAM Role 사용
    

정리

- IAM Role = 권한
    
- OIDC = 인증
    
- 합치면 = 외부에서 AWS 안전하게 접근
    

---

### 언제, 어떻게 사용할까?

#### 1️⃣ GitHub Actions → Amazon EKS 배포

#### 목적

GitHub Actions가 AWS 접근할 때  
**Access Key 없이 IAM Role 임시 권한 사용**

#### 전체 흐름

1. GitHub Actions 실행
    
2. GitHub가 **OIDC 토큰 발급**
    
3. AWS에 "이 토큰으로 Role 사용 가능?" 요청
    
4. AWS가 OIDC 검증
    
5. **IAM Role 임시 credential 발급**
    
6. EKS / ECR 접근
    

---

#### 설정 단계

#### ① AWS에 OIDC Provider 생성

GitHub OIDC 등록

```
https://token.actions.githubusercontent.com
```

---

#### ② IAM Role 생성 (trust policy)

예시:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:sub": "repo:OWNER/REPO:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

👉 main 브랜치에서만 허용

---

#### ③ Role에 권한 붙이기

예:

- EKS 접근
    
- ECR push
    
- kubectl 사용
    

---

#### ④ GitHub Actions workflow 설정

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: actions/checkout@v4

  - name: Configure AWS credentials
    uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::ACCOUNT_ID:role/github-actions-role
      aws-region: ap-northeast-2
```

이제 Access Key 없이 AWS 접근됨

---

#### 2️⃣ Amazon EKS Pod → AWS 접근 (IRSA)

이건 **Pod가 S3 / DynamoDB 접근할 때** 사용

#### 흐름

1. Pod 실행
    
2. ServiceAccount에 IAM Role 연결
    
3. Pod가 OIDC 토큰 생성
    
4. AWS STS 호출
    
5. 임시 credential 발급
    
6. AWS API 호출
    

> [!info] 왜 Pod에서 AWS로 접근시 인증이 필요할까?
> 
> **Pod는 AWS 입장에서 그냥 외부 서버**
> 
> 아무 인증 없이 접근 허용하면  
> → 누구나 AWS 리소스 접근 가능
> 
> 그래서 AWS는 반드시 확인함:
> - 누가 요청했는지
> - 어떤 권한 있는지
> 
> 즉 흐름:
> 
> Pod → AWS 요청  
> AWS → "너 누구야?"
> 
> 그래서 **IAM Role 필요**
> 
> ---
> 
> **왜 Pod가 AWS 입장에서 “외부”인가**
> 
> Amazon EKS 는  
> 쿠버네티스 **Control Plane만 AWS가 관리**
> 
> Pod는 어디서 실행되냐?  
> → EC2 Node 위에서 실행됨
> 
> 구조:
> 
> AWS  
> └─ EKS Control Plane (AWS 관리)  
> └─ EC2 Node (고객 계정)  
> 　   └─ Pod (컨테이너)
> 
> AWS 서비스 (예: S3) 입장에서는
> - EC2 내부 프로세스인지
> - Docker 컨테이너인지
> - Kubernetes Pod인지
> 
> **구분 불가능**
> 
> 단순히:
> 
> 어떤 IP에서 API 요청 옴
> 
> 이렇게만 보임
---

#### 설정 단계

#### ① EKS OIDC provider 생성

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --approve
```

---

#### ② IAM Role 생성 (trust policy)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/oidc.eks.ap-northeast-2.amazonaws.com/id/XXXXX"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.ap-northeast-2.amazonaws.com/id/XXXXX:sub": "system:serviceaccount:default:my-sa"
        }
      }
    }
  ]
}
```

---

#### ③ ServiceAccount 생성

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/eks-role
```

---

#### ④ Pod에서 사용

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test
spec:
  serviceAccountName: my-sa
  containers:
  - name: app
    image: nginx
```

이제 Pod에서 AWS SDK 바로 사용 가능

---

#### 핵심 차이

|구분|GitHub Actions|EKS|
|---|---|---|
|주체|CI/CD|Pod|
|토큰 발급|GitHub|Kubernetes|
|Role 연결|workflow|ServiceAccount|
|목적|배포|AWS 리소스 접근|

---

#### 가장 많이 쓰는 실제 예

GitHub Actions

- ECR push
    
- kubectl apply
    

EKS

- Pod → S3 읽기
    
- Pod → Secrets Manager 읽기
    
- Pod → DynamoDB 쓰기
    

---

#### 한 줄 정리

- GitHub Actions → 배포용 인증
    
- EKS IRSA → Pod AWS 접근용 인증
    

지금 구성하려는 게 뭐야?

- GitHub → EKS 배포?
    
- Pod → S3 접근?
    
- 둘 다?
    

말해주면 YAML까지 맞춰서 바로 만들어 줄게.
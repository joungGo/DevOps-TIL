## Terraform 코드 비교

### AES256

```hcl
resource "aws_ecr_repository" "app" {
  name = "app"

  encryption_configuration {
    encryption_type = "AES256"
  }
}
```

### KMS

```hcl
resource "aws_ecr_repository" "app" {
  name = "app"

  encryption_configuration {
    encryption_type = "KMS"
    kms_key         = aws_kms_key.ecr.arn # 내가 만든 resource 타입의 key 이름
  }
}
```

---

## 핵심 차이

|항목|AES256|KMS|
|---|---|---|
|키 관리|AWS 자동|사용자가 직접|
|권한 제어|❌ 불가능|✅ 가능|
|IAM 정책 적용|❌|✅|
|특정 Role만 pull 제한|❌|✅|
|감사 로그 (CloudTrail)|제한적|상세|
|설정 난이도|쉬움|약간 복잡|
|추가 비용|없음|있음|

---

## 권한 제어 차이 (핵심)

### AES256

```text
누가 ECR pull 하든 → 자동 복호화
제어 불가
```

즉 IAM으로 **ECR 접근만 제어 가능**  
암호화 키 자체 제어는 못함

---

### KMS

```text
KMS 정책으로 제한 가능

허용된 Role → pull 가능
허용 안된 Role → pull 실패
```

Terraform 예시

```hcl
resource "aws_kms_key" "ecr" {
  policy = jsonencode({
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          AWS = aws_iam_role.eks_node.arn
        }
        Action = [
          "kms:Decrypt"
        ]
        Resource = "*"
      }
    ]
  })
}
```

➡️ 특정 Role만 이미지 pull 가능

---

## 언제 사용?

### AES256

- 개인 프로젝트
    
- 테스트 환경
    
- 단순 구성
    

### KMS

- 운영 환경
    
- 계정 간 접근 제어 필요
    
- 보안 요구 있음
    
- EKS / CI Role 제한 필요
    

---

## 한 줄 정리

- AES256 → "암호화만"
    
- KMS → "암호화 + 누가 복호화할지 Terraform으로 제어"
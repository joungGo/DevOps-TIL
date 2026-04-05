`terraform.tfstate` 파일은 인프라 상태를 저장합니다. 여러 사람이 동시에 apply하면 state 파일이 꼬일 수 있어서, 한 번에 한 명만 작업할 수 있도록 Lock을 겁니다. 하지만 이 파일에는 보안적으로 공개되면 안되는 데이터가 있어 Github에 올려서는 안됩니다. 그럼 어떻게 해야할까요?

**아니요!** `terraform.tfstate` 파일은 **절대 GitHub에 올리면 안 됩니다**.

## 이유

```
tfstate 파일에 포함되는 민감 정보:
- DB 비밀번호
- IAM 키
- 리소스 ARN
- Private IP 등
```

GitHub에 올리면 보안 사고 발생할 수 있습니다.

---

## 협업할 때 해결책: Remote Backend

**S3 + DynamoDB 조합** (AWS 사용 시 권장)

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "url-shortener/terraform.tfstate"
    region         = "ap-northeast-2"
    dynamodb_table = "terraform-locks"  # Lock 관리용
    encrypt        = true
  }
}
```

이렇게 하면:

- State 파일은 **S3에 안전하게 저장**
- DynamoDB로 **Lock 관리** (동시 작업 방지)
- 팀원 모두 같은 state 공유 가능

---

## GitHub에 올릴 것 / 올리지 말 것

|파일|GitHub|이유|
|---|---|---|
|`*.tf`|✅ 올림|인프라 코드|
|`terraform.tfstate`|❌ 절대 안됨|민감정보|
|`.terraform/`|❌ 안됨|로컬 캐시|
|`*.tfvars`|⚠️ 주의|비밀번호 있으면 안됨|

`.gitignore` 예시:

```
.terraform/
*.tfstate
*.tfstate.*
*.tfvars
```
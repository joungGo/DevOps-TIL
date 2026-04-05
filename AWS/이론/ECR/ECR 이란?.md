## Amazon Elastic Container Registry (ECR) 한 줄 정의

**컨테이너 이미지(Docker 이미지)를 저장하는 AWS의 관리형 레지스트리 서비스**  
→ Terraform + EKS/ECS 배포 시 **이미지 저장소 역할**

AWS 공식 설명도 “안전하고 확장 가능한 컨테이너 이미지 레지스트리 서비스”라고 정의합니다. ([AWS Documentation](https://docs.aws.amazon.com/ko_kr/AmazonECR/latest/userguide/what-is-ecr.html?utm_source=chatgpt.com "Amazon Elastic Container Registry란 무엇입니까? - Amazon ECR"))

---

# 왜 필요해? (Terraform 관점)

Terraform으로 인프라 만들면 흐름이 이렇게 됩니다:

```
docker build → ECR push → EKS/ECS에서 pull → Pod 실행
```

즉,

- 코드 → 이미지 생성
    
- 이미지를 **ECR에 저장**
    
- Kubernetes/ECS가 **ECR에서 pull 해서 실행**
    

---

# 핵심 기능

- Docker/OCI 이미지 저장 및 관리
    
- IAM 기반 접근 제어
    
- EKS/ECS와 자동 연동
    
- HTTPS 전송 + 저장 시 암호화
    
- 멀티 리전 복제 가능
    

AWS 문서에서도 ECR은 이미지 저장, 접근 제어, 암호화 등을 제공한다고 설명합니다. ([Amazon Web Services, Inc.](https://aws.amazon.com/ecr/features/?utm_source=chatgpt.com "Amazon ECR Features | Docker Container Registry | Amazon Web Services"))

---

# encryption_type = "AES256" 와 연결

ECR 이미지 저장 시 **기본 암호화 방식**이 바로 AES256입니다.

- `AES256` → AWS가 관리하는 기본 키로 자동 암호화
    
- `KMS` → 사용자가 직접 KMS 키 지정
    

기본 동작:

> AES256 사용 시 Amazon S3 서버측 암호화로 저장 시 자동 암호화됨 ([AWS Documentation](https://docs.aws.amazon.com/es_es/AmazonECR/latest/APIReference/API_EncryptionConfiguration.html?utm_source=chatgpt.com "EncryptionConfiguration - Amazon Elastic Container Registry"))

---

# Terraform 예시

```hcl
resource "aws_ecr_repository" "app" {
  name = "my-app"

  encryption_configuration {
    encryption_type = "AES256"
  }
}
```

---

# Docker Hub vs ECR

|항목|ECR|Docker Hub|
|---|---|---|
|AWS 연동|매우 좋음|없음|
|IAM 권한|가능|없음|
|Private registry|기본|유료|
|EKS 사용|최적|가능하지만 비효율|

---

# 한 줄 정리

ECR = **Terraform으로 만든 EKS/ECS가 사용할 Docker 이미지 저장소**

---

원하면 바로 이어서:

- Terraform으로 ECR 생성
    
- EKS에서 ECR 이미지 사용하는 코드  
    까지 한 번에 보여줄게.
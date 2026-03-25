1. I AM 계정 생성
	- 사용자 이름: admin-user-pjh
	- 사용자 그룹 이름: admin-group
		- 연결된 정책: AdminstratorAccess
		- IAMUserChangePassword
2. AWS CLI 설치 및 aws configure 설정
3. terraform 폴더 및 파일 구조 설계
```markdown
Url-Shortener-EKS-Platform/
├── url-shortener/
└── terraform/                     # 인프라 코드 (Week4)
    ├── main.tf                   # 리소스 정의 (VPC, EKS 등)
    ├── variables.tf              # 변수 선언
    ├── outputs.tf                # 출력 값 정의
    ├── providers.tf              # AWS provider 설정
    ├── terraform.tfvars          # 실제 변수 값 (gitignore 권장)
    └── iam_oidc.tf               # OIDC Provider (IRSA 설정 시)
```

4. terraform 코드 작성
	- main.tf는 수동 작성 후 모듈 작성 전환 진행

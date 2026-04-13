### 면접 답변

ALB Controller 설치는 **Terraform(AWS 영역)** 과 **Helm(Kubernetes 영역)** 두 단계로 나뉘며, 연결 고리는 **ServiceAccount 이름과 IAM Role ARN** 입니다.

---

**1단계: Terraform으로 AWS IAM 준비**

```hcl
module "alb_controller_irsa" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"

  role_name = "${var.project_name}-alb-controller-role"
  attach_load_balancer_controller_policy = true

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["kube-system:aws-load-balancer-controller"]
    }
  }
}
```

`terraform apply` 시 세 가지가 생성됩니다.

```
① IAM Role (urlshortener-alb-controller-role)
② IAM Policy (ALB 생성/삭제 등 권한)
③ 신뢰 정책 (Trust Policy)
   → kube-system 네임스페이스의
     aws-load-balancer-controller ServiceAccount만 이 Role을 assume 가능
```

이 시점에서 AWS에는 권한이 준비되어 있지만 Kubernetes에는 아무것도 없습니다.

---

**2단계: Helm으로 Kubernetes에 설치**

```bash
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set serviceAccount.name=aws-load-balancer-controller \        # ← tf 신뢰 정책의 이름과 일치
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=${ALB_ROLE_ARN}
  #                                                                ← tf가 만든 Role ARN 지정
```

Helm이 하는 일:

```
① K8s ServiceAccount 생성
   name: aws-load-balancer-controller   ← tf 신뢰 정책 조건과 반드시 일치해야 함
   annotation:
     eks.amazonaws.com/role-arn: arn:...urlshortener-alb-controller-role

② ALB Controller Pod 생성
   → 위 ServiceAccount를 사용
```

---

**3단계: Pod 기동 시 IRSA 동작**

```
Pod 기동
  ↓
EKS: ServiceAccount annotation에서 role-arn 확인
  ↓
EKS OIDC Provider → JWT 토큰 발급
  sub: "system:serviceaccount:kube-system:aws-load-balancer-controller"
  ↓
AWS STS: JWT 검증
  → tf 신뢰 정책의 namespace_service_accounts 조건과 일치 ✅
  ↓
임시 자격증명 발급 → Pod 환경변수에 자동 주입
  ↓
ALB Controller Pod가 AWS API 호출 가능
```

---

**핵심 연결 고리**

```
Terraform (AWS)                       Helm (Kubernetes)
──────────────────────────────        ──────────────────────────────
namespace_service_accounts            serviceAccount.name
"kube-system:aws-load-balancer-controller"  =  aws-load-balancer-controller

terraform output alb_controller_role_arn  →  serviceAccount.annotations.role-arn
```

이 두 값이 하나라도 다르면 STS 검증 단계에서 조건 불일치로 `AccessDenied`가 발생하고 ALB Controller는 AWS API를 호출할 수 없게 됩니다.

---

**왜 이 구조를 사용하는가?**

기존 방식은 AWS Access Key를 K8s Secret에 저장해서 Pod에 주입했습니다. 키가 노출되면 영구적으로 위험합니다.

IRSA 방식은 Pod가 실행될 때마다 EKS OIDC가 단기 JWT 토큰을 발급하고, STS가 검증 후 **1시간짜리 임시 자격증명**을 줍니다. 키를 저장하지 않고, 유출되어도 자동 만료되어 보안이 강화됩니다.
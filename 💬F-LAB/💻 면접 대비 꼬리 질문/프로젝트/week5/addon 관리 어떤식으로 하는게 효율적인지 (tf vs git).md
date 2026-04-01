## EKS Addon 관리 — Terraform vs GitOps

---

## Addon이란

EKS 클러스터에 기본 기능 외에 추가로 설치하는 컴포넌트입니다.

```
AWS 관리형 Addon (EKS 콘솔/API로 관리)
  - vpc-cni          → Pod 네트워킹
  - kube-proxy       → Service 라우팅
  - coredns          → 클러스터 DNS
  - aws-ebs-csi-driver → EBS 볼륨 마운트

커뮤니티 Addon (Helm으로 설치)
  - aws-load-balancer-controller → ALB 자동 생성
  - cluster-autoscaler           → Node 자동 스케일
  - metrics-server               → kubectl top 지원
  - argocd                       → GitOps 도구
  - prometheus / grafana         → 모니터링
```

---

## 두 가지 관리 방식

---

### Terraform으로 관리

```hcl
resource "aws_eks_addon" "vpc_cni" {
  cluster_name      = aws_eks_cluster.main.name
  addon_name        = "vpc-cni"
  addon_version     = "v1.16.0-eksbuild.1"
  resolve_conflicts = "OVERWRITE"
}

resource "aws_eks_addon" "coredns" {
  cluster_name  = aws_eks_cluster.main.name
  addon_name    = "coredns"
  addon_version = "v1.11.1-eksbuild.4"
}

resource "aws_eks_addon" "kube_proxy" {
  cluster_name  = aws_eks_cluster.main.name
  addon_name    = "kube-proxy"
  addon_version = "v1.29.1-eksbuild.2"
}
```

**장점**

```
- 클러스터 생성과 Addon 설치가 terraform apply 한 번에 완료
- Addon 버전이 코드로 고정되어 재현 가능
- terraform destroy 시 Addon도 같이 삭제
- 인프라와 Addon이 같은 Git 히스토리에 기록
```

**단점**

```
- Addon 버전 업데이트마다 terraform apply 필요
- 커뮤니티 Addon(Helm)은 aws_eks_addon 리소스 없음
  → helm_release 프로바이더를 섞어야 해서 복잡해짐
- Terraform state가 무거워짐
```

---

### GitOps (ArgoCD/Flux)로 관리

클러스터 자체는 Terraform으로 만들고, 그 위에 올라가는 Addon은 ArgoCD가 Git을 보고 자동 설치합니다.

```
Git 저장소 구조:
infra/
  terraform/          ← 클러스터 생성
    main.tf
    variables.tf

clusters/
  addons/             ← ArgoCD가 이걸 감시
    aws-load-balancer-controller/
      values.yaml
    cluster-autoscaler/
      values.yaml
    prometheus/
      values.yaml
```

Git에 `values.yaml`을 push하면 ArgoCD가 감지해서 Helm으로 자동 설치/업데이트합니다.

**장점**

```
- Addon 업데이트가 PR → merge → 자동 반영
- 드리프트 감지: 누가 콘솔에서 직접 바꾸면 자동 복구
- Helm / Kustomize 모두 지원
- 클러스터가 날아가도 Git에서 전체 복구 가능
- Addon별 독립적인 배포 이력 관리
```

**단점**

```
- ArgoCD 자체를 먼저 설치해야 함 (닭-달걀 문제)
- 초기 설정이 복잡
- 팀이 GitOps 방식에 익숙해야 함
```

---

## 실무에서 가장 많이 쓰는 패턴

둘을 역할에 따라 나눠서 씁니다.

```
Terraform으로 관리 (클러스터 생명주기와 묶인 것들)
  - vpc-cni
  - coredns
  - kube-proxy
  - aws-ebs-csi-driver
  → 클러스터가 생성될 때 반드시 있어야 하는 것들

ArgoCD/Helm으로 관리 (앱 레이어에 가까운 것들)
  - aws-load-balancer-controller
  - cluster-autoscaler
  - metrics-server
  - prometheus / grafana
  - argocd 자체
  → 설치/업데이트가 잦고 앱 배포와 함께 관리되는 것들
```

---

## 실제 적용 순서

```
① terraform apply
   → VPC + EKS 클러스터 생성
   → vpc-cni, coredns, kube-proxy 설치 (Terraform)

② ArgoCD 초기 설치 (한 번만)
   helm install argocd ...

③ ArgoCD가 Git을 감시 시작
   → aws-load-balancer-controller 자동 설치
   → cluster-autoscaler 자동 설치
   → prometheus 자동 설치
   ...

④ 이후 Addon 업데이트
   → Git에서 values.yaml의 버전만 바꾸고 PR
   → merge되면 ArgoCD가 자동 반영
```

---

## 핵심 기준 한 줄 요약

```
"클러스터가 뜰 때 반드시 필요한 것"  →  Terraform
"그 위에 올라가는 것"               →  GitOps (ArgoCD)
```
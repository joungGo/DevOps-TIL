## EKS 버저닝 방법

---

## EKS 버전이란

EKS 버전 = Kubernetes 버전입니다. `1.29`, `1.30`, `1.31` 같은 형태입니다.

```
1  .  29  .  3
↑     ↑      ↑
메이저 마이너  패치
(거의  (매년   (버그
안변함) 2~3개  수정)
       출시)
```

AWS는 보통 Kubernetes 공식 릴리즈 후 몇 달 내에 EKS에서 지원을 시작합니다. 각 버전의 지원 기간은 약 14개월입니다. 지원이 끝나면 AWS가 강제 업그레이드를 진행합니다.

---

## 버전 관리 대상 2가지

EKS에서 버전을 관리해야 하는 대상이 두 개입니다.

```
① Control Plane 버전   →  클러스터 자체의 K8s 버전
② Node AMI 버전        →  Worker Node EC2의 K8s 버전

둘이 항상 같아야 함 (최대 2 마이너 버전 차이까지만 허용)
```

---

## Terraform에서 버전 설정 방법

Week4 코드 기준으로 두 곳에서 설정합니다.

**Control Plane 버전**

```hcl
# variables.tf
variable "eks_cluster_version" {
  default = "1.29"
}

# main.tf
resource "aws_eks_cluster" "main" {
  version = var.eks_cluster_version  # ← 여기
}
```

**Node AMI 버전**

```hcl
resource "aws_eks_node_group" "main" {
  # 명시 안 하면 Control Plane 버전과 동일한 최신 AMI 자동 선택
  # 명시하고 싶으면:
  release_version = "1.29.3-20240928"  # AMI 릴리즈 버전
}
```

---

## 업그레이드 순서 (중요)

반드시 **Control Plane 먼저, Node 나중** 순서를 지켜야 합니다.

```
현재: Control Plane 1.28 / Node 1.28

Step 1: Control Plane 1.28 → 1.29 업그레이드
        (이 시점에 Control Plane=1.29, Node=1.28 혼재 → 허용)

Step 2: NodeGroup AMI 1.28 → 1.29 업그레이드
        (max_unavailable=1 → Rolling Update)

완료: Control Plane 1.29 / Node 1.29
```

반대 순서(Node 먼저)는 지원하지 않습니다.

**마이너 버전은 반드시 한 단계씩만** 올려야 합니다.

```
1.27 → 1.28 → 1.29  (O)
1.27 → 1.29          (X)  스킵 불가
```

> [!NOTE] EKS 마이너 버전 업그레이드 제한 이유 : 왜 한 단계씩만 올려야 할까?
> Kubernetes는 API가 버전별로 단계적으로 deprecated 및 제거되기 때문에 한 번에 여러 마이너 버전을 건너뛰면 리소스 호환성 문제가 발생할 수 있습니다.  
> 또한 Control Plane, Node, Add-on 간 버전 호환성을 보장하기 위해 EKS는 한 번에 하나의 마이너 버전씩만 업그레이드를 허용합니다. ⚙️🔒

---

## NodeGroup Rolling Update 동작

Week4에서 설정한 `max_unavailable = 1`이 여기서 동작합니다.

```
Node 2개 기준 업그레이드 과정:

① Node-A Cordon (새 Pod 스케줄 차단)
② Node-A Drain  (기존 Pod를 Node-B로 이동)
③ Node-A 교체  (새 AMI로 새 EC2 생성)
④ Node-A Ready 확인
⑤ Node-B 동일하게 반복

→ 서비스 중단 없음
```

---

## 실무 버전 관리 패턴

**tfvars로 환경별 버전 분리**

```
# dev.tfvars
eks_cluster_version = "1.30"   ← dev에서 먼저 테스트

# prod.tfvars
eks_cluster_version = "1.29"   ← 검증 후 올림
```

dev에서 새 버전을 먼저 올려서 문제 없으면 prod에 적용하는 패턴입니다.

**버전 고정 이유**

```hcl
# 이렇게 하면 안 됨
version = "latest"   # latest는 EKS에서 지원 안 함

# 이렇게 해야 함
version = "1.29"     # 명시적으로 고정
```

버전을 고정하지 않으면 `terraform apply` 시점에 따라 다른 버전이 설치될 수 있고, 팀원 간 환경이 달라집니다.

---

## 버전 확인 명령어

```bash
# 현재 클러스터 버전 확인
aws eks describe-cluster --name url-shortener \
  --query "cluster.version" --output text

# Node 버전 확인
kubectl get nodes
# NAME              STATUS   VERSION
# ip-10-0-10-x...  Ready    v1.29.3

# 사용 가능한 EKS 버전 목록
aws eks describe-addon-versions \
  --query "addons[0].addonVersions[].compatibilities[].clusterVersion" \
  --output text | tr '\t' '\n' | sort -u
```

---

## 핵심 요약

```
버전 관리 원칙:
1. terraform.tfvars에 버전 명시적 고정
2. dev → prod 순서로 검증 후 업그레이드
3. 마이너 버전은 한 단계씩만 올리기
4. Control Plane 먼저, NodeGroup 나중
5. 지원 종료 전 (14개월) 미리 업그레이드
```
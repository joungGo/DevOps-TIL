# 프로젝트 작업 기록

---

## 1. IAM 계정 생성

- 사용자 이름: admin-user-pjh
- 사용자 그룹 이름: admin-group
    - 연결된 정책: AdministratorAccess, IAMUserChangePassword

---

## 2. AWS CLI 설치 및 설정

aws configure 명령을 통해 CLI 설정 완료.

---

## 3. Terraform 폴더 및 파일 구조 설계

```
Url-Shortener-EKS-Platform/
├── url-shortener/
└── terraform/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    ├── providers.tf
    ├── terraform.tfvars
    └── iam_oidc.tf
```

---

## 4. Terraform 코드 작성

main.tf는 수동 작성 후 모듈 작성으로 전환하는 방식으로 진행.

### VPC

```terraform
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${var.project_name}-vpc"
    "kubernetes.io/cluster/${var.project_name}" = "shared"
  }
}
```

`enable_dns_support = false`로 설정하면 VPC의 DNS resolver가 비활성화되어 CoreDNS가 사용할 upstream DNS가 사라진다. CoreDNS 자체가 정상 동작하기 위해 upstream DNS에 의존하므로 DNS 체인이 깨지게 되며, 결과적으로 외부 도메인 조회뿐 아니라 Kubernetes Service Discovery도 실패하여 Pod 간 내부 통신까지 영향을 받는다.

---

### Public Subnet

```terraform
resource "aws_subnet" "public" {
  count                   = length(var.availability_zones)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.project_name}-public-${var.availability_zones[count.index]}"
    "kubernetes.io/cluster/${var.project_name}" = "shared"
    "kubernetes.io/role/elb" = "1"
  }
}
```

**주요 참고 사항**

- `cidrsubnet`의 `netnum`이 `2^newbits` 이상이면 에러가 발생하므로 AZ 개수와 `newbits`의 관계를 사전에 확인해야 한다.
- `count` 방식은 인덱스(숫자)로 리소스를 구분하며 순서가 중요한 Subnet, AZ 배열에 적합하다. `for_each` 방식은 키(문자열)로 구분하며 IAM 정책처럼 이름이 중요한 리소스에 적합하다.
- AWS는 각 서브넷에서 5개의 IP를 자동 예약한다.

|IP|용도|
|---|---|
|.0|네트워크 주소|
|.1|VPC router|
|.2|AWS DNS|
|.3|AWS reserved|
|.255|broadcast|

---

### Private Subnet

```terraform
resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "${var.project_name}-private-${var.availability_zones[count.index]}"
    "kubernetes.io/cluster/${var.project_name}" = "owned"
    "kubernetes.io/role/internal-elb" = "1"
  }
}
```

`count.index + 10`을 사용하는 이유는 Public Subnet과 CIDR이 겹치지 않도록 하기 위해서다. 일반적인 서브넷 인덱스 대역 관례는 아래와 같다.

|용도|CIDR 범위|
|---|---|
|public|0 ~ 9|
|private|10 ~ 19|
|db|20 ~ 29|

Private Subnet에 `map_public_ip_on_launch`를 설정하지 않는 이유는 보안 원칙상 Private Subnet은 외부에서 직접 접근할 수 없어야 하기 때문이다. 해당 옵션을 활성화하면 Public IP가 할당되어 외부 접근이 가능해지며, Kubernetes의 경우 외부에서 Worker Node에 직접 접근할 수 있게 되어 보안 문제가 발생한다.

---

### NAT Gateway

```terraform
resource "aws_eip" "nat" {
  domain = "vpc"

  tags = {
    Name = "${var.project_name}-nat-eip"
  }

  depends_on = [aws_internet_gateway.main]
}
```

IGW가 EIP 및 NAT Gateway보다 먼저 생성되어야 하는 이유는, NAT Gateway가 배치되는 Public Subnet이 인터넷으로 나가기 위해 IGW가 먼저 연결되어 있어야 하기 때문이다.

AZ당 NAT Gateway를 1개씩 배치하는 것이 권장되는 이유는 특정 AZ의 NAT Gateway가 단일 장애점(SPOF)이 되지 않도록 가용성을 확보하기 위해서다.

---

### EKS Managed NodeGroup

```terraform
resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "${var.project_name}-nodegroup"
  node_role_arn   = aws_iam_role.eks_node.arn
  subnet_ids      = aws_subnet.private[*].id
  instance_types  = [var.node_instance_type]

  scaling_config {
    desired_size = var.node_desired_size
    min_size     = var.node_min_size
    max_size     = var.node_max_size
  }

  update_config {
    max_unavailable = 1
  }

  tags = {
    Name = "${var.project_name}-node"
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_worker_node,
    aws_iam_role_policy_attachment.eks_cni,
    aws_iam_role_policy_attachment.ecr_read,
  ]
}
```

Spot 인스턴스는 비용 절감 효과가 있으나, AWS가 자원을 회수할 경우 2분 전 알림과 함께 강제 종료될 수 있다. 주로 Stateless 애플리케이션 Pod에 적용하며, Cold Start 문제와 Spot 종료 시 On-Demand 노드로 Pod가 급격히 몰리는 상황에 대한 예방책이 필요하다. 이번 단계(Week 4)에서는 Spot을 적용하지 않았다.

**terraform plan 점검 항목**

|점검 항목|확인 위치|정상 기준|문제 시 영향|중요도|
|---|---|---|---|---|
|노드 개수|desired_size|2 이상|Pod 스케줄링 실패|높음|
|최대 노드|max_size|desired보다 큼|AutoScaling 불가|중|
|최소 노드|min_size|1 이상|전체 장애 가능|높음|
|인스턴스 타입|instance_types|여러 개 (Spot 시)|Spot 실패율 증가|중|
|Spot 여부|capacity_type|SPOT or ON_DEMAND|비용 증가|중|
|AZ 개수|subnet AZ|2개 이상|단일 AZ 장애|높음|
|Private subnet|subnet type|private 사용|보안 위험|높음|
|NAT Gateway 수|nat gateway|AZ 수만큼|인터넷 장애|중|
|VPC CIDR|vpc cidr|/16 권장|IP 부족|중|
|서브넷 CIDR|subnet cidr|겹치지 않음|네트워크 오류|높음|
|EKS 버전|version|최신 안정 버전|지원 종료 위험|낮음|
|노드 IAM Role|node_role|생성됨|노드 join 실패|높음|
|Cluster endpoint|endpoint|private 권장|보안 문제|낮음|
|kubeconfig output|output|존재|kubectl 접속 불가|낮음|

**자체 체크리스트**

|항목|현재 상태|평가|
|---|---|---|
|AZ|2개|양호|
|Node|2개|양호|
|Instance|t3.medium 1개|개선 가능|
|Spot|미사용|비용 증가|
|NAT|1개|SPOF 위험|
|Private subnet|사용|양호|

---

## 5. terraform apply 이후 작업

### kubeconfig 업데이트

로컬에서 minikube를 사용할 때는 kubeconfig가 자동으로 설정되지만, EKS는 클라우드에 위치하므로 연결 정보를 직접 설정해야 한다. kubeconfig는 kubectl이 어느 클러스터에 어떻게 접속할지를 담은 설정 파일이다.

```bash
aws eks update-kubeconfig \
  --region ap-northeast-2 \
  --name url-shortener
```

현재 연결된 클러스터 확인:

```bash
kubectl config current-context
```

---

### IRSA 설정

Pod가 AWS 서비스를 호출하려면 IAM 권한이 필요하다. OIDC issuer 확인 명령어는 아래와 같다.

```bash
aws eks describe-cluster --name url-shortener \
  --query "cluster.identity.oidc.issuer" --output text
```
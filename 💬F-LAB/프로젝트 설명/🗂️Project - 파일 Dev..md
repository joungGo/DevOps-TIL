### Kubernetes - Minikube

- 이미지 이름: `url-shortener-image:v1`

---
### Helm

- Deployment(API): `url-shortener-api`
- Service(API): `url-shortener-svc`
- Ingress(ingress.enabled=true일 때): `url-shortener-ingress`
- StatefulSet(Postgres): `url-shortener-postgres`
- Service(Postgres): `url-shortener-postgres-svc`
- PVC(Postgres): `url-shortener-postgres-pvc`
- Secret: `url-shortener-secret`
- Postgres Pod 이름(StatefulSet): `url-shortener-postgres-0` (replicas=1 기준)

---
### Terraform

#### 1. 파일 구성

|파일명|역할|주요 내용|
|---|---|---|
|`providers.tf`|Provider 설정|AWS ~> 5.0, Terraform >= 1.5.0|
|`variables.tf`|변수 선언|project_name, vpc_cidr, eks_cluster_version 등|
|`main.tf`|핵심 인프라|VPC, Subnet, IGW, NAT, EKS, NodeGroup, IAM|
|`iam_oidc.tf`|OIDC Provider (미구현)|템플릿 상태|
|`outputs.tf`|출력값|vpc_id, eks_cluster_name, endpoint 등|

---

#### 2. 리소스별 태그 현황

|리소스 타입|Terraform 리소스명|Name 태그 (실제 값)|추가 태그|
|---|---|---|---|
|`aws_vpc`|`aws_vpc.main`|`url-shortener-vpc`|`kubernetes.io/cluster/url-shortener = "shared"`|
|`aws_subnet` (public × 2)|`aws_subnet.public[0~1]`|`url-shortener-public-ap-northeast-2a/2c`|`k8s cluster = "shared"`, `kubernetes.io/role/elb = "1"`|
|`aws_subnet` (private × 2)|`aws_subnet.private[0~1]`|`url-shortener-private-ap-northeast-2a/2c`|`kubernetes.io/cluster/url-shortener = "owned"`|
|`aws_internet_gateway`|`aws_internet_gateway.main`|`url-shortener-igw`|-|
|`aws_eip`|`aws_eip.nat`|`url-shortener-nat-eip`|-|
|`aws_nat_gateway`|`aws_nat_gateway.main`|`url-shortener-nat`|-|
|`aws_route_table` (public)|`aws_route_table.public`|`url-shortener-public-rt`|-|
|`aws_route_table` (private)|`aws_route_table.private`|`url-shortener-private-rt`|-|
|`aws_route_table_association` × 4|`public/private[0~1]`|❌ 태그 없음|-|
|`aws_eks_cluster`|`aws_eks_cluster.main`|`url-shortener-eks`|-|
|`aws_eks_node_group`|`aws_eks_node_group.main`|`url-shortener-node`|-|
|`aws_iam_role` (cluster)|`aws_iam_role.eks_cluster`|❌ 태그 없음|-|
|`aws_iam_role` (node)|`aws_iam_role.eks_node`|❌ 태그 없음|-|
|`aws_iam_role_policy_attachment` × 4|`eks_cluster_policy` 외 3개|❌ 태그 없음|-|

---

#### 3. 태그 네이밍 패턴

|패턴|형식|의미|
|---|---|---|
|기본 Name|`${var.project_name}-{리소스타입}`|-|
|Subnet Name|`${var.project_name}-{public/private}-{az}`|AZ 구분|
|`kubernetes.io/cluster/... = "shared"`|VPC, Public Subnet|여러 클러스터 공유 가능|
|`kubernetes.io/cluster/... = "owned"`|Private Subnet|이 클러스터 전용|
|`kubernetes.io/role/elb = "1"`|Public Subnet|외부 LB 서브넷 지정|

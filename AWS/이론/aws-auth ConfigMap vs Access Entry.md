### 배경: EKS 인증 흐름

```
kubectl 명령어
    ↓
AWS IAM 인증 (내가 누구인지)
    ↓
EKS 인증 (클러스터 내에서 어떤 권한인지)
    ↓
Kubernetes RBAC
```

IAM 인증은 AWS가 처리하고, **EKS 인증은 "이 IAM User/Role이 클러스터에서 어떤 권한을 갖는지"를 매핑하는 단계**입니다. 이 매핑을 관리하는 방식이 aws-auth ConfigMap과 Access Entry입니다.

---

### aws-auth ConfigMap (구방식)

EKS가 클러스터 내부 `kube-system` 네임스페이스에 저장하는 YAML 파일입니다.

```yaml
# kubectl edit configmap aws-auth -n kube-system
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  # IAM Role → Kubernetes 사용자/그룹 매핑
  mapRoles: |
    - rolearn: arn:aws:iam::123456789:role/nodegroup-role
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes

    - rolearn: arn:aws:iam::123456789:role/admin-role
      username: admin
      groups:
        - system:masters

  # IAM User 직접 매핑
  mapUsers: |
    - userarn: arn:aws:iam::123456789:user/jounghyeon
      username: jounghyeon
      groups:
        - system:masters
```

**문제점**

```bash
# 클러스터 관리자가 직접 편집해야 함
kubectl edit configmap aws-auth -n kube-system

# 실수로 들여쓰기 하나 틀리면
mapRoles: |
  - rolearn: arn:...
   username: admin   # ← 들여쓰기 오류
    groups:          # ← YAML 파싱 실패

# 결과: 모든 사람이 클러스터 접근 불가 (자기 자신 포함)
```

---

### Access Entry (신방식, EKS 모듈 v21)

AWS API로 관리하는 방식입니다. ConfigMap 대신 EKS가 직접 IAM ↔ Kubernetes 권한을 관리합니다.

```hcl
# terraform/main.tf
access_entries = {
  admin = {
    # 어떤 IAM 주체에게 권한을 줄 것인가
    principal_arn = "arn:aws:iam::123456789:user/jounghyeon"

    policy_associations = {
      admin = {
        # AWS가 미리 만들어둔 관리형 정책
        policy_arn   = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"
        access_scope = { type = "cluster" }  # 클러스터 전체
      }
    }
  }
}
```

```bash
# AWS CLI로도 확인/관리 가능
aws eks list-access-entries --cluster-name urlshortener
aws eks list-access-policies
```

**AWS가 제공하는 관리형 정책 목록**

|정책|권한|
|---|---|
|`AmazonEKSClusterAdminPolicy`|클러스터 전체 admin|
|`AmazonEKSAdminPolicy`|대부분의 리소스 관리|
|`AmazonEKSEditPolicy`|리소스 수정 가능|
|`AmazonEKSViewPolicy`|읽기 전용|

---

### 핵심 차이 비교

||aws-auth ConfigMap|Access Entry|
|---|---|---|
|저장 위치|클러스터 내부 (kube-system)|AWS API|
|관리 방법|kubectl로 YAML 직접 편집|Terraform / AWS CLI / Console|
|실수 위험|높음 (YAML 파싱 오류 → 접근 불가)|낮음|
|노드 role 등록|수동으로 mapRoles에 추가|EKS가 자동 처리|
|Terraform 지원|별도 리소스 필요|`access_entries` 블록으로 선언|

---

### 현재 코드에서의 역할

```
authentication_mode = "API_AND_CONFIG_MAP"
```

두 방식을 동시에 사용합니다.

```
IAM User (jounghyeon)
    → Access Entry로 등록 → kubectl 사용 가능

NodeGroup EC2 노드
    → EKS 모듈이 자동으로 EC2_LINUX 타입 Access Entry 등록
    → aws-auth ConfigMap에도 자동 반영 (API_AND_CONFIG_MAP이므로)
```

`API`만 사용하면 aws-auth ConfigMap을 완전히 무시하고 Access Entry만 사용하는데, 일부 구버전 도구가 ConfigMap에 의존하는 경우가 있어 `API_AND_CONFIG_MAP`으로 두는 게 안전합니다.
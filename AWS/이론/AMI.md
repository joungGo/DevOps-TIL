## 한 줄 요약

```
AMI = EC2를 찍어내는 OS + 소프트웨어 템플릿
EKS에서는 K8s 버전이 포함된 AMI로 Worker Node를 만들고
버전 업그레이드 시 새 AMI로 Node를 교체함
```

---
## AMI (Amazon Machine Image)

**EC2 인스턴스를 찍어내는 템플릿**입니다.

---

## 쉬운 비유

```
AMI = 붕어빵 틀

틀(AMI) 하나로 붕어빵(EC2)을 여러 개 찍어낼 수 있음
틀이 바뀌면(AMI 업데이트) 새로 찍어낸 붕어빵도 달라짐
```

---

## AMI에 담긴 것들

```
AMI
├── OS (Amazon Linux 2, Ubuntu 22.04 등)
├── 기본 설치 소프트웨어
├── 스토리지 설정 (루트 볼륨 크기 등)
└── 부팅 설정
```

EKS Worker Node용 AMI에는 추가로 이것들이 포함됩니다.

```
EKS 최적화 AMI
├── OS (Amazon Linux 2)
├── kubelet (Node → API Server 통신)
├── kube-proxy (iptables 관리)
├── containerd (컨테이너 런타임)
└── aws-node (VPC CNI)
```

---

## EKS에서 AMI가 중요한 이유

AMI 버전 = Kubernetes 버전이 연결되어 있습니다.

```
EKS 1.29 클러스터
  → Worker Node도 K8s 1.29용 AMI 사용해야 함
  → AMI 버전이 맞지 않으면 Node가 클러스터에 못 붙음
```

버전 업그레이드 시 Worker Node를 새 AMI로 교체하는 게 Rolling Update입니다.

```
업그레이드 흐름:
  K8s 1.29 AMI Node → (Rolling Update) → K8s 1.30 AMI Node

max_unavailable=1 설정으로
한 번에 1개 Node씩 새 AMI로 교체 → 무중단
```

---

## Terraform에서의 AMI

Week4 코드에서 NodeGroup에 AMI를 명시하지 않으면 AWS가 Control Plane 버전에 맞는 최신 EKS 최적화 AMI를 자동 선택합니다.

```hcl
resource "aws_eks_node_group" "main" {
  # ami_type 명시 안 하면 자동 선택
  # AL2_x86_64 = Amazon Linux 2 (기본값)
  ami_type = "AL2_x86_64"
}
```


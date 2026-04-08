## 노드 그룹(Node Group)이란

**Worker Node(EC2)들을 묶어서 관리하는 단위**입니다.

```
EKS 클러스터 (url-shortener)
  └── 노드 그룹 (url-shortener-nodegroup)
        ├── Node 1 (EC2) - ap-northeast-2a
        └── Node 2 (EC2) - ap-northeast-2c
```

---

## 역할

**① EC2 수명 관리**

```
노드 그룹에 desired_size = 2 설정
  → EC2 2대 자동 생성
  → 1대가 죽으면 자동으로 새 EC2 생성해서 2대 유지
```

**② 스케일링**

```
min_size = 1, max_size = 4, desired_size = 2

트래픽 증가 시 → EC2 최대 4대까지 자동 증가
트래픽 감소 시 → EC2 최소 1대까지 자동 감소
```

**③ 업데이트**

```
update_config = { max_unavailable = 1 }

노드 업데이트 시 1대씩 순서대로 교체
→ 나머지 노드가 트래픽 처리하므로 서비스 무중단
```

---

## 노드 그룹이 여러 개인 경우

용도별로 노드 그룹을 분리할 수 있습니다:

```hcl
eks_managed_node_groups = {
  # 일반 앱용 노드 그룹
  general = {
    instance_types = ["t3.medium"]
    desired_size   = 2
  }

  # ML 연산용 노드 그룹 (GPU)
  gpu = {
    instance_types = ["g4dn.xlarge"]
    desired_size   = 1
  }
}
```

```
일반 Pod  → general 노드 그룹 EC2에 배치
ML Pod    → gpu 노드 그룹 EC2에 배치
```

현재 프로젝트는 노드 그룹이 1개(`main`)이므로 모든 Pod가 이 그룹의 EC2에 배치됩니다.

---
## 노드그룹이 2개 이상이어야 하는 경우

![[Pasted image 20260408121042.png]]

### 1. 워크로드 성격이 다를 때

```
일반 앱 노드 그룹 (t3.medium, On-demand)
  └── 웹 서버, API 서버 등 일반 Pod

GPU 노드 그룹 (g4dn.xlarge)
  └── AI 모델 추론, 이미지 처리 Pod
```

GPU 인스턴스는 비싸기 때문에 GPU가 필요한 Pod만 골라서 올리고 싶을 때 분리합니다.

---

### 2. 비용 최적화 (On-demand + Spot 혼합)

```
On-demand 노드 그룹 (항상 실행, 안정적)
  └── DB, 결제 서비스 등 중단되면 안 되는 Pod

Spot 노드 그룹 (최대 70% 저렴, 중단 가능)
  └── 배치 작업, 로그 처리 등 중단돼도 괜찮은 Pod
```

---

### 3. 환경 분리

```
시스템 노드 그룹
  └── ArgoCD, Karpenter, ALB Controller 등 인프라 Pod

앱 노드 그룹
  └── url-shortener 등 실제 서비스 Pod
```

인프라 Pod와 앱 Pod를 분리해서 앱 트래픽이 인프라에 영향을 못 주게 합니다.

---

### 4. 리소스 크기가 다를 때

```
소형 노드 그룹 (t3.small)
  └── 트래픽 적은 내부 서비스

대형 노드 그룹 (c5.2xlarge)
  └── CPU 집약적인 연산 서비스
```

---

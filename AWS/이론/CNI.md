**CNI**는 Kubernetes에서 **Pod 네트워크를 연결해주는 플러그인 인터페이스**야.  
쉽게 말하면 👉 **Pod가 IP를 받고 서로 통신하게 해주는 네트워크 구성 요소** 🌐

---

# 정의

Container Network Interface  
→ 컨테이너 네트워크를 설정하기 위한 표준 인터페이스

---

# Kubernetes에서 역할

Pod 생성 시 Kubernetes는 CNI에게 요청함:

```text
Pod 생성
   ↓
CNI 호출
   ↓
IP 할당
   ↓
네트워크 연결
```

즉:

- Pod IP 할당
    
- Pod ↔ Pod 통신
    
- Pod ↔ Service 통신
    
- Pod ↔ 외부 통신
    

---

# EKS에서 사용하는 CNI

Amazon VPC CNI

특징:

- Pod에 **VPC IP 직접 할당**
    
- NAT 없이 Pod 간 통신
    
- AWS 네트워크와 통합
    

---

# CNI가 없으면?

```text
Node 생성됨
Pod 생성됨
IP 없음
통신 불가
```

👉 Kubernetes 동작 불가

---

# 실제 Pod 확인 시 보이는 것

```bash
kubectl get pods -n kube-system
```

출력 예:

```text
aws-node-xxxxx   Running
```

여기서 `aws-node`가 바로 CNI

---

# 네 EKS 구조에서

```text
EC2 Node
   ├─ aws-node (CNI)
   ├─ kube-proxy
   └─ Pod
```

👉 `aws-node` = 네트워크 담당

---

# 한 줄 정리

👉 **CNI = Pod에 IP를 할당하고 네트워크 연결해주는 Kubernetes 플러그인**

---
---

# AWS VPC CNI 존재 여부 아키텍처 차이

![[Pasted image 20260330142203.png]]

**VPC CNI** — Pod가 VPC IP를 직접 가지기 때문에 VPC Router가 Pod를 EC2처럼 직접 라우팅합니다. 패킷이 그대로 전달됩니다.

**Overlay** — Pod IP가 VPC가 모르는 별도 대역(192.168.x.x)이라서, VPC를 통과하려면 "겉에 Node IP로 포장(캡슐화)"해서 보내고 목적지에서 다시 풀어야 합니다. VPC Router는 바깥 포장지(Node IP)만 보고 라우팅합니다.

>여기서 말하는 Node Ip의 노드는 인스턴스를 의미. 그렇다고 pod ip -> Service(cluster ip) 변환 과정이 없는 건 아님. 그냥 설명 상 중요하지 않은 내용이라 생략한 것임

Week4 구조에서 특히 중요한 건 마지막 행인 **t3.medium 최대 Pod 수 17개 제한**입니다. VPC CNI는 ENI 개수 × ENI당 IP 수로 Pod 수가 제한되는데, t3.medium은 ENI 3개 × IP 6개 - 1(Node 자신) = 최대 17개입니다. Overlay는 이 제한이 없지만 EKS에서는 VPC CNI가 기본값이라 이 제한을 인지하고 있어야 합니다.


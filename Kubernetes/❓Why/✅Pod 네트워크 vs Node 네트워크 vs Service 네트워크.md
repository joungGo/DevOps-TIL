> [!note] 면접 답변
> 쿠버네티스 네트워크는 **Pod 간 직접 통신을 위한 Pod 네트워크, Node 간 패킷 전달을 위한 Node 네트워크, 그리고 서비스 접근을 위한 가상 IP 공간인 Service 네트워크로 구성됩니다.**

---
# Pod 네트워크 vs Node 네트워크 vs Service 네트워크

이 3개는 **쿠버네티스에서 서로 다른 목적의 IP 대역**이다.

예시

```text
Pod network      10.244.0.0/16
Service network  10.96.0.0/12
Node network     192.168.0.0/24
```

각각의 역할을 설명하면 다음과 같다.

---

# 1) Pod 네트워크

목적

**Pod끼리 직접 통신하기 위한 네트워크**

특징

- 모든 Pod는 **고유 IP**
    
- **클러스터 전체에서 라우팅 가능**
    
- CNI가 관리
    

예

```text
Pod A 10.244.1.5
Pod B 10.244.2.7
```

Pod A → Pod B

```text
10.244.1.5 → 10.244.2.7
```

즉

```text
Pod ↔ Pod 통신
```

---

# 2) Node 네트워크

목적

**클러스터의 실제 서버(Node)들이 연결된 물리 네트워크**

예

```text
Node1 192.168.0.10
Node2 192.168.0.11
```

이 네트워크는

- 클라우드 VPC
    
- 회사 IDC 네트워크
    
- 로컬 네트워크
    

같은 **실제 네트워크 인프라**다.

Pod 간 통신 시 실제 흐름

```text
Pod A
 ↓
Node1 (192.168.0.10)
 ↓
Node2 (192.168.0.11)
 ↓
Pod B
```

즉 Node 네트워크는

**Node 간 패킷 전달 경로**

---

# 3) Service 네트워크

Service 네트워크는

**Service의 가상 IP(ClusterIP)를 위한 네트워크 대역**이다.

예

```text
payment-service
ClusterIP 10.96.0.15
```

하지만 중요한 사실

**이 IP는 실제 네트워크 인터페이스가 아니다.**

즉

```text
10.96.0.15
```

는 실제 서버가 아니다.

이 IP는

```text
iptables / ipvs
```

규칙으로 Pod로 변환된다.

예

```text
10.96.0.15
 ↓
Pod1 10.244.1.5
Pod2 10.244.2.8
```

즉 Service 네트워크는

**가상의 서비스 주소 공간**

---

# 3개 네트워크를 한 번에 정리

```text
Pod network
10.244.x.x
Pod 실제 IP

Service network
10.96.x.x
Service 가상 IP

Node network
192.168.x.x
실제 서버 네트워크
```

전체 흐름

```text
Pod
 ↓
Service IP (10.96.x.x)
 ↓
iptables (kube-proxy)
 ↓
Pod IP (10.244.x.x)
 ↓
Node network
 ↓
다른 Node
 ↓
Pod
```

---

# 한 줄 핵심 정리

|네트워크|목적|
|---|---|
|Pod 네트워크|Pod 간 직접 통신|
|Node 네트워크|Node 간 실제 패킷 전달|
|Service 네트워크|Service의 가상 IP 공간|

---

쿠버네티스를 공부하면 여기서 **정말 중요한 질문이 하나 더 나온다.**

많은 사람들이 처음에 헷갈리는 부분이다.

**"Pod는 외부 인터넷과 어떻게 통신할까?"**

즉

```text
Pod (10.244.x.x)
→
Google.com
```

이 과정에서 **SNAT / kube-proxy / CNI**가 어떻게 동작하는지 이해하면  
쿠버네티스 네트워크가 거의 완성된다.
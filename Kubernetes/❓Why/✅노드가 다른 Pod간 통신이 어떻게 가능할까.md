> [!note] 면접 답변
> Pod는 veth pair로 Node 네트워크에 연결되고, CNI가 설정한 Node 라우팅을 통해 다른 Node의 Pod 네트워크로 패킷이 전달되기 때문에 Node가 달라도 Pod 간 통신이 가능하다.

---

### Pod IP는 Node가 다른데 어떻게 서로 통신할까

쿠버네티스에는 중요한 네트워크 규칙이 있다.

**Pod 네트워크 모델**

1. 모든 Pod는 **고유한 IP**를 가진다
    
2. **Pod끼리는 NAT 없이 서로 통신 가능해야 한다**
    
3. **Node가 달라도 Pod IP로 직접 통신 가능해야 한다**
    

예

```text
Node1
  Pod A → 10.244.1.5

Node2
  Pod B → 10.244.2.7
```

이때 Pod A는 **10.244.2.7**로 바로 통신할 수 있어야 한다.

---

# 1. 이 문제를 해결하는 것: CNI

이 역할을 하는 것이 **CNI(Container Network Interface)** 이다.

CNI는 **쿠버네티스 Pod 네트워크를 구성하는 플러그인 시스템**이다.

즉

```text
Pod 네트워크 구성 → CNI
```

대표적인 CNI

|CNI|특징|
|---|---|
|Flannel|가장 단순|
|Calico|네트워크 정책 지원|
|Weave|자동 네트워크 구성|
|Cilium|eBPF 기반|

---

# 2. Pod IP는 누가 할당할까

Pod가 생성될 때

```text
kubelet → CNI plugin 호출
```

CNI가 다음 작업을 수행한다.

1. Pod에 IP 할당
    
2. 가상 네트워크 인터페이스 생성
    
3. Node 네트워크와 연결
    

예

```text
Pod
eth0 → 10.244.1.5
```

---

# 3. Pod 네트워크 구조

Pod는 **자기만의 네트워크 namespace**를 가진다.

구조

```text
Pod Network Namespace
   │
   │ veth pair
   │
Node Network
```

veth pair는 **가상의 네트워크 케이블**이다.

```text
Pod side veth  ↔  Node side veth
```

예

```text
Pod
  eth0
   │
   │
veth1234
   │
   │
Node bridge
```

---

# 4. Node가 다른 Pod끼리 통신

이제 중요한 부분이다.

예

```text
Node1
  Pod A → 10.244.1.5

Node2
  Pod B → 10.244.2.7
```

Pod A가 Pod B로 요청

```text
10.244.1.5 → 10.244.2.7
```

Node1 라우팅 테이블

```text
10.244.2.0/24 → Node2
```

즉

```text
Pod A
 ↓
Node1
 ↓
Node2
 ↓
Pod B
```

---

# 5. 전체 네트워크 흐름

```text
Pod A
 ↓
veth
 ↓
Node1 network
 ↓
Node routing
 ↓
Node2 network
 ↓
veth
 ↓
Pod B
```

즉 Pod IP는 **클러스터 전체에서 라우팅 가능한 네트워크**다.

---

# 6. Flannel 예시 (가장 쉬운 구조)

Flannel은 **Overlay Network**를 사용한다.

예

```text
Node1 subnet
10.244.1.0/24

Node2 subnet
10.244.2.0/24
```

Node1 라우팅

```text
10.244.2.0/24 → Node2
```

패킷 전달

```text
Pod A
 ↓
Node1
 ↓
VXLAN 터널
 ↓
Node2
 ↓
Pod B
```

---

# 7. 지금까지 전체 흐름 정리

쿠버네티스 내부 요청 흐름

```text
Pod
 ↓
DNS (CoreDNS)
 ↓
Service name
 ↓
ClusterIP
 ↓
kube-proxy
 ↓
Pod 선택
 ↓
CNI 네트워크
 ↓
Pod
```

구성 요소 역할

|구성요소|역할|
|---|---|
|CoreDNS|서비스 이름 → IP|
|Service|안정적인 접근 주소|
|kube-proxy|트래픽 로드밸런싱|
|CNI|Pod 네트워크 구성|

---

# 한 줄 정리

Pod가 Node가 달라도 통신할 수 있는 이유는 **CNI 플러그인이 Pod 네트워크를 구성하고 Node 간 라우팅을 설정하기 때문**이다.

---

쿠버네티스를 공부할 때 여기서 **아주 중요한 개념 하나가 더 이어진다.**

"**[[✅Pod 네트워크 vs Node 네트워크 vs Service 네트워크]]**"


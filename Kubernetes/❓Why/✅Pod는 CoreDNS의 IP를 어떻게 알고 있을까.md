> [!note] 면접 답변
> Pod는 CoreDNS의 IP를 직접 아는 것이 아니라, Pod 생성 시 kubelet이 `/etc/resolv.conf`에 CoreDNS Service의 ClusterIP를 nameserver로 설정해주기 때문에 DNS 질의를 CoreDNS로 보내게 됩니다.

---
### Pod는 CoreDNS의 IP를 어떻게 알고 있을까

이것은 **kubelet이 Pod를 생성할 때 DNS 설정을 자동으로 넣어주기 때문**이다.

Pod가 생성되면 내부에 다음 파일이 만들어진다.

```text
/etc/resolv.conf
```

Pod 내부에서 확인 가능

```bash
cat /etc/resolv.conf
```

예시

```text
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

여기서 중요한 부분

```text
nameserver 10.96.0.10
```

이 IP가 **CoreDNS Service의 ClusterIP**다.

---

### 전체 흐름

1. CoreDNS가 Service로 실행됨
    

```text
Service name: kube-dns
ClusterIP: 10.96.0.10
```

2. kubelet이 Pod 생성
    
3. kubelet이 Pod 내부 DNS 설정 작성
    

```text
/etc/resolv.conf
```

4. Pod가 DNS 조회
    

```text
payment-service
```

5. 요청이 CoreDNS로 전달
    

전체 구조

```text
Pod
 ↓
/etc/resolv.conf
 ↓
CoreDNS Service (10.96.0.10)
 ↓
Service IP 반환
 ↓
Pod 통신
```

---

### 매우 중요한 흐름 (쿠버네티스 네트워크 전체)

쿠버네티스 내부 서비스 호출은 실제로 이렇게 진행된다.

```text
Pod
 ↓
DNS 조회 (CoreDNS)
 ↓
Service ClusterIP
 ↓
kube-proxy
 ↓
Pod
```

---

### 한 줄 정리

Pod는 **kubelet이 생성 시 자동으로 넣어주는 `/etc/resolv.conf` 설정을 통해 CoreDNS의 IP를 알게 된다.**

---

쿠버네티스를 공부하다 보면 여기서 대부분 사람들이 다음 질문을 한다.

**"[[✅Service는 실제로 어떻게 Pod로 트래픽을 보내는가]]?"**

# 3️⃣ forwarded_port → NAT → iptables 흐름 (핵심 네트워크)

이 부분은 **실무에서 가장 중요**함.

---

## 3-1️⃣ forwarded_port 한 줄의 정체

`cfg.vm.network "forwarded_port", guest: 80, host: 10080`

이 한 줄이 의미하는 것:

> “호스트의 10080 포트로 들어온 트래픽을  
> VM의 80번 포트로 전달해라”

---

## 3-2️⃣ 실제 네트워크 흐름

`[브라우저]    ↓ localhost:10080 [호스트 OS]    ↓ VirtualBox NAT [VM eth0]    ↓ port 80 [Nginx / Web Server]`

✔ VM의 private IP(192.168.x.x)는 **외부에서 접근 불가**  
✔ NAT 장치가 중간에서 변환

---

## 3-3️⃣ iptables 관점에서 보면

VirtualBox 내부에서 자동 생성되는 규칙 (개념적):

`PREROUTING  host:10080 → guest:80`

👉 Vagrant는:

- 직접 iptables를 만지지 않음
    
- **VirtualBox API를 통해 NAT 룰 생성**
    

---

## 3-4️⃣ 왜 외부에서 VM IP로 접속이 안 될까?

`외부 → 192.168.32.10 ❌ 외부 → localhost:10080 ⭕`

이유:

- `192.168.32.10` 은 **Host-only 네트워크**
    
- 외부 라우팅 불가
    
- 오직 **호스트만 접근 가능**
    

```md
외부(인터넷)에서 *http://192.168.32.10*으로 요청을 보냄 -> 포트 포워딩 작용X
외부(인터넷)에서 *http://localhost:10080*으로 요청을 보냄 -> 포트 포워딩 작용O

192.168.32.10으로 요청을 보낸 경우 VM으로 직접 접근을 하기 때문에 기능하지 않는다.
1. 외부(인터넷)에서 192.168.32.10으로 접근이 가능한 이유는 그냥 목적지 ip를 알기 때문이다.
2. 포트 포워딩은 호스트OS의 IP에서 VM의 사설IP로 변환해주는 NAT 레이어에 위치하기 때문에 호스트로의 요청인 localhost로 요청이 들어와야 포트 포워딩이 기능할 수 있다.
   
| 현재 위의 구조 상 `host-only 네트워크`와 `NAT(포트 포워딩) 네트워크` 2개가 존재한다.
```
[[host-only 네트워크 vs NAT(포트 포워딩 네트워크) 구조에 따른 포트 포워드 동작 이해]] 

---

## 3-5️⃣ forwarded_port vs private_network 역할 분리

| 항목              | 용도       |
| --------------- | -------- |
| private_network | VM ↔ 호스트 |
| forwarded_port  | 외부 ↔ VM  |
| NAT             | 인터넷 접근   |
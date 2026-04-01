## Private Subnet → 외부 네트워크 경로 & NAT Gateway 동작 원리

---

## 전체 경로

```
Pod (10.0.10.11)
    ↓
Node eth0 (10.0.10.5)
    ↓  [iptables — 경우에 따라 SNAT]
Private Route Table
    ↓  [0.0.0.0/0 → NAT GW]
NAT Gateway (10.0.0.x / EIP: 52.x.x.x)
    ↓  [SNAT: 내부 IP → EIP]
Internet Gateway
    ↓
인터넷 (Google, ECR 등)
```

---

## NAT Gateway 동작 원리 — SNAT

NAT Gateway의 핵심은 **SNAT(Source NAT)** 입니다. 출발지 IP를 바꿔치기합니다.

```
패킷 출발 시:
  src: 10.0.10.11 (Pod IP)   dst: 142.250.x.x (Google)
          ↓
  NAT GW가 src를 EIP로 변환
  src: 52.x.x.x (EIP)        dst: 142.250.x.x (Google)
          ↓
  Connection Table에 기록
  (10.0.10.11 : 포트1234) ↔ (52.x.x.x : 포트5678)
          ↓
  인터넷으로 전송
```

인터넷에서는 EIP만 보입니다. 내부 사설 IP는 완전히 숨겨집니다.

---

## 응답이 돌아오는 경로 — Connection Table 역조회

```
Google → 52.x.x.x (EIP) 로 응답 전송
          ↓
  NAT GW가 Connection Table 역조회
  (52.x.x.x : 포트5678) → (10.0.10.11 : 포트1234)
          ↓
  dst를 원래 내부 IP로 DNAT
  dst: 10.0.10.11 (Pod IP)
          ↓
  Node → Pod 전달
```

---

## VPC CNI 환경에서의 차이

SNAT이 몇 번 발생하는지가 다릅니다.

```
일반 CNI (Flannel 등):
  Pod IP → (Node에서 SNAT) → Node IP
         → (NAT GW에서 SNAT) → EIP
  SNAT 2번 발생

AWS VPC CNI:
  Pod IP = 실제 VPC IP (10.0.10.11)
         → (NAT GW에서 SNAT) → EIP
  SNAT 1번만 발생
  Node 단계 SNAT 생략
```

VPC CNI는 Pod가 VPC IP를 직접 보유하므로 NAT GW가 Pod IP를 바로 인식해서 SNAT합니다.

---

## Route Table이 핵심

NAT GW가 있어도 Route Table 설정이 없으면 트래픽이 NAT GW로 가지 않습니다.

```
Private Route Table 설정:
  Destination     Target
  10.0.0.0/16     local        ← VPC 내부는 직접
  0.0.0.0/0       nat-xxxxxx   ← 그 외 모두 NAT GW로
```

Public Route Table과의 차이가 여기서 납니다.

```
Public Route Table:
  0.0.0.0/0 → IGW   ← 인터넷 직접 연결

Private Route Table:
  0.0.0.0/0 → NAT GW ← NAT GW 경유 후 IGW
```

---

## NAT GW가 Public Subnet에 있어야 하는 이유

```
NAT GW 자신도 인터넷과 통신해야 함
  → 퍼블릭 IP(EIP) 필요
  → EIP 사용하려면 IGW 접근 필요
  → IGW 접근 = Public Subnet

Private Subnet에 NAT GW를 두면?
  → NAT GW 자체가 인터넷 못 나감
  → 의미 없음
```

---

## 실무 주의사항

```
NAT GW 1개 (이 프로젝트):
  AZ-a NAT GW 장애 → AZ-c Private Subnet도 인터넷 불가
  비용: $0.045/시간 ≈ $32/월

NAT GW AZ당 1개 (운영 권장):
  AZ-a NAT GW 장애 → AZ-c는 자체 NAT GW로 정상
  비용: $32/월 × AZ 수

VPC Endpoint 활용:
  ECR, S3, DynamoDB는 VPC Endpoint로 NAT GW 우회
  → 데이터 전송 비용 절감
  → 보안 강화 (인터넷 경유 안 함)
```

---

## 한 줄 요약

```
Private Subnet → Route Table(0.0.0.0/0 → NAT GW)
→ NAT GW에서 SNAT(내부 IP → EIP)
→ IGW → 인터넷
응답은 Connection Table 역조회로 원래 Pod에 전달
```
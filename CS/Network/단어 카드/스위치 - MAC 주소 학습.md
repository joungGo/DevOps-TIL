1. 플러딩
2. 필터링 & 포워딩
3. 에이징
## 예시 네트워크 구성

```
[PC A] ──(Port 1)──┐
                   │
[PC B] ──(Port 2)──┤  [ Switch ]
                   │
[PC C] ──(Port 3)──┘
```

|장비|MAC 주소|
|---|---|
|PC A|AA-AA|
|PC B|BB-BB|
|PC C|CC-CC|

---

## ARP 요청 (ARP Request – Broadcast)

### 상황

- PC A는 **PC B의 IP 주소만 알고 있고**
    
- **PC B의 MAC 주소(BB-BB)를 모르는 상태**
    
- 실제 데이터 전송 전에 **MAC 주소 확인 필요**
    

---

### 동작 과정

1. PC A는 ARP Request 생성
    
    - 출발지 MAC: **AA-AA**
        
    - 목적지 MAC: **FF:FF:FF:FF:FF:FF (Broadcast)**
        
2. Port 1에서 스위치가 프레임 수신
    
3. **출발지 MAC (AA-AA)** 을 테이블에 기록
    
4. 목적지가 브로드캐스트이므로  
    → **모든 포트로 Flooding**
    

---

### MAC 주소 테이블 상태 (ARP Request 후)

```markdown
| MAC 주소 | 포트 |
|---------|------|
| AA-AA   | 1    |
```

---

## ARP 응답 (ARP Reply – Unicast)

### 상황

- PC B가 ARP Request 수신
    
- 자신의 IP에 대한 요청임을 확인
    

---

### 동작 과정

1. PC B는 ARP Reply 전송
    
    - 출발지 MAC: **BB-BB**
        
    - 목적지 MAC: **AA-AA**
        
2. Port 2에서 스위치가 프레임 수신
    
3. **출발지 MAC (BB-BB)** 학습
    
4. 목적지 MAC (AA-AA) 존재
    
5. **Port 1으로만 전달**
    

---

### MAC 주소 테이블 상태 (ARP 완료 후)

```markdown
| MAC 주소 | 포트 |
|---------|------|
| AA-AA   | 1    |
| BB-BB   | 2    |
```

---

## 플러딩 (Flooding)

### 상황

- **PC A → PC B** 로 처음 실제 데이터 통신 시도
    
- (ARP 과정에서 이미 MAC 학습이 완료된 상태)
    

※ 만약 ARP 이전에 Unknown Unicast 프레임이 들어왔다면  
→ 이 단계에서 플러딩이 발생했을 것임

---

### 동작 과정

1. Port 1에서 프레임 수신
    
2. 출발지 MAC (AA-AA) 이미 존재
    
3. 목적지 MAC (BB-BB) 존재
    
4. **Flooding 없이 Port 2로 전달**
    

---

## 필터링 & 포워딩 (Filtering & Forwarding)

### 상황

- PC B가 PC A에게 데이터 응답
    

---

### 동작 과정

1. Port 2에서 프레임 수신
    
2. 출발지 MAC (BB-BB) 이미 학습됨
    
3. 목적지 MAC (AA-AA) 존재
    
4. **Port 1으로만 전달**
    
5. 다른 포트는 **필터링(차단)**
    

---

### MAC 주소 테이블 상태

```markdown
| MAC 주소 | 포트 |
|---------|------|
| AA-AA   | 1    |
| BB-BB   | 2    |
```

---

## 에이징 (Aging)

### 상황

- PC C는 한동안 통신하지 않음
    
- PC B도 장시간 트래픽 없음
    

---

### 동작 과정

- 스위치는 MAC 엔트리에 **타이머 유지**
    
- 기본값: **300초 (5분)**
    
- 해당 시간 동안 프레임 미수신 시 삭제
    

---

### 일정 시간 후 MAC 주소 테이블

```markdown
| MAC 주소 | 포트 |
|---------|------|
| AA-AA   | 1    |
```

> BB-BB 엔트리는 트래픽이 없어 제거됨

---

## 이후 재통신 시

- BB-BB가 다시 통신하면
    
    - **ARP 캐시가 없으면 ARP 재요청**
        
    - **스위치는 다시 MAC 학습**
        
- 네트워크 변화에 자동 적응
    

---

## 전체 흐름 한 줄 요약

> **ARP로 IP→MAC을 먼저 해결하고,  
> 스위치는 출발지 MAC으로 학습하며,  
> 목적지 MAC이 없으면 플러딩,  
> 있으면 필터링 & 포워딩을 수행하고,  
> 오래 쓰이지 않는 정보는 에이징으로 제거한다.**

---

## 시험·실무에서 자주 나오는 포인트

- ARP Request는 **Broadcast**
    
- ARP Reply는 **Unicast**
    
- MAC 학습은 **항상 출발지 MAC 기준**
    
- Flooding은 **Broadcast + Unknown Unicast**
    
- Aging 없으면 **MAC 테이블 오염**
    

---

이 버전은  
📌 **이론 설명 / 시험 답안 / 기술 블로그** 어디에 써도 바로 통하는 구조야.

다음으로 가면 흐름이 완전히 닫혀 👇

- ❓ **다른 네트워크일 때 왜 게이트웨이 MAC을 쓰는가**
    
- ❓ **VLAN 분리 시 ARP는 어디까지 퍼질까**
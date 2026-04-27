# 1. RDS (PostgreSQL) WAL 저장 예시

Amazon RDS (PostgreSQL 기준)

## 상황

```sql
UPDATE users SET name = 'kim' WHERE id = 1;
```

---

## WAL에 기록되는 형태 (개념 예시)

```text
LSN: 0/16B6C50
TXID: 1001
ACTION: UPDATE
TABLE: users
OLD:
  id=1, name='park'
NEW:
  id=1, name='kim'
```

또는 더 실제에 가까운 형태:

```text
LSN: 0/16B6C50
heap_update:
  rel=users
  block=42
  offset=3
  old_tuple=(id=1, name='park')
  new_tuple=(id=1, name='kim')
```

---

## 핵심 포인트

- **변경된 데이터 자체를 기록**
    
- row 단위 변경
    
- LSN(Log Sequence Number) 기반 순서 보장
    

---

## Replica 동작

```text
Primary:
  WAL 생성 → 전송

Replica:
  WAL 수신 → replay → 동일 데이터 생성
```

👉 WAL = **복제 데이터 그 자체**

---

# 2. Prometheus WAL 저장 예시

Prometheus

## 상황

```text
http_requests_total{method="GET", status="200"} 1024
```

---

## WAL 기록 (1) - 시계열 생성

```text
[Series]
series_id: 1
metric: http_requests_total
labels:
  method="GET"
  status="200"
```

---

## WAL 기록 (2) - 값 저장

```text
[Sample]
series_id: 1
timestamp: 1714200000
value: 1024
```

---

## 이후 scrape 계속 발생

```text
[Sample]
series_id: 1, ts=1714200015, value=1030

[Sample]
series_id: 1, ts=1714200030, value=1050
```

---

## 핵심 포인트

- 데이터 변경이 아니라
    
- **“시계열 이벤트” 기록**
    

즉:

- series 정의 (1번)
    
- 이후 값만 계속 append
    

---

# 3. 구조적 차이 핵심 비교

|항목|RDS WAL|Prometheus WAL|
|---|---|---|
|기록 대상|데이터 변경 (row)|시계열 데이터|
|목적|복제|장애 복구|
|단위|row/page|metric + label|
|식별 방식|LSN|series_id|
|데이터 의미|DB 상태 변경|관측 값 기록|
|사용 방식|replay → DB 재구성|replay → 메모리 복구|

---

# 4. 직관적인 비유

### RDS WAL

```text
"users 테이블 1번 row를 park → kim으로 바꿨다"
```

👉 상태 변경 기록

---

### Prometheus WAL

```text
"이 metric에서 이 시점에 값이 1024였다"
```

👉 관측 로그

---

# 5. Terraform / 아키텍처 관점 인사이트

## RDS

- WAL → 복제 기반 HA
    
- 별도 설계 없이도 안정성 확보
    

---

## Prometheus

- WAL → 단순 복구용
    
- HA 없음
    

👉 반드시 필요:

```text
- PVC
- remote_write (Thanos / Mimir / VictoriaMetrics)
```

---

# 최종 한 줄 정리

👉 RDS WAL = “데이터를 다시 만들기 위한 변경 로그”  
👉 Prometheus WAL = “메모리를 복구하기 위한 시계열 이벤트 로그”

---

원하면

- “PostgreSQL WAL 내부 구조 (segment, LSN 흐름)”
    
- “Prometheus WAL replay 시 느려지는 이유”
    
- “카디널리티 증가 시 WAL이 어떻게 커지는지”
    

이거 더 깊게 파줄게.
## RDS Primary → Replica 동기화 방식

AWS RDS Read Replica는 **비동기 복제(Asynchronous Replication)** 를 사용합니다.

---

### 동기화 흐름

```
클라이언트
  → Primary (쓰기)
      ↓ 1. WAL (Write-Ahead Log) 생성
      ↓ 2. Primary 디스크에 커밋 완료 → 클라이언트에 응답
      ↓ 3. WAL을 Replica로 스트리밍 전송
  → Replica
      ↓ 4. WAL 수신 후 동일한 변경사항 재실행(Replay)
      ↓ 5. Replica 데이터 반영 완료
```

---

### WAL이란?

```
WAL (Write-Ahead Log)
  = DB의 모든 변경사항을 순서대로 기록한 로그 파일

INSERT INTO items (name) VALUES ('hello');
  → 실제 데이터 파일 변경 전에 WAL에 먼저 기록
  → WAL이 있으면 장애가 나도 복구 가능
  → 이 WAL을 그대로 Replica에 보내서 동일하게 재실행
```

---

### 비동기이기 때문에 생기는 현상

```
Primary에 INSERT 완료
  → 클라이언트: "성공" 응답 수신
  → 바로 Replica 조회
  → 아직 반영 안 됨 (Replication Lag)
```

- **Replication Lag** = Primary 커밋 시각과 Replica 반영 시각의 차이
- 보통 수 ms ~ 수백 ms
- 네트워크 지연, Replica 부하, 대량 쓰기 시 수 초까지 벌어질 수 있음

---

### 이 프로젝트에서의 영향

```python
# item.py — POST /items (쓰기)
db.add(item)
db.commit()          # ← Primary에 커밋 완료, 클라이언트에 응답

# 바로 아래에서 Replica 조회하면?
items = read_db.query(Item).all()   # ← Replication Lag으로 방금 쓴 데이터가 없을 수 있음
```

이 때문에:

- **쓰기(POST/DELETE)** → Primary (`get_write_db`)
- **읽기(GET)** → Replica (`get_read_db`)
- 캐시 무효화 후 첫 조회에서 Lag이 있으면 → 이전 데이터가 잠깐 캐싱될 수 있음
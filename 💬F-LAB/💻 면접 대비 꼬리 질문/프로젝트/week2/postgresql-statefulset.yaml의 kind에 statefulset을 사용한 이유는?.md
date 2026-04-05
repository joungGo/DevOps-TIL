**면접용 한 줄 답변**  
StatefulSet은 PostgreSQL처럼 **Pod 이름과 스토리지가 고정되어야 하고 재시작 시 데이터가 유지되어야 하는 상태 기반(Stateful) 워크로드이기 때문에 Deployment 대신 사용했습니다.

---

## StatefulSet을 사용한 이유

PostgreSQL 요구사항:

- 고정된 Pod 이름 필요 (`postgres-0`)
    
- 재시작 시 동일 스토리지 연결 필요
    
- 순차적 시작/종료 필요
    
- 데이터 영속성 필요
    

이 특성을 StatefulSet이 보장합니다

> [!NOTE] StatefulSet을 사용하는 이유 (면접 답변용 정리) 📌  
> **StatefulSet은 고정된 네임, 안정적 PVC 바인딩, 순차적 라이프사이클을 제공하여 DB 같은 상태 기반 서비스의 가용성과 일관성을 보장한다.**
> 
> **1) 고정 이름이 필요한 이유 (Stable DNS) 🧭**
> 
> - StatefulSet은 `name-ordinal` 형식(`postgres-0`)의 고정 Pod 이름과 DNS(`postgres-0.<svc>`) 제공
>     
> - DB 복제(primary/replica), 클러스터 초기화 시 **정해진 호스트명 필요**
>     
> - 예: replica가 `postgres-0.postgres-svc`로 primary 연결 → 재시작 후에도 동일 대상 유지
>     
> - Deployment는 Pod 이름이 변경 → 연결 실패 가능
>     
> 
> **2) 재시작 시 동일 스토리지(PVC) 보장 문제 💾**
> 
> - Deployment
>     
>     - Pod 재생성 시 다른 Pod가 기존 PVC 마운트 시도 → `ReadWriteOnce` 충돌
>         
>     - 스케일 시 동일 PVC 공유 → 데이터 손상 위험
>         
> - StatefulSet
>     
>     - `volumeClaimTemplates`로 Pod별 고유 PVC 자동 생성 (`postgres-0-pvc`)
>         
>     - Pod 재생성 시 동일 PVC 재연결 → 데이터 연속성 보장
>         
> 
> **3) 순차적 시작/종료가 필요한 이유 🔄**
> 
> - DB 클러스터는 **primary → replica 순서 필요**
>     
> - Deployment: 병렬 생성/삭제 (순서 없음)
>     
> - StatefulSet: `0 → 1 → 2` 순차 생성, 삭제는 역순
>     
> - 결과: replica 초기 동기화 등 클러스터 구성 안정적 동작
>     
> 
> **4) volume만으로 부족한 이유 ⚠️**
> 
> - 단일 인스턴스: PVC 하나로 충분
>     
> - 운영환경 요구사항
>     
>     - Pod별 고유 스토리지 필요
>         
>     - Pod 식별자 + 스토리지 바인딩 일관성 필요
>         
>     - 스케일 시 자동 PVC 관리 필요
>         
> - StatefulSet은 **고정 식별자 + 1:1 PVC + 순차 lifecycle** 모두 제공
>

---

## Deployment와 비교

Deployment 특징

- Pod 이름 랜덤
    
- 스케일 시 새 Pod 생성
    
- 스토리지 고정 어려움
    
- stateless 앱용
    

문제 상황 (Deployment 사용 시)

```
postgres-6f7c9d8d7b-abcde
삭제 → 새 Pod 생성
postgres-6f7c9d8d7b-xyz12
```

→ DB 데이터 연결 깨짐 가능

StatefulSet 특징

```
postgres-0
삭제 → 다시 생성
postgres-0 (동일 이름 + 동일 PVC)
```

→ 데이터 유지

---

## DaemonSet과 비교

DaemonSet 특징

- 모든 노드마다 Pod 1개
    
- 로그 수집, 에이전트 용도
    
- DB 용도 아님
    

Postgres에 부적합 이유

- 여러 노드에 DB 분산 생성됨
    
- 데이터 일관성 깨짐
    

---

## Job / CronJob과 비교

Job 특징

- 한번 실행 후 종료
    
- 배치 처리용
    

DB는 계속 실행 필요 → 부적합

---

## 정리 비교

|종류|용도|DB 적합성|
|---|---|---|
|Deployment|Stateless API|부적합|
|StatefulSet|Stateful DB|적합|
|DaemonSet|노드별 에이전트|부적합|
|Job|일회성 작업|부적합|

결론  
PostgreSQL은 **상태 유지 + 고정 스토리지**가 필요하므로 StatefulSet이 가장 적합합니다.
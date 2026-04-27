# 1. 핵심 결론

```text
Recording rule은 cardinality 폭발을 직접 막는 도구가 아니다.
대신, 고비용 쿼리를 미리 계산하고 label을 축소하여
쿼리 시점의 부하를 줄이는 역할을 한다.
```

---

# 2. Cardinality 폭발이 발생하는 이유

Prometheus에서 시계열은 아래 구조로 생성된다.

```text
metric_name + label 조합 = 하나의 시계열
```

예시:

```text
http_requests_total{user_id="123", path="/user/123"}
```

👉 문제 상황

```promql
sum(rate(http_requests_total[5m])) by (user_id, path, method, status)
```

- `user_id` → 유저 수만큼 증가
    
- `path` → 동적 값이면 무한 증가
    
- label 조합 → 기하급수적으로 증가
    

👉 결과

```text
시계열 수 폭증 → 메모리/CPU 사용량 급증 → 쿼리 실패 또는 시스템 장애
```

---

# 3. Recording Rule의 역할

Recording rule의 본질:

```text
비싼 PromQL 연산을 미리 계산하여 새로운 metric으로 저장
```

---

## 동작 방식

### ❌ 기존 방식 (문제)

```text
사용자가 쿼리 실행 시
→ 전체 데이터 한 번에 계산
→ 부하 집중
```

---

### ✅ Recording Rule 적용

#### 1단계: 사전 계산 (주기 실행)

```yaml
- record: job:http_requests:rate1m
  expr: sum(rate(http_requests_total[1m])) by (job)
```

👉 특징

- user_id 제거
    
- label 축소
    
- 1분 단위로 계산
    

---

#### 2단계: 가벼운 조회

```promql
sum(job:http_requests:rate1m)
```

👉 효과

- 이미 계산된 결과 사용
    
- 쿼리 부하 감소
    

---

# 4. “1분 단위로 쪼갠다”의 의미

이 표현은 두 가지를 포함한다.

---

## 1) 시간 단위 분할

```text
rate(...[5m]) → 매번 5분 데이터 전체 계산
rate(...[1m]) → 더 작은 범위 계산
```

👉 한 번에 처리하는 데이터 감소

---

## 2) 계산 단계 분할

기존:

```text
raw metric → rate → sum → group
```

Recording rule:

```text
1단계: 일부 계산 후 저장
2단계: 재사용하여 추가 계산
```

👉 쿼리 파이프라인화

---

# 5. 중요한 오해

## ❌ Recording rule만으로 폭발 방지 불가

```yaml
- record: user:http_requests:rate1m
  expr: rate(http_requests_total[1m]) by (user_id)
```

👉 user_id 유지 → 여전히 폭발

---

## ✅ 핵심은 label 축소

```yaml
- record: job:http_requests:rate1m
  expr: sum(rate(http_requests_total[1m])) by (job)
```

👉 cardinality 감소

---

# 6. 정리 (면접용)

```text
Cardinality 폭발은 label 조합 증가로 인해 발생한다.

Recording rule은 이를 직접 해결하지는 않지만,
고비용 쿼리를 사전에 계산하고 label을 축소하여 저장함으로써
쿼리 시점의 리소스 사용량을 줄이고 시스템 안정성을 높인다.
```

---

# 7. 한줄 요약

```text
Recording rule = "폭발을 막는 도구가 아니라, 폭발하기 전에 부하를 분산시키는 전략"
```
좋다. 이건 **cardinality, churn과 같이 묶어서 반드시 이해해야 하는 개념**이다.

---

# ingestion rate

> **“Prometheus로 초당 얼마나 많은 sample(데이터 포인트)이 들어오느냐”**

---

# 1. sample이 뭐냐

Prometheus 데이터 구조:

```text
metric + label + timestamp + value
```

여기서 **한 번 scrape 될 때 들어오는 한 줄 = 1 sample**

---

## 예시

```text
http_requests_total{status="200"}  1027
```

→ 이게 1 sample

---

# 2. ingestion rate 계산 감각

> **초당 sample 수 = ingestion rate**

---

## 예시

- 타겟 100개
    
- 각 타겟이 100개 metric 노출
    
- scrape interval = 10초
    

```text
100 × 100 = 10,000 samples / 10초
→ 1,000 samples/sec
```

---

# 3. 왜 중요한가

> **Prometheus 성능의 “직접적인 부하”를 결정하는 값**

---

## ingestion rate 높으면

- WAL write 증가
    
- 디스크 I/O 증가
    
- 메모리 증가
    
- CPU 증가
    
- 결국 → **Prometheus 터짐**
    

---

# 4. cardinality / churn과 관계

이 3개는 같이 봐야 한다.

---

## 관계 구조

```text
cardinality → 시계열 개수
churn → 시계열 생성 속도
ingestion rate → 데이터 유입 속도
```

---

## 감각

- cardinality 높음 → 메모리 문제
    
- churn 높음 → TSDB 부담
    
- ingestion 높음 → 전체 시스템 부하
    

---

# 5. 무엇이 ingestion rate를 키우냐

---

## 1) scrape interval

```text
15s → 5s
```

→ 3배 증가

---

## 2) 타겟 수 증가

```text
pod 100 → 1000
```

→ 10배 증가

---

## 3) metric 개수 증가

```text
metric 100 → 1000
```

→ 10배 증가

---

## 4) 불필요한 label

→ 시계열 증가 → ingestion 증가

---

# 6. 해결 방법 (Terraform 관점)

---

## 1) scrape 간격 조정

```hcl
set {
  name  = "server.global.scrape_interval"
  value = "30s"
}
```

---

## 2) 불필요 metric drop

```hcl
relabelings = [{
  action = "drop"
  regex  = "go_.*"
}]
```

---

## 3) label 정리

```hcl
relabelings = [{
  action = "labeldrop"
  regex  = "pod|instance"
}]
```

---

## 4) 필요한 것만 scrape

```hcl
selector = {
  matchLabels = {
    app = "important-service"
  }
}
```

---

# 7. 한 줄 정리 (면접용)

> **ingestion rate는 Prometheus로 초당 유입되는 sample 수이며, 시스템 부하를 결정하는 핵심 지표다**

---

# 8. 한 단계 더 (실무 감각)

좋은 상태:

```text
cardinality 낮음
churn 낮음
ingestion 적절
```

망하는 상태:

```text
cardinality 높음
churn 높음
ingestion 높음
```

→ **Prometheus OOM + 디스크 터짐**

---

원하면 다음으로 이어서:

- “내 Prometheus ingestion rate 계산하는 방법”
    
- “몇 samples/sec부터 위험한지 기준”
    
- “Thanos / Mimir 도입 기준”
    

이건 진짜 실무 운영 레벨이다.
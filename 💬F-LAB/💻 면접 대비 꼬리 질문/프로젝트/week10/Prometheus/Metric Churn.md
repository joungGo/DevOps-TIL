# metric churn

> **“시계열(time series)이 얼마나 자주 새로 생기고 사라지느냐”**

---

# 1. 왜 생기냐 (핵심 원인)

Prometheus에서 시계열은:

```text
metric + label set = 하나의 시계열
```

즉, **label 값이 바뀌면 새로운 시계열 생성**

---

## 대표 원인

```text
pod="api-abc123" → api-xyz789
container_id="docker://..."
request_id="random-uuid"
```

→ 매번 새로운 시계열 생성됨

---

# 2. cardinality랑 차이

많이 헷갈리는 부분

|개념|의미|
|---|---|
|cardinality|전체 시계열 개수|
|metric churn|시계열 생성/삭제 속도|

---

## 감각

- cardinality: “총 몇 개냐”
    
- churn: “얼마나 빨리 바뀌냐”
    

---

# 3. 왜 문제냐

metric churn이 높으면:

### 1) 메모리 증가

- 계속 새로운 시계열 생성
    

### 2) WAL 증가

- write 폭증
    

### 3) compaction 비용 증가

### 4) query 성능 저하

---

# 4. 실전 예시

## 나쁜 예

```text
http_requests_total{user_id="123456"}
```

→ user_id가 계속 바뀜  
→ churn 폭발

---

## 또 다른 예 (Kubernetes)

```text
pod="api-xxxxx"
```

→ deploy 때마다 이름 변경  
→ churn 발생

---

# 5. 해결 방법 (Terraform / Operator 기준)

---

## 1) label 제거 (가장 중요)

```hcl
relabelings = [{
  action = "labeldrop"
  regex  = "pod|container_id|request_id"
}]
```

---

## 2) stable label만 사용

```text
app="api"
env="prod"
```

---

## 3) PodMonitor 대신 ServiceMonitor

- Pod 기반 → churn 높음
    
- Service 기반 → 안정적
    

---

## 4) aggregation 사용

```text
sum(rate(http_requests_total[5m])) by (app)
```

→ 불필요 label 제거

---

# 6. 한 줄 정리 (면접용)

> **metric churn은 label 변화로 인해 시계열이 빠르게 생성·삭제되는 현상이며, 성능 저하의 주요 원인이다**

---

원하면 다음 이어서:

- “cardinality 터지는 실제 사례”
    
- “churn vs ingestion rate 관계”
    
- “relabel_config 실전 템플릿”
    

이건 실무에서 바로 사고 나는 부분이라 이어서 보면 좋다.
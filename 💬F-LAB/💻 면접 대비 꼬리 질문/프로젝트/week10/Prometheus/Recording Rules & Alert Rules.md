# 1. Recording Rules (기록 규칙)

👉 목적: **비싼 PromQL을 미리 계산해서 새로운 메트릭으로 저장**

>High Cardinality (폭발)이 일어날 수 있는 무거운 쿼리를 단계적으로 줄여서, 시스템이 죽는 걸 방지할 수 있다.

[[Recording Rule과 Cardinality 폭발 관계 정리]]

## 왜 필요하냐

- 매번 `rate + sum + by` 같은 무거운 쿼리 실행하면 비용 큼
    
- Grafana 대시보드 성능 개선
    
- Alert에서도 재사용 가능
    

---

## 구조

```yaml
groups:
  - name: example-recording
    interval: 30s
    rules:
      - record: job:http_requests:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job)
```

---

## 핵심

```text
expr → 계산
record → 새로운 metric 이름으로 저장
```

👉 결과:

```text
job:http_requests:rate5m
```

이 메트릭이 실제로 생성됨

---

## 실무 예시

### Pod CPU 사용량 미리 계산

```yaml
- record: pod:cpu_usage:rate5m
  expr: sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)
```

👉 이후 쿼리:

```promql
pod:cpu_usage:rate5m
```

👉 훨씬 빠름

---

# 2. Alert Rules (알림 규칙)

👉 목적: **조건 만족 시 Alert 발생**

---

## 구조

```yaml
groups:
  - name: example-alert
    rules:
      - alert: HighCPUUsage
        expr: sum(rate(container_cpu_usage_seconds_total[5m])) > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CPU usage is high"
```

---

## 핵심

```text
expr → 조건
for → 유지 시간
alert → 이름
```

---

## 동작 흐름

```text
조건 만족 → for 시간 유지 → firing → Alertmanager 전달
```

---

## 실무 예시

### 에러율 알림

```yaml
- alert: HighErrorRate
  expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.1
  for: 2m
```

---

# 3. Recording vs Alert 차이

|구분|Recording Rule|Alert Rule|
|---|---|---|
|목적|계산 결과 저장|조건 감지|
|결과|새로운 metric 생성|alert 발생|
|사용 위치|Grafana, Alert|Alertmanager|
|비용|쿼리 최적화|실시간 감지|

---

# 4. 같이 쓰는 패턴 (실무 핵심)

👉 가장 중요한 구조

## 1단계: Recording

```yaml
- record: job:error_rate:5m
  expr: sum(rate(http_requests_total{status=~"5.."}[5m])) 
        / sum(rate(http_requests_total[5m]))
```

## 2단계: Alert

```yaml
- alert: HighErrorRate
  expr: job:error_rate:5m > 0.05
  for: 2m
```

👉 장점

- Alert 가벼워짐
    
- 쿼리 재사용 가능
    
- 유지보수 쉬움
    

---

# 5. Terraform 관점

Prometheus Operator 기준:

## Recording + Alert 정의

```hcl
resource "kubernetes_manifest" "prometheus_rule" {
  manifest = {
    apiVersion = "monitoring.coreos.com/v1"
    kind       = "PrometheusRule"

    metadata = {
      name      = "example-rules"
      namespace = "monitoring"
    }

    spec = {
      groups = [
        {
          name = "example"

          rules = [
            {
              record = "job:http_requests:rate5m"
              expr   = "sum(rate(http_requests_total[5m])) by (job)"
            },
            {
              alert = "HighCPUUsage"
              expr  = "sum(rate(container_cpu_usage_seconds_total[5m])) > 0.8"
              for   = "5m"
            }
          ]
        }
      ]
    }
  }
}
```

---

# 6. 한줄 정리

```text
Recording = 미리 계산해서 저장
Alert = 조건 만족 시 알림
```

---

원하면 다음 단계로

- Alertmanager 라우팅 구조 (group_by, receiver)
    
- 실무 알람 설계 (noise 줄이기)
    
- SLO 기반 alert (에러율, latency)
    

까지 이어서 Terraform 기준으로 설명해줄게.
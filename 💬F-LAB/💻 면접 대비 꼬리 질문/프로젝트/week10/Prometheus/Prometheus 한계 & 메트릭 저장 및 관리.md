# 1. 메트릭 저장 구조 (TSDB)

Prometheus는 내부적으로 **TSDB(Time Series Database)**를 사용한다.

### 기본 개념

- **Metric + Labels = 하나의 시계열**
    
- 데이터 형태:
    
    ```
    (timestamp, value)
    ```
    

---

### 디스크 구조

```
/data
 ├── wal/        # Write Ahead Log
 ├── chunks/     # 실제 데이터
 └── blocks/     # 압축된 블록
```

#### 동작 흐름

1. 데이터 수집 → 메모리(Head Block)
    
2. 동시에 WAL 기록 (복구용)
    
3. 일정 시간 후 → Block으로 압축 저장
    

---

## 여기서 발생하는 한계

### 1) 로컬 스토리지 기반 (WAL(디스크) + 메모리)

- 기본적으로 **단일 노드 디스크에 저장**
    
- 장애 발생 시 데이터 유실 가능
    

👉 해결

- PVC 사용 (Kubernetes)
    
- 또는 Thanos 같은 외부 스토리지
    

---

### 2) 수평 확장 어려움

- TSDB가 단일 인스턴스 중심 구조
    
- 샤딩/분산이 기본 지원되지 않음
    

👉 결과

- 대규모 환경에서 한계
    

---

# 2. 데이터 수집 방식 (Pull)

Prometheus는 Pull 기반

```yaml
scrape_configs:
  - job_name: "app"
    static_configs:
      - targets: ["localhost:8080"]
```

---

## 한계

### 1) 네트워크 의존성

- 대상이 다운 → 데이터 수집 불가
    

### 2) Push 기반 시스템과 충돌

- batch job, short-lived job에 약함
    

👉 해결

- Pushgateway (하지만 권장 제한적)
    

---

# 3. 메트릭 타입

- Counter
    
- Gauge
    
- Histogram
    
- Summary
    

---

## 한계

### Histogram / Summary 문제

- Summary는 aggregation 불가능
    
- Histogram은 bucket 설계 어려움
    

👉 잘못 설계하면:

- 쓸모 없는 데이터
    
- 과도한 시계열 생성
    

---

# 4. 메트릭 관리 핵심 (Cardinality)

### 핵심 개념

- 라벨 조합 수 = 시계열 개수
    

>시계열(Time Series): 시간에 따라 갑이 계속 쌓이는 구조
>(같은 metric + 같은 Label 조합에 대해 시간에 따라 계쏙 기록되는 값의 흐름)

> [!example] Prometheus 시계열이 쌓이는 과정
> 
> **상황:**  
> 애플리케이션이 다음 메트릭을 노출하고, Prometheus가 15초마다 scrape 한다.
> 
> ```plaintext
> http_requests_total{method="GET", status="200"} 0
> ```
> 
> ---
> 
> **⏱ 10:00:00**
> 
> ```plaintext
> http_requests_total{method="GET", status="200"} 0
> ```
> 
> ```
> 시계열 #1 생성
> metric: http_requests_total
> labels: method="GET", status="200"
> 
> (10:00:00, 0)
> ```
> 
> ---
> 
> **⏱ 10:00:15**
> 
> ```plaintext
> http_requests_total{method="GET", status="200"} 5
> ```
> 
> ```
> (10:00:00, 0)
> (10:00:15, 5)
> ```
> 
> ---
> 
> **⏱ 10:00:30**
> 
> ```plaintext
> http_requests_total{method="GET", status="200"} 12
> ```
> 
> ```
> (10:00:00, 0)
> (10:00:15, 5)
> (10:00:30, 12)
> ```
> 
> ---
> 
> **⏱ 10:01:00 (새로운 label 등장)**
> 
> ```plaintext
> http_requests_total{method="POST", status="200"} 3
> ```
> 
> ```
> 시계열 #2 생성
> 
> http_requests_total{method="POST", status="200"}
> (10:01:00, 3)
> ```
> 
> ---
> 
> **최종 구조**
> 
> ```
> 시계열 #1
> http_requests_total{method="GET", status="200"}
> (10:00:00, 0)
> (10:00:15, 5)
> (10:00:30, 12)
> 
> 시계열 #2
> http_requests_total{method="POST", status="200"}
> (10:01:00, 3)
> ```
> 
> ---
> 
> **핵심**
> 
> - 같은 metric + label → 기존 시계열에 계속 append
>     
> - label이 바뀌면 → 새로운 시계열 생성
>

---

## 가장 치명적인 한계

### High Cardinality 문제

예:

```
user_id=12345
session_id=abcd
```

👉 결과

- 시계열 폭발
    
- 메모리 급증
    
- Prometheus 다운
    

---

## 왜 치명적이냐

- Prometheus는 **메모리에 시계열 유지**
    
- 시계열 수 증가 = 메모리 직결
    

👉 즉,  
**설계 잘못하면 시스템이 터짐**

---

# 5. 데이터 보관 (Retention)

```bash
--storage.tsdb.retention.time=15d
```

Terraform 관점:

```hcl
args = [
  "--storage.tsdb.retention.time=15d"
]
```

---

## 한계

### 1) 장기 저장 부적합

- 기본적으로 몇 주 단위 저장
    
- 수개월~수년 데이터 관리 어려움
    

### 2) 디스크 사용량 증가

- retention 늘리면 디스크 폭증
    

---

# 6. 압축 및 블록 관리

- 2시간 단위 block 생성
    
- compaction 수행
    

---

## 한계

### 1) I/O 부담

- compaction 시 디스크 부하 증가
    

### 2) 쿼리 성능 저하

- 데이터 많아질수록 느려짐
    

---

# 7. 확장 (필수 영역)

Prometheus 단독 사용은 한계가 명확함

### 대표 확장 솔루션

- Thanos
    
- Cortex
    
- Grafana Mimir
    

---

### 역할

- 장기 저장 (S3 등)
    
- 글로벌 쿼리
    
- 수평 확장
    

---

# 8. Terraform 관점 운영 포인트

### 1) PVC 필수

```hcl
volume_claim_template {
  spec {
    access_modes = ["ReadWriteOnce"]

    resources {
      requests = {
        storage = "50Gi"
      }
    }
  }
}
```

👉 emptyDir 사용 시 데이터 전부 유실

---

### 2) retention 명확히 설정

```hcl
args = [
  "--storage.tsdb.retention.time=15d"
]
```

---

### 3) remote_write 고려

```hcl
remote_write {
  url = "http://thanos:19291/api/v1/receive"
}
```

---

### 4) scrape_interval 조정

```hcl
scrape_interval = "15s"
```

---

# 최종 정리

Prometheus는:

- 강점  
    → 간단한 구조, 강력한 쿼리(PromQL), 빠른 조회
    
- 하지만 구조적 한계
    
    1. 단일 노드 기반 (확장성 제한)
        
    2. High Cardinality에 매우 취약
        
    3. 장기 저장 부적합
        
    4. 메모리 의존 구조
        
    5. Pull 방식 한계
        

---

## 한줄 결론

👉 Prometheus는 “완성형 모니터링 시스템”이 아니라  
👉 “로컬 시계열 수집 + 단기 저장 엔진”

그래서 실무에서는 거의 항상  
→ Thanos / Mimir 같은 시스템과 함께 사용한다.

---

원하면 다음 단계로

- “카디널리티 줄이는 설계 패턴”
    
- “Prometheus 메모리 계산 공식”
    
- “Terraform으로 Thanos까지 포함한 전체 아키텍처”
    

이렇게 이어서 설명해줄게.
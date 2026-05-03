# node_exporter

> **서버(OS) 레벨 메트릭을 수집해서 Prometheus에 노출하는 exporter**

---

# 1. 뭐 하는 놈이냐

> **“리눅스 서버 상태를 /metrics로 보여주는 프로그램”**

---

## 동작 구조

```text
Linux OS
→ node_exporter
→ http://<host>:9100/metrics
→ Prometheus가 scrape
```

---

# 2. 어떤 데이터를 주냐

> **인프라(노드) 상태**

---

## 대표 메트릭

### CPU

```text
node_cpu_seconds_total
```

---

### 메모리

```text
node_memory_MemAvailable_bytes
```

---

### 디스크

```text
node_disk_read_bytes_total
node_disk_io_time_seconds_total
```

---

### 네트워크

```text
node_network_receive_bytes_total
```

---

### 파일시스템

```text
node_filesystem_avail_bytes
```

---

# 3. 핵심 특징

---

## 1) exporter다

- 직접 메트릭 생성 ❌
    
- OS 정보 “읽어서 노출” ✔
    

---

## 2) agent 형태

- 각 노드마다 1개씩 실행
    

---

## 3) pull 기반

- Prometheus가 가져감
    

---

# 4. 왜 필요한가

> **“서버가 문제인지 아닌지 판단하는 기준”**

---

## 예시

### CPU 95%

```text
node_cpu_seconds_total
```

→ 서버 과부하

---

### 디스크 꽉 참

```text
node_filesystem_avail_bytes
```

→ 장애 직전

---

# 5. cAdvisor랑 차이

많이 헷갈리는 부분

---

|항목|node_exporter|cAdvisor|
|---|---|---|
|대상|서버(OS)|컨테이너|
|관점|인프라|컨테이너|
|예|CPU, 메모리|컨테이너 CPU|

---

# 6. Prometheus Operator / Terraform 관점

---

## 보통 자동 포함됨

```hcl
# kube-prometheus-stack 설치 시
node_exporter 기본 포함
```

---

## ServiceMonitor로 연결됨

- 각 노드에 DaemonSet으로 배포
    
- Prometheus가 자동 scrape
    

---

# 7. 한계 (중요)

> **애플리케이션 상태는 모른다**

---

예:

```text
CPU 정상
```

→ 그런데 서비스는 느림

→ 이유: DB 문제 / 코드 문제

---

# 8. 한 줄 정리 (면접용)

> **node_exporter는 서버(OS) 레벨 메트릭을 수집하는 Prometheus exporter이다**

---

# 9. 실무 팁 (중요)

---

## 1) 꼭 같이 봐야 한다

- node_exporter → 인프라
    
- application metric → 서비스
    

---

## 2) alert 예시

```text
디스크 사용량 > 90%
CPU usage > 80%
```

---

## 3) cardinality는 낮음 (장점)

- label 거의 없음
    
- 안정적
    

---

원하면 이어서:

- cAdvisor (컨테이너 레벨)
    
- kubelet metrics 구조
    
- node_exporter 주요 metric 해석법
    

이건 모니터링 설계까지 이어진다.
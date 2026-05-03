# cAdvisor

> **컨테이너(Container) 레벨 메트릭을 수집하는 exporter**

---

# 1. 뭐 하는 놈이냐

> **“컨테이너가 실제로 얼마나 CPU/메모리를 쓰는지 보여주는 도구”**

---

## 동작 구조

```text
Container runtime (docker / containerd)
→ cAdvisor
→ /metrics
→ Prometheus
```

---

# 2. 어떤 데이터를 주냐

> **컨테이너 단위 리소스 사용량**

---

## 대표 메트릭

### CPU

```text
container_cpu_usage_seconds_total
```

---

### 메모리

```text
container_memory_usage_bytes
```

---

### 네트워크

```text
container_network_receive_bytes_total
```

---

### 파일시스템

```text
container_fs_usage_bytes
```

---

# 3. node_exporter vs cAdvisor 차이 (핵심)

|항목|node_exporter|cAdvisor|
|---|---|---|
|대상|서버(OS)|컨테이너|
|관점|전체 노드|개별 컨테이너|
|예|CPU 80%|어떤 컨테이너가 CPU 쓰는지|

---

## 감각

- node_exporter:
    
    > “서버 CPU 90%야”
    
- cAdvisor:
    
    > “그 중에서 nginx 컨테이너가 70% 쓰고 있음”
    

---

# 4. Kubernetes에서는 어떻게 동작하냐

중요 포인트:

> **요즘은 cAdvisor를 따로 안 띄움**

---

## 이유

- kubelet 안에 cAdvisor 포함됨
    

```text
kubelet → cAdvisor 포함 → /metrics/cadvisor
```

→ Prometheus가 kubelet에서 같이 scrape

---

# 5. Prometheus Operator 관점

---

## 자동 수집됨

```hcl
# kube-prometheus-stack 사용 시
kubelet metrics 안에 포함됨
```

→ 따로 exporter 안 띄워도 됨

---

# 6. 한계

> **컨테이너 리소스까지만 보인다**

---

못 보는 것:

- request latency
    
- error rate
    
- 비즈니스 로직
    

→ 그래서 **instrumentation 필요**

---

# 7. 언제 쓰냐 (실전)

---

## 1) 특정 컨테이너 CPU 폭증 찾기

```text
container_cpu_usage_seconds_total
```

---

## 2) OOM 원인 분석

```text
container_memory_usage_bytes
```

---

## 3) Pod 리소스 문제

- limit 초과
    
- throttling 발생
    

---

# 8. 한 줄 정리 (면접용)

> **cAdvisor는 컨테이너 단위의 CPU, 메모리, 네트워크 등의 리소스 사용량을 수집하는 exporter이다**

---

# 9. 전체 구조 한 번에 정리

```text
node_exporter → 서버
cAdvisor       → 컨테이너
kube-state-metrics → Kubernetes 상태
application metric → 서비스 내부
```

---

이 4개를 같이 써야:

> **“왜 느린지” 끝까지 추적 가능**

---

원하면 다음으로 이어서:

- kubelet metrics 구조 (cAdvisor 포함 구조)
    
- container throttling 실제 분석 방법
    
- PromQL로 CPU usage 계산 방법
    

여기부터가 진짜 실무 분석이다.
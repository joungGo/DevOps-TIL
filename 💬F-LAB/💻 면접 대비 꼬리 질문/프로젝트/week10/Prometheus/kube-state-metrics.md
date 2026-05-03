# kube-state-metrics

> **Kubernetes “리소스 상태(state)”를 메트릭으로 변환해주는 exporter**

---

# 1. 뭐 하는 놈이냐

Prometheus는 기본적으로:

- CPU / memory → 잘 봄 (node_exporter, cAdvisor)
    
- **Kubernetes 리소스 상태 → 못 봄**
    

그래서 등장한 게:

> **kube-state-metrics = “K8s 오브젝트 상태를 메트릭으로 노출”**

---

# 2. exporter인가?

맞다. 정확히는:

> **Kubernetes API → 읽어서 → /metrics로 노출하는 exporter**

---

## 동작 흐름

```text
Kubernetes API (Pod, Deployment, Node ...)
→ kube-state-metrics
→ /metrics
→ Prometheus scrape
```

---

# 3. 어떤 데이터를 주냐 (핵심)

이건 “리소스 상태”다.

---

## 예시

### Pod 상태

```text
kube_pod_status_phase{phase="Running"}
kube_pod_container_status_restarts_total
```

---

### Deployment 상태

```text
kube_deployment_spec_replicas
kube_deployment_status_replicas_available
```

---

### Node 상태

```text
kube_node_status_condition{condition="Ready"}
```

---

# 4. node_exporter / cAdvisor랑 차이

이거 구분 못하면 헷갈린다.

---

## 비교

|도구|보는 것|
|---|---|
|node_exporter|서버 상태 (CPU, 메모리)|
|cAdvisor|컨테이너 리소스|
|kube-state-metrics|Kubernetes 리소스 상태|

---

## 핵심 차이

- node_exporter → “CPU 몇 %”
    
- kube-state-metrics → “Pod 몇 개 살아있냐”
    

---

# 5. 왜 중요한가 (실무 핵심)

> **“Kubernetes 상태 기반 모니터링” 가능하게 해줌**

---

## 예시

### 1) Pod 장애 감지

```text
kube_pod_status_phase{phase="Failed"}
```

---

### 2) Deployment 문제

```text
desired replicas != available replicas
```

---

### 3) CrashLoop 감지

```text
kube_pod_container_status_restarts_total 증가
```

---

# 6. Prometheus Operator 관점

---

## 보통 자동 포함됨

```hcl
# kube-prometheus-stack 사용 시
kube-state-metrics 포함됨
```

---

## ServiceMonitor로 연결

```hcl
resource "kubernetes_manifest" "kube_state_metrics_sm" {
  manifest = {
    kind = "ServiceMonitor"
    spec = {
      selector = {
        matchLabels = {
          app = "kube-state-metrics"
        }
      }
    }
  }
}
```

---

# 7. 주의할 점 (중요)

## 1) 리소스 상태만 본다

→ CPU / memory 없음

---

## 2) cardinality 문제 있음

예:

```text
kube_pod_labels{label_app="...", label_env="..."}
```

→ label 많으면 터짐

---

# 8. 한 줄 정리 (면접용)

> **kube-state-metrics는 Kubernetes 리소스 상태를 Prometheus 메트릭으로 변환해주는 exporter이다**

---

# 9. 감 잡는 비유

- node_exporter → “서버 건강검진”
    
- cAdvisor → “컨테이너 상태”
    
- kube-state-metrics → “Kubernetes 관리 정보”
    

---

원하면 다음 이어서:

- “kube-state-metrics 때문에 cardinality 터지는 사례”
    
- “실전 alert rule (Deployment 장애 감지)”
    
- “relabel_config로 필터링하는 방법”
    

이건 진짜 운영에서 바로 쓰는 부분이다.
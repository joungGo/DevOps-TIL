# Target discovery (자동화)가 뭐냐

> **“Prometheus가 수집할 대상(target)을 자동으로 찾아주는 것”**

---

# 기존 방식 (자동화 없음)

Prometheus는 원래 이렇게 썼다:

```yaml
scrape_configs:
  - job_name: my-app
    static_configs:
      - targets:
        - 10.0.0.1:8080
        - 10.0.0.2:8080
```

문제:

- Pod IP 바뀜 → 깨짐
    
- scale out → 수동 추가
    
- 운영자가 계속 수정해야 함
    

---

# Kubernetes 환경에서 문제

Pod는 이렇게 변한다:

```text
api-abc123 (IP: 10.0.0.1)
→ 죽음
→ api-xyz789 (IP: 10.0.0.9)
```

→ IP 기반이면 **100% 깨짐**

---

# Target discovery 자동화 (핵심)

> **“라벨 기반으로 대상 자동 탐색”**

---

## Operator + ServiceMonitor 사용 시

```yaml
selector:
  matchLabels:
    app: my-app
```

이 의미:

> “app=my-app 라벨 가진 애들 다 찾아서 scrape 해”

---

## 내부에서 일어나는 일

1. Kubernetes API 조회
    
2. label=app=my-app 인 Service 찾음
    
3. 해당 Service의 Endpoint(Pod들) 찾음
    
4. Prometheus target으로 자동 등록
    

---

## 결과

Pod가 이렇게 바뀌어도:

```text
api-abc123 → api-xyz789
```

→ **자동으로 target 교체됨**

---

# 왜 중요한가

이게 없으면:

- scale 못함
    
- auto healing 깨짐
    
- 운영 지옥
    

이게 있으면:

- pod 생성 → 자동 수집
    
- pod 삭제 → 자동 제거
    
- config 수정 없음
    

---

# Terraform 관점

우리는 이렇게만 쓴다:

```hcl
selector = {
  matchLabels = {
    app = "my-app"
  }
}
```

→ “누굴 수집할지”만 정의

---

Operator가 대신 하는 것:

- target 찾기
    
- scrape config 생성
    
- 변경 시 reload
    

---

# 한 줄 정리

> **Target discovery 자동화 = “IP가 아니라 label로 수집 대상을 자동 추적하는 것”**

---

# 감 잡는 비유

- 기존: “IP 주소 적어서 관리” (엑셀)
    
- 자동화: “조건 검색으로 자동 추적” (필터)
    

---

이해됐으면 다음으로 이거 이어서 보면 좋다:

- relabel_config (자동으로 찾은 target을 필터링하는 기술)
    
- cardinality / churn이 여기서 왜 같이 터지는지
    

필요하면 이어서 설명해준다.
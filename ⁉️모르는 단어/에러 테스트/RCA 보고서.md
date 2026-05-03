RCA(Root Cause Analysis) 보고서는 인시던트의 “근본 원인”을 분석한 문서야.

단순히:

- “서버가 죽었습니다”
    

가 아니라,

- 왜 죽었는지
    
- 어떤 영향이 있었는지
    
- 어떻게 복구했는지
    
- 다시 안 발생하게 하려면 무엇을 해야 하는지
    

까지 정리하는 문서야.

보통 SRE, DevOps, 플랫폼 엔지니어링에서 장애 대응 후 작성해.

일반적인 구조는 이런 형태야:

# 1. Incident Summary

장애 요약

예:

- 발생 시간
    
- 종료 시간
    
- 영향 범위
    
- Severity
    

예시:

- 2026-05-02 14:03 ~ 14:18
    
- API 응답 실패율 65%
    
- 사용자 로그인 불가
    
- Sev-1
    

---

# 2. Timeline

시간순 이벤트 기록

예:

- 14:03 Alert 발생
    
- 14:05 On-call 확인
    
- 14:08 Pod OOM 확인
    
- 14:12 Replica 증가
    
- 14:18 정상화
    

---

# 3. Symptoms

겉으로 보인 현상

예:

- 502 증가
    
- latency 증가
    
- CPU spike
    
- Pod restart 반복
    

---

# 4. Root Cause

근본 원인

예:

- 메모리 제한값 부족
    
- 잘못된 rollout
    
- DB connection leak
    
- Node disk full
    

중요한 건 “진짜 원인”까지 파고드는 거야.

예:

- Pod 죽음 ← 현상
    
- OOMKilled ← 직접 원인
    
- memory limit 잘못 설정 ← 근본 원인
    

---

# 5. Mitigation / Recovery

복구 방법

예:

- Deployment rollback
    
- HPA scale out
    
- Node 교체
    
- 캐시 flush
    

---

# 6. Preventive Actions

재발 방지 대책

예:

- Alert 추가
    
- Load test 강화
    
- Resource limit 재조정
    
- Chaos test 추가
    

---

네가 만들려고 하는 AI Incident Investigation Platform에서는:

- Prometheus metrics
    
- Loki logs
    
- Kubernetes events
    
- Alert history
    

를 기반으로 AI가 자동으로 RCA 보고서를 생성하는 구조가 될 가능성이 커.

예를 들면:

```text
Root Cause:
A recent deployment introduced a memory leak in the comment-service.
Pods exceeded memory limits and were OOMKilled repeatedly,
causing elevated API error rates.

Evidence:
- OOMKilled events observed
- memory_usage_bytes steadily increased
- pod restart count spike
- errors increased after deployment revision 42
```

같은 식으로 생성하는 거야.
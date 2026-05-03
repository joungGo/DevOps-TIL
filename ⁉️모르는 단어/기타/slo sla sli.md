## 한 줄 요약

- **SLI = 측정값**
    
- **SLO = 목표**
    
- **SLA = 약속(계약)**
    

---

## 1. SLI (Service Level Indicator)

**서비스 상태를 나타내는 실제 지표**

- 예시
    
    - 응답 시간(latency)
        
    - 에러율(error rate)
        
    - 가용성(availability)
        

👉 “지금 서비스가 얼마나 잘 되고 있는지 측정하는 값” ([devopskit.tech](https://devopskit.tech/en/posts/sla-slo-sli/?utm_source=chatgpt.com "What are SLO, SLA, and SLI? Simple Explanation with Examples | Real-World DevOps: CI/CD, Monitoring & Kubernetes Guides"))

---

## 2. SLO (Service Level Objective)

**SLI에 대한 목표값 (내부 기준)**

- 예시
    
    - “99.9% 요청이 300ms 이하로 응답해야 한다”
        
    - “월간 가용성 99.95% 유지”
        

👉 “이 정도면 괜찮다”라는 내부 목표 ([devopskit.tech](https://devopskit.tech/en/posts/sla-slo-sli/?utm_source=chatgpt.com "What are SLO, SLA, and SLI? Simple Explanation with Examples | Real-World DevOps: CI/CD, Monitoring & Kubernetes Guides"))

---

## 3. SLA (Service Level Agreement)

**고객과의 계약 (외부 약속 + 패널티 포함)**

- 예시
    
    - “99.9% 미만이면 요금 10% 환불”
        
    - “응답 시간 초과 시 크레딧 지급”
        

👉 목표를 못 지키면 **보상/패널티가 있는 계약** ([devopskit.tech](https://devopskit.tech/en/posts/sla-slo-sli/?utm_source=chatgpt.com "What are SLO, SLA, and SLI? Simple Explanation with Examples | Real-World DevOps: CI/CD, Monitoring & Kubernetes Guides"))

---

## 관계 (중요)

```
SLI (측정) → SLO (목표) → SLA (계약)
```

- SLI로 측정해서
    
- SLO 달성 여부 판단하고
    
- 실패하면 SLA 위반
    

👉 SLO는 SLA를 깨지 않기 위한 내부 안전선 역할 ([DevOps Bible](https://devopsbible.com/docs/sla-slo-sli/?utm_source=chatgpt.com "SLAs, SLOs, and SLIs | DevOps Bible"))

---

## 면접용 핵심 답변

“SLI는 서비스의 상태를 측정하는 지표이고, SLO는 그 지표에 대한 내부 목표입니다. SLA는 고객과의 계약으로, SLO를 지키지 못했을 때 보상이나 패널티가 포함됩니다. 즉 SLI로 측정하고, SLO로 관리하며, SLA로 책임을 지는 구조입니다.”

---

원하면 Prometheus 기준으로 SLI/SLO 설정 예시도 같이 정리해줄게.
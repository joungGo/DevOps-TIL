`Cloud Service Provider`를 의미해.

즉:

- AWS
    
- GCP
    
- Azure
    

같은 클라우드 제공 업체를 말하는 거야.

대화 내용을 해석하면:

- `ALB`를 많이 쓰면 AWS 기능에 강하게 묶이고
    
- 비용도 증가하고
    
- 특정 클라우드 기능에 의존하게 되므로
    
- `NLB + ingress-nginx` 구조로 직접 제어하는 연습을 하겠다는 의미야.
    

특히:

- ALB Ingress Controller
    
- AWS Load Balancer Controller
    
- ACM
    
- WAF
    
- TargetGroupBinding
    

같은 AWS 전용 기능을 많이 쓰면 Kubernetes 자체보다는 AWS 관리형 기능에 의존하게 되는데,  
그걸 보통:

> "CSP 종속적이다"  
> "벤더 락인(vendor lock-in) 된다"

라고 표현해.
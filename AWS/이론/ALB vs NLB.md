두 방식의 핵심 차이를 다이어그램과 함께 설명해 드릴게요.## 핵심 차이점
![[Pasted image 20260407182015.png]]
### 1. L7 처리 위치

- **NGINX 방식**: 클러스터 내부의 NGINX Pod가 L7 라우팅 담당. NLB는 단순히 TCP 트래픽만 전달
- **ALB 방식**: AWS ALB가 직접 L7 라우팅 처리. 경로 기반, 호스트 기반 라우팅을 AWS 인프라에서 수행

### 2. 네트워크 홉 수

- **NGINX 방식**: 인터넷 → NLB → NGINX → Service → Pod (4홉)
- **ALB 방식**: 인터넷 → ALB → Pod (2홉, IP 타입 타겟 그룹 사용 시)

### 3. 운영 관점

|항목|NGINX|ALB Controller|
|---|---|---|
|**관리 주체**|사용자가 NGINX Pod 운영|AWS가 ALB 관리|
|**스케일링**|NGINX HPA 직접 설정|AWS 자동 스케일링|
|**SSL 종료**|NGINX에서 처리|ALB에서 처리 (ACM 연동)|
|**커스터마이징**|자유로움 (nginx.conf)|AWS 기능 범위 내|
|**비용**|EC2 + NLB 비용|ALB 시간당 + LCU 비용|

### 4. 언제 어떤 걸 선택?

- **NGINX**: 복잡한 라우팅 규칙, 커스텀 헤더 조작, 레이트 리미팅 등 세밀한 제어 필요할 때
- **ALB**: AWS 서비스 통합(WAF, Cognito), 운영 부담 최소화, 간단한 라우팅 규칙일 때

> [!info] ALB Controller 동작 원리
> 
> ## 핵심 개념
> 
> **selector는 ALB가 아니라 Service에 정의**되어 있고, ALB Controller가 그걸 참조한다.
> 
> ![[Pasted image 20260407183728.png]]
> 
> ## 동작 흐름
> 
> ```
> ① Ingress: "api-service로 보내줘"
>      ↓
> ② Service: "app: api 라벨 가진 Pod들이 내 대상이야" (selector 정의)
>      ↓
> ③ ALB Controller: Service의 selector로 Pod 목록 조회 → Pod IP 추출
>      ↓
> ④ AWS Target Group: Pod IP들 등록 (10.0.1.15, 10.0.2.23...)
>      ↓
> ⑤ ALB: Target Group 보고 Pod IP로 직접 트래픽 전달
> ```
> 
> ## 각 컴포넌트 역할
> 
> | 컴포넌트 | 역할 |
> |----------|------|
> | **ALB** | Target Group에 등록된 IP로 트래픽 전달 (selector 모름) |
> | **ALB Controller** | K8s 리소스 감시 → AWS Target Group 동기화 (중간 번역기) |
> | **Service** | 트래픽 경로 아님, Pod 목록 정보 제공용 |
> 
> ## Pod 스케일 아웃 시
> 
> 1. 새 Pod 생성 (label: `app: api`)
> 2. Service의 Endpoints에 자동 추가
> 3. ALB Controller가 감지
> 4. Target Group에 새 Pod IP 등록
> 5. ALB가 새 Pod로도 트래픽 분산

---

## 한 줄 정의

```
ALB (Application Load Balancer)
→ HTTP/HTTPS 내용을 보고 라우팅

NLB (Network Load Balancer)
→ IP/포트만 보고 라우팅 (내용 안 봄)
```

---

## 언제 뭘 쓰나

- **NGINX**: 복잡한 라우팅 규칙, 커스텀 헤더 조작, 레이트 리미팅 등 세밀한 제어 필요할 때
- **ALB**: AWS 서비스 통합(WAF, Cognito), 운영 부담 최소화, 간단한 라우팅 규칙일 때
  
> - AWS 서비스 통합: ALB는 AWS 네이티브 서비스라서 **다른 AWS 서비스들과 클릭 몇 번으로 연결**됩니다.
> - 운영 부담이 줄어드는 이유는 아래 표와 같다.

|항목|NGINX 방식|ALB 방식|
|---|---|---|
|**스케일링**|HPA 설정, 리소스 튜닝, Pod 수 관리|AWS가 자동으로 처리 (신경 쓸 것 없음)|
|**고가용성**|여러 AZ에 NGINX Pod 분산 배치 직접 설계|기본 내장 (Multi-AZ)|
|**SSL 인증서**|인증서 발급, 갱신, 시크릿 관리|ACM에서 자동 발급/갱신|
|**보안 패치**|NGINX 버전 업데이트 직접|AWS가 알아서 패치|
|**모니터링**|Prometheus, Grafana 등 직접 구축|CloudWatch 기본 연동|
|**로그**|로그 수집 파이프라인 구축|S3로 자동 저장 옵션|
|**장애 대응**|NGINX Pod 죽으면 직접 트러블슈팅|AWS 책임 (SLA 보장)|

---

**ALB를 쓰는 경우**

```
- 웹 애플리케이션 (HTTP/HTTPS)
- URL 기반으로 여러 서비스로 나눠야 할 때
- 도메인별로 다른 서비스로 라우팅
- WAF(웹 방화벽) 연동 필요할 때
- 인증/SSL 종료를 LB에서 처리하고 싶을 때
→ Week5에서 url-shortener 배포 시 ALB 사용
```

**NLB를 쓰는 경우**

```
- TCP/UDP 통신 (DB, Redis, gRPC)
- 극도로 낮은 레이턴시 필요 (게임, 금융)
- 고정 IP가 필요할 때 (방화벽 화이트리스트)
- 초당 수백만 요청 처리
- 클라이언트 원본 IP를 그대로 받아야 할 때
```

---

## EKS에서 사용 방법

**ALB — Ingress로 생성**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        backend:
          service:
            name: my-svc
            port:
              number: 80
```

**NLB — Service로 생성**

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
  type: LoadBalancer
  ports:
  - port: 80
```

---

## 한 줄 요약

```
ALB = 내용(URL/헤더)을 보고 똑똑하게 라우팅 → 웹 앱
NLB = 내용 안 보고 빠르게 전달            → DB/게임/금융
```

![[Pasted image 20260401225919.png]]


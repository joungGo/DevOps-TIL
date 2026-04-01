## 한 줄 정의

```
ALB (Application Load Balancer)
→ HTTP/HTTPS 내용을 보고 라우팅

NLB (Network Load Balancer)
→ IP/포트만 보고 라우팅 (내용 안 봄)
```

---

## OSI 계층 차이

```
ALB → L7 (애플리케이션 계층)
      HTTP 헤더, URL 경로, 호스트명을 봄

NLB → L4 (전송 계층)
      IP 주소, 포트, 프로토콜만 봄
```

---

## 라우팅 방식 차이

**ALB**

```
/api/*      → API 서버 Pod
/admin/*    → Admin 서버 Pod
/static/*   → 정적 파일 서버 Pod

host: api.example.com   → API 클러스터
host: admin.example.com → Admin 클러스터
```

**NLB**

```
포트 3306 → DB 서버
포트 6379 → Redis 서버
포트 8080 → 앱 서버

내용은 전혀 안 봄, 포트만 보고 전달
```

---

## 핵심 차이 비교

```
                ALB             NLB
계층            L7              L4
프로토콜        HTTP/HTTPS      TCP/UDP/TLS
라우팅 기준     URL/헤더/호스트  IP/포트
속도            상대적으로 느림  매우 빠름
고정 IP         없음            있음 (EIP 할당)
WebSocket       지원            지원
gRPC            지원            지원
TLS 종료        ALB에서 처리    NLB 또는 Pod에서 처리
클라이언트 IP   X-Forwarded-For 원본 IP 그대로
가격            비교적 비쌈     비교적 저렴
```

---

## 언제 뭘 쓰나

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


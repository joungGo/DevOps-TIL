## 개요

두 구조 모두:

```text
Ingress 리소스는 ingress-nginx Controller가 처리
```

한다는 공통점이 있다.

즉 둘 다:

```yaml
ingressClassName: nginx
```

를 사용하며,  
실제 host/path routing은 ingress-nginx가 수행한다.

차이점은:

```text
앞단 Load Balancer가 어디까지 처리하느냐
```

이다.

---

# 1. NLB → ingress-nginx 구조

## 아키텍처

```text
Client
 ↓ HTTPS
NLB
 ↓ HTTPS(TCP 그대로 전달)
ingress-nginx
 ↓ HTTP
Service
 ↓ 
Pod
```

---

## 동작 방식

NLB는 L4(TCP) 로드밸런서로 동작한다.

즉:

- TCP 연결 전달
    
- 포트 전달
    

만 수행하며 HTTP 내용을 해석하지 않는다.

실제 L7 처리는 ingress-nginx가 담당한다.

즉 ingress-nginx가:

- HTTPS 종료(TLS termination)
    
- Host/Path Routing
    
- HTTP → HTTPS Redirect
    
- Rate Limit
    
- IP Allow/Block
    

등을 모두 수행한다.

---

## HTTPS 처리 흐름

```text
Client
 ↓ HTTPS
NLB
 ↓ TCP 443 전달
ingress-nginx
 ↓ TLS 종료
Application
```

즉 HTTPS 종료 위치는 ingress-nginx이다.

---

## 특징

### 장점

#### 1. ingress-nginx 기능을 완전히 활용 가능

annotation 기반 기능을 가장 자연스럽게 사용할 수 있다.

예:

```yaml
nginx.ingress.kubernetes.io/limit-rps: "10"
```

---

#### 2. Kubernetes Native 구조

AWS 의존성이 낮다.

즉:

- 온프레미스
    
- Minikube
    
- K3s
    
- 다른 Cloud
    

에서도 유사하게 구성 가능하다.

---

#### 3. 구조가 단순함

L7 처리 주체가 ingress-nginx 하나뿐이다.

---

## 단점

### 1. TLS 인증서 관리 필요

ingress-nginx가 직접 인증서를 관리해야 한다.

즉:

- cert-manager
    
- Let's Encrypt
    
- Kubernetes Secret
    

등을 사용해야 한다.

---

### 2. AWS WAF 연동이 어려움

AWS WAF는 일반적으로:

- ALB
    
- CloudFront
    

에 연결한다.

따라서 NLB 구조에서는 WAF 연동이 상대적으로 어렵다.

---

# 2. ALB → ingress-nginx 구조

## 아키텍처

```text
Client
 ↓
ALB (L7)
 ↓
ingress-nginx
 ↓
Service
 ↓
Pod
```

---

## 동작 방식

ALB가 먼저 L7 처리를 수행한다.

즉 ALB가:

- HTTPS 종료
    
- ACM 인증서 적용
    
- AWS WAF 적용
    

등을 담당한다.

그 이후 ingress-nginx가 내부 ingress routing을 수행한다.

즉 ingress-nginx는:

- Host/Path Routing
    
- Rate Limit
    
- Rewrite
    
- 내부 정책 처리
    

등을 담당한다.

---

## HTTPS 처리 흐름

```text
Client
 ↓ HTTPS
ALB
 ↓ HTTP
ingress-nginx
 ↓
Application
```

즉 HTTPS 종료 위치는 ALB이다.

---

## 특징

### 장점

#### 1. ACM 사용이 매우 쉬움

AWS ACM 인증서를 ALB에 바로 연결 가능하다.

즉 TLS 자동 갱신 및 관리가 쉬워진다.

---

#### 2. AWS WAF 연동 가능

AWS WAF를 ALB에 연결하여:

- SQL Injection 방어
    
- IP Block
    
- Bot 차단
    

등을 쉽게 적용 가능하다.

---

#### 3. AWS 운영 친화적

ALB가 AWS Managed 서비스이므로:

- TLS 최적화
    
- 고가용성
    
- 인증서 갱신
    

등을 AWS가 관리한다.

---

## 단점

### 1. 구조가 복잡함

L7 처리 주체가 두 개 존재한다.

```text
ALB (L7)
 ↓
ingress-nginx (L7)
```

---

### 2. 역할 중복 가능성

예를 들어:

- Redirect
    
- Allow/Block
    

등을:

- ALB에서 처리할지
    
- ingress-nginx에서 처리할지
    

애매해질 수 있다.

---

### 3. 비용 증가 가능성

ALB 사용 비용이 추가된다.

---

# 핵심 차이 비교

|항목|NLB → ingress-nginx|ALB → ingress-nginx|
|---|---|---|
|앞단 LB|NLB(L4)|ALB(L7)|
|HTTPS 종료 위치|ingress-nginx|ALB|
|ACM 사용|어려움|매우 쉬움|
|WAF 연결|어려움|매우 쉬움|
|ingress 처리 주체|ingress-nginx|ingress-nginx|
|L7 처리 주체|ingress-nginx 하나|ALB + ingress-nginx|
|구조 복잡도|단순|상대적으로 복잡|
|AWS 의존성|낮음|높음|
|실무 AWS 친화성|보통|높음|

---

# 기존 AWS Load Balancer Controller 방식과의 차이

기존 방식은:

```text
Ingress(class: alb)
 ↓
AWS Load Balancer Controller
 ↓
ALB 생성 및 라우팅 구성
```

구조였다.

즉:

```text
ALB 자체가 ingress 역할
```

을 수행했다.

반면 ingress-nginx 구조에서는:

```text
Ingress(class: nginx)
 ↓
ingress-nginx Controller
 ↓
nginx.conf 생성
```

즉 ingress-nginx가 실제 ingress 역할을 수행한다.

---

> [!info] ALB → ingress-nginx vs NLB → ingress-nginx 핵심 정리
>
> ## 공통점
>
> 두 구조 모두:
>
> ```text
> Ingress(class: nginx)
>  ↓
> ingress-nginx-controller
> ```
>
> 형태이며,
> 실제 Ingress 리소스를 읽는 주체는 둘 다 `ingress-nginx-controller`이다.
>
> 즉 둘 다:
>
> - host/path routing
> - nginx annotation 처리
> - rate limit
> - rewrite
>
> 등을 ingress-nginx가 수행한다.
>
> ---
>
> ## 1. NLB → ingress-nginx 구조
>
> ### 구조
>
> ```text
> Client
>  ↓ HTTPS
> NLB (L4)
>  ↓ TCP 그대로 전달
> ingress-nginx (L7)
>  ↓
> Service
>  ↓
> Pod
> ```
>
> ---
>
> ### 핵심 특징
>
> NLB는 L4(Network Load Balancer)이므로:
>
> - TCP/UDP 전달만 수행
> - HTTP 내용을 이해하지 못함
> - HTTPS 복호화 불가능
>
> 즉:
>
> ```text
> HTTPS(TCP)를 그대로 nginx에 전달
> ```
>
> 한다.
>
> 따라서 ingress-nginx가 직접:
>
> - TLS 종료(HTTPS 복호화)
> - HTTP 해석
> - ingress routing
> - rate limit
> - allow/block
>
> 등을 모두 처리한다.
>
> ---
>
> ### HTTPS 처리 흐름
>
> ```text
> Client
>  ↓ HTTPS
> NLB
>  ↓ HTTPS 그대로 전달
> ingress-nginx
>  ↓ TLS 종료
> HTTP
>  ↓
> Pod
> ```
>
> 즉:
>
> ```text
> L7 처리 주체 = ingress-nginx 하나
> ```
>
> 이다.
>
> ---
>
> ### 장점
>
> - ingress-nginx 기능을 가장 자연스럽게 사용 가능
> - Kubernetes Native 구조
> - AWS 의존성이 낮음
> - 구조가 단순함
>
> ---
>
> ### 단점
>
> - TLS 인증서 직접 관리 필요
> - ACM 사용 어려움
> - AWS WAF 연결 어려움
>
> ---
>
> ## 2. ALB → ingress-nginx 구조
>
> ### 구조
>
> ```text
> Client
>  ↓ HTTPS
> ALB (L7)
>  ↓ HTTP
> ingress-nginx
>  ↓
> Service
>  ↓
> Pod
> ```
>
> ---
>
> ### 핵심 특징
>
> ALB는 L7(Application Load Balancer)이므로:
>
> - HTTPS 내용을 이해 가능
> - Host/Header/Path 분석 가능
> - ACM/WAF 연동 가능
>
> 이를 위해 ALB가 먼저:
>
> ```text
> TLS 종료(TLS Termination)
> ```
>
> 를 수행한다.
>
> 즉:
>
> ```text
> HTTPS 복호화를 ALB가 먼저 수행
> ```
>
> 한 뒤 내부적으로 HTTP로 ingress-nginx에 전달한다.
>
> ---
>
> ### HTTPS 처리 흐름
>
> ```text
> Client
>  ↓ HTTPS
> ALB
>  ↓ TLS 종료
> HTTP
>  ↓
> ingress-nginx
>  ↓
> Pod
> ```
>
> ---
>
> ## 왜 L7 처리 주체가 2개인가?
>
> ALB도 L7,
> ingress-nginx도 L7이기 때문이다.
>
> ---
>
> ### ALB 역할
>
> AWS Edge/Gateway 역할:
>
> - HTTPS 종료(TLS Termination)
> - ACM 인증서 사용
> - AWS WAF 연결
> - 외부 인터넷 트래픽 처리
>
> ---
>
> ### ingress-nginx 역할
>
> Kubernetes 내부 ingress 처리:
>
> - ingress routing
> - nginx annotation 처리
> - rate limit
> - rewrite
> - 내부 reverse proxy 역할
>
> ---
>
> ## 핵심 차이
>
> | 항목 | NLB → ingress-nginx | ALB → ingress-nginx |
> |---|---|---|
> | LB 계층 | L4 | L7 |
> | HTTPS 종료 위치 | ingress-nginx | ALB |
> | ACM 사용 | 어려움 | 쉬움 |
> | WAF 연결 | 어려움 | 쉬움 |
> | L7 처리 주체 | ingress-nginx 하나 | ALB + ingress-nginx |
> | 구조 복잡도 | 단순 | 상대적으로 복잡 |
>
> ---
>
> ## 가장 중요한 핵심
>
> ### NLB → ingress-nginx
>
> ```text
> NLB는 TCP만 전달
> 실제 웹 처리는 nginx가 전부 수행
> ```
>
> ---
>
> ### ALB → ingress-nginx
>
> ```text
> ALB가 HTTPS/WAF 처리
> ingress-nginx가 Kubernetes ingress 처리
> ```
>
> 즉:
>
> ```text
> "HTTPS/TLS/WAF를 어디서 처리하느냐"
> ```
>
> 가 두 구조의 가장 큰 차이이다.

---
# 과제 관점 추천

## ingress-nginx 학습 중심

다음 구조가 가장 교육적이다.

```text
NLB
 ↓
ingress-nginx
```

이유:

- ingress-nginx 역할 이해 가능
    
- annotation 기반 제어 학습 가능
    
- L7 처리 흐름 이해 가능
    

---

## AWS 운영 실무까지 고려

다음 구조가 더 실무적이다.

```text
ALB
 ↓
ingress-nginx
```

이유:

- ACM 사용 편리
    
- WAF 적용 용이
    
- AWS Native 보안 기능 활용 가능
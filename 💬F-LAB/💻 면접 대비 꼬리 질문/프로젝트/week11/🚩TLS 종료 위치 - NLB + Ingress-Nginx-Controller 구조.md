> L7 처리를 한다는게 TLS 종료도 같이 한다는 의미는 아니다.

맞아요. `ingress-nginx-controller`가 L7 라우팅을 한다고 해서 **반드시 TLS 종료까지 ingress-nginx가 한다는 뜻은 아닙니다.**  
TLS 종료는 **어디에 인증서를 두고, 어느 계층에서 HTTPS를 해제하느냐**에 따라 달라집니다.

NLB 구조에서는 보통 3가지 경우가 있습니다.

1. `ingress-nginx`에서 TLS 종료
가장 흔한 Kubernetes 방식 중 하나입니다.

흐름:
- 사용자 `HTTPS`
- `NLB`는 TCP/443 전달만 함
- `ingress-nginx-controller`가 인증서로 TLS 종료
- 그 뒤 HTTP로 앱 Service에 프록시

즉:
```text
User -> NLB(TCP pass-through) -> ingress-nginx(여기서 TLS 종료) -> App
```

이 경우 필요:
- Kubernetes `Secret`에 TLS 인증서/키 저장
- Ingress에 `tls:` 설정

---

2. NLB에서 TLS 종료
AWS NLB도 TLS listener를 둘 수 있어서, **NLB에 ACM 인증서를 붙여 TLS 종료**할 수 있습니다.

흐름:
- 사용자 `HTTPS`
- `NLB`가 TLS 종료
- 이후 평문 HTTP 또는 다시 TLS로 `ingress-nginx`에 전달
- `ingress-nginx`는 L7 라우팅만 수행

즉:
```text
User -> NLB(여기서 TLS 종료) -> ingress-nginx -> App
```

이 경우 필요:
- Service annotation으로 NLB TLS listener 설정
- ACM 인증서 ARN 연결

---

3. TLS 종료를 안 하고 end-to-end TLS
드물지만 가능합니다.

흐름:
- 사용자 `HTTPS`
- `NLB`는 그대로 TCP 전달
- `ingress-nginx`도 HTTPS upstream으로 넘김
- 앱까지 TLS 유지

다만 지금 네 구조에서 일반적으로 생각하는 건 1번 또는 2번입니다.

중요한 구분:
- `NLB`는 ALB처럼 풍부한 L7 기능을 기본 제공하는 쪽은 아님
- 하지만 **TLS 종료 자체는 할 수 있음**
- `ingress-nginx`도 TLS 종료 가능
- 그래서 “NLB 구조면 TLS 종료는 무조건 ingress-nginx”는 아님

어디서 TLS 종료하는지 확인하는 포인트:
- `ingress-nginx` Ingress에 `tls:`가 있는지
- `ingress-nginx-controller Service` annotation에 ACM 인증서 ARN / SSL ports 설정이 있는지
- NLB listener가 `TCP:443`인지 `TLS:443`인지

한 줄로 정리하면:
- NLB 구조에서 TLS 종료는 **NLB에서 할 수도 있고 ingress-nginx에서 할 수도 있습니다**
- 실제 위치는 **인증서를 어디에 붙였는지**로 결정됩니다

원하면 다음엔
- `NLB에서 TLS 종료하는 설정 예시`
- `ingress-nginx에서 TLS 종료하는 설정 예시`
두 개를 YAML로 비교해드릴게요.
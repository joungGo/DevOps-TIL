정확히 말하면, **ACM 방식을 쓴다고 해서 ingress-nginx가 직접 HTTPS 종료를 “원천적으로 못하는 것”은 아닙니다.**  
하지만 **일반적인 ACM 퍼블릭 인증서를 그대로 가져다가 ingress-nginx가 직접 종료하는 방식은 보통 안 됩니다.**

핵심은 인증서가 어디에 있느냐예요.

1. `ingress-nginx`가 직접 HTTPS 종료하려면
- `tls.crt`
- `tls.key`

형태의 인증서/개인키가 **Kubernetes Secret**으로 있어야 합니다.

즉 ingress-nginx는 보통 이런 식으로 동작합니다:
- Ingress `spec.tls`
- Secret 참조
- nginx Pod가 그 Secret을 읽어서 TLS 종료

2. ACM 퍼블릭 인증서는
- AWS가 관리하는 인증서이고
- 일반적으로 개인키를 내려받아 `tls.key` 형태로 꺼내 쓸 수 없습니다

즉 보통은:
- ALB
- NLB
- CloudFront
같은 **AWS 서비스가 TLS 종료 지점에서 사용**합니다.

그래서 일반적인 의미의 `ACM 방식`은 보통:
- **AWS 쪽(NLB/ALB)에서 TLS 종료**
를 뜻합니다.

즉:
- `ACM 사용` = 보통 AWS 진입점에서 TLS 종료
- `ingress-nginx 직접 HTTPS 종료` = 보통 cert-manager/Secret 기반

그래서 질문에 짧게 답하면:
- **ACM 방식을 쓰면 보통 ingress-nginx가 직접 HTTPS 종료하지 않습니다**
- 이유는 ACM 인증서를 nginx Pod 안 Secret처럼 직접 쓰는 모델이 아니기 때문입니다

예외적으로는:
- ACM에서 발급받은 게 아니라
- 네가 직접 가진 인증서/키를 Secret으로 넣으면
- ingress-nginx가 직접 종료 가능

하지만 그건 “ACM 인증서를 ingress-nginx가 직접 쓴다”는 의미는 아닙니다.

정리:
- `cert-manager + Secret` -> ingress-nginx 직접 TLS 종료
- `ACM + NLB/ALB` -> AWS 쪽 TLS 종료
- 둘을 동시에 섞는 건 가능하지만, 보통 메인 종료 지점은 하나로 잡습니다

한 줄 요약:
- **ACM 방식을 메인으로 쓰면 보통 ingress-nginx 직접 HTTPS 종료는 안 한다고 보면 됩니다.**
- `week11` 문서도 그래서 메인 경로를 `cert-manager + ingress-nginx 직접 TLS 종료`로 잡고, `ACM`은 대안 경로로 분리해 둔 거예요.

원하면 다음엔
- `cert-manager 방식`
- `ACM + NLB 방식`
두 구조를 TLS 종료 지점 중심으로 딱 비교해드릴게요.
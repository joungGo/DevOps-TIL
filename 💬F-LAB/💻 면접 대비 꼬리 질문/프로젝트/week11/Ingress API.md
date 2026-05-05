Kubernetes Ingress API는 클러스터 외부에서 들어오는 HTTP/HTTPS 트래픽을 내부 Service로 라우팅하기 위한 Kubernetes 표준 리소스야.

쉽게 말하면:

```text
외부 사용자
   ↓
Ingress Controller
   ↓
Service
   ↓
Pod
```

구조를 정의하는 API라고 보면 돼.

Ingress 자체는 “설정”만 정의하고,  
실제 트래픽 처리는 Ingress Controller가 담당해.

대표적인 Controller:

- ingress-nginx
    
- Traefik
    
- HAProxy
    
- Kong
    
- AWS Load Balancer Controller
    

예시:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-service
                port:
                  number: 80
```

이 의미는:

```text
app.example.com 요청이 오면
→ app-service:80 으로 전달
```

이야.

핵심 특징:

|기능|설명|
|---|---|
|Host 기반 라우팅|api.example.com|
|Path 기반 라우팅|/api, /admin|
|TLS 지원|HTTPS|
|L7 처리|HTTP Header, URL 기반|
|Reverse Proxy|가능|

예:

```text
/api → api-service
/admin → admin-service
```

Ingress가 필요한 이유는:

Pod/Service는 기본적으로 클러스터 내부용이기 때문이야.

외부 공개를 위해:

- NodePort
    
- LoadBalancer
    
- Ingress
    

중 하나를 사용해.

보통 운영에서는:

```text
Internet
  ↓
ALB / NLB
  ↓
Ingress Controller
  ↓
Service
  ↓
Pod
```

형태가 많아.

특히 ingress-nginx는:

- 내부적으로 nginx reverse proxy를 띄우고
    
- Ingress 리소스를 읽어서
    
- nginx.conf를 자동 생성해.
    

즉:

```text
Ingress API = 선언
Ingress Controller = 실제 동작 엔진
```

관계야.

하지만 Ingress에는 한계가 있었어:

- annotation 남발
    
- 구현체마다 설정 다름
    
- TCP/UDP 제한
    
- traffic split 어려움
    
- 역할 분리 부족
    

그래서 나온 차세대 표준이:

- Gateway API
    

야.

즉 흐름은:

```text
Ingress API
  ↓ (한계)
Gateway API
```

Terraform 관점에서는 보통:

```hcl
resource "kubernetes_ingress_v1" "app" {
  metadata {
    name = "app-ingress"
  }

  spec {
    ingress_class_name = "nginx"

    rule {
      host = "app.example.com"

      http {
        path {
          path = "/"

          backend {
            service {
              name = "app-service"
              port {
                number = 80
              }
            }
          }
        }
      }
    }
  }
}
```

처럼 관리해.

| 항목             | `AWS ALB Controller` 방식                | `ingress-nginx Controller` 방식                    |
| -------------- | -------------------------------------- | ------------------------------------------------ |
| L7 라우팅 위치      | AWS ALB                                | 클러스터 내부 `nginx-controller`                       |
| 앞단 LB          | ALB 자체가 L7 LB                          | 보통 AWS LB(NLB/L4) + NGINX(L7)                    |
| 운영 주체          | AWS가 LB 데이터 플레인 관리                     | 팀이 NGINX 컨트롤러 운영                                 |
| Ingress 증가 시   | ALB가 늘어날 가능성 큼                         | 보통 기존 LB/NGINX에 규칙만 추가                           |
| 비용 구조          | 서비스 수가 늘면 ALB 비용 증가 가능                 | 공용 진입점으로 비용 절감 가능                                |
| AWS 연동         | ACM, WAF, 보안그룹, Target Group 연동이 자연스러움 | 가능은 하지만 ALB보다 덜 직접적                              |
| 설정 유연성         | AWS annotation 범위 내에서 제어               | rewrite, header, rate limit, custom nginx 설정에 강함 |
| 장애 성격          | 외부 진입점은 ALB지만 AWS 관리형                  | NGINX 계층이 직접적인 운영 장애 포인트                         |
| 가시성/운영 난이도     | 비교적 단순                                 | 튜닝/업그레이드/HA 설계 필요                                |
| 멀티 서비스 통합      | 공유 설정 없으면 ALB가 분산될 수 있음                | 여러 서비스를 하나 ingress 계층에 묶기 쉬움                     |
| Kubernetes 이식성 | AWS 의존적                                | 클라우드/온프렘 간 이식성 좋음                                |
| 적합한 스타일        | AWS 네이티브 운영                            | 쿠버네티스 중심 공용 ingress 운영                           |

**장단점 요약**

`AWS ALB Controller` 장점:
1. AWS EKS 환경과 매우 잘 맞습니다.
2. ALB, ACM, WAF, 보안그룹 등 AWS 기능 연동이 편합니다.
3. 우리가 NGINX 프록시 계층을 직접 운영하지 않아도 됩니다.
4. 운영 구성이 비교적 단순합니다.

`AWS ALB Controller` 단점:
1. 서비스/Ingress가 늘면 ALB 수도 늘어 비용이 커질 수 있습니다.
2. 세밀한 HTTP 프록시 커스터마이징은 NGINX보다 제한적입니다.
3. AWS 의존성이 강합니다.
4. 멀티 클라우드/온프렘으로 가져가기 불편합니다.

`ingress-nginx` 장점:
1. 여러 앱을 하나의 ingress 계층으로 묶기 쉽습니다.
2. path rewrite, header 조작, rate limit 등 L7 제어가 강력합니다.
3. Kubernetes 표준적인 운영 감각에 가깝습니다.
4. 클라우드 종속성이 낮아 이식성이 좋습니다.

`ingress-nginx` 단점:
1. 컨트롤러 자체를 직접 운영해야 합니다.
2. replica, rollout, 장애 복구, 업그레이드 전략을 신경 써야 합니다.
3. ingress 계층 문제가 여러 서비스에 동시에 영향을 줄 수 있습니다.
4. AWS 네이티브 기능 연동은 ALB 방식보다 덜 자연스럽습니다.

**언제 무엇을 쓰나**

`AWS ALB Controller`를 쓰는 경우:
1. EKS가 메인이고 AWS 기능을 적극 활용할 때
2. 서비스 수가 많지 않고 운영 단순성이 중요할 때
3. ACM TLS 종료, WAF, SG 연동이 중요할 때
4. 팀이 인프라 운영보다 애플리케이션 개발에 집중하고 싶을 때

`ingress-nginx`를 쓰는 경우:
1. 여러 서비스를 하나의 공용 ingress로 묶고 싶을 때
2. LB 비용을 줄이고 싶을 때
3. rewrite, custom routing, header 정책 등 세밀한 L7 제어가 필요할 때
4. AWS 종속성을 낮추고 싶거나 멀티 환경 이식성이 중요할 때

**이 프로젝트 기준 추천**

이 프로젝트는 이미 운영 설정이 `ALB` 중심으로 짜여 있어서, 지금 상태와 가장 자연스러운 선택은 `ALB 유지`입니다. 특히 EKS 실습/운영 흐름, ArgoCD, Grafana 노출 방식까지 ALB로 맞춰져 있어서 흐름이 일관됩니다.

반대로 앞으로 이 클러스터에 앱이 계속 늘어나고, 여러 서비스를 공용 ingress 한 곳으로 통합하려는 목적이 크다면 `ingress-nginx` 전환을 검토할 만합니다.

한 줄 추천:
1. `AWS 친화적이고 단순하게 운영`하려면 `ALB`
2. `여러 서비스 통합과 L7 제어력`이 중요하면 `ingress-nginx`

원하시면 다음 답변에서 이 표를 바탕으로 `면접 답변용 1분 설명` 형태로도 정리해드릴게요.
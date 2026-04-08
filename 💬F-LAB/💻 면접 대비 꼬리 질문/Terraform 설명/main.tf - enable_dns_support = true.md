### 면접용 답변
해당 옵션이 false로 설정된다면 통신이 불가능합니다. 인스턴스에는 public ip와 public hostname이 할당됩니다(옵션 지정 시). 우리는 통신을 할 때 최종적으로 ip를 사용합니다. 즉, 만약 hostname으로 통신을 했다면 hostname을 ip로 변환하는 친구가 있어야 한다는 것이죠. 이 역할을 하는 친구를 활성화 하는 옵션이 dns_support 입니다.


---
## `enable_dns_support = true`

VPC 내부에서 **AWS DNS 서버(Route 53 Resolver)를 사용할 수 있게** 허용하는 설정입니다.

VPC에는 기본 DNS 서버가 있습니다. 주소는 항상 **VPC CIDR의 두 번째 IP**입니다.

```
VPC CIDR: 10.0.0.0/16
→ AWS DNS 서버 주소: 10.0.0.2
```

---

## `false`일 때 vs `true`일 때

```
# enable_dns_support = false (비활성화)

EKS Worker Node (10.0.10.5)
  → "ecr.ap-northeast-2.amazonaws.com 의 IP가 뭐야?"
  → DNS 서버 없음
  → 응답 없음 → ECR에서 이미지 pull 실패
```

```
# enable_dns_support = true (활성화)

EKS Worker Node (10.0.10.5)
  → "ecr.ap-northeast-2.amazonaws.com 의 IP가 뭐야?"
  → 10.0.0.2 (AWS DNS) 에게 질의
  → "3.34.51.89 야" 응답
  → ECR pull 성공
```

---

## EKS에서 특히 중요한 이유

EKS 노드가 동작하려면 아래 도메인들을 DNS로 조회해야 합니다:

|조회 대상|용도|
|---|---|
|`ecr.ap-northeast-2.amazonaws.com`|이미지 pull|
|`eks.amazonaws.com`|Control Plane API 통신|
|`s3.amazonaws.com`|부트스트랩 스크립트 다운로드|
|`sts.amazonaws.com`|IRSA 토큰 교환|

`enable_dns_support = false`면 이 모든 통신이 막혀서 **노드가 클러스터에 조인조차 못합니다.**

---

## `enable_dns_hostnames`와의 차이

|설정|역할|
|---|---|
|`enable_dns_support`|VPC 안에서 DNS **조회** 가능 여부|
|`enable_dns_hostnames`|EC2에 퍼블릭 DNS 이름 **부여** 여부 (`ip-10-0-10-5.ap-northeast-2.compute.internal`)|

둘 다 `true`여야 `kubectl`로 노드 이름이 제대로 표시됩니다.
VPC와 서브넷은 AWS 네트워크를 이해할 때 가장 기본이 되는 개념이야.

- VPC = “내가 만드는 가상 네트워크 전체”
    
- Subnet = “그 네트워크를 더 작게 나눈 구역”
    

이라고 이해하면 돼.

## VPC (Virtual Private Cloud)

VPC는 AWS 안에서 사용하는 **논리적으로 분리된 가상 네트워크**야.  
즉, AWS 클라우드 안에 내가 직접 네트워크를 하나 만드는 개념이야. ([AWS Documentation](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Subnets.html?utm_source=chatgpt.com "How Amazon VPC works - Amazon Virtual Private Cloud"))

예를 들어:

- 회사 내부망: `10.0.0.0/16`
    
- 서버, DB, Kubernetes 노드 등을 이 네트워크 안에 배치
    

할 수 있어.

VPC에서 설정하는 대표 요소:

- IP 대역(CIDR)
    
- 서브넷
    
- 라우팅 테이블
    
- 인터넷 게이트웨이
    
- NAT 게이트웨이
    
- 보안 그룹
    

즉, AWS에서 인프라를 만들기 전에 먼저 “네트워크 공간”을 만드는 단계라고 보면 돼.

예시:

```text
VPC: 10.0.0.0/16
```

이 경우:

- 사용할 수 있는 IP 범위가 매우 큼
    
- 내부에서 여러 서브넷으로 나눌 수 있음
    

---

## Subnet (서브넷)

서브넷은 VPC를 더 작은 네트워크 단위로 나눈 거야.  
즉, VPC 내부의 IP 대역 일부를 잘라서 사용하는 개념이야. ([AWS Documentation](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Subnets.html?utm_source=chatgpt.com "How Amazon VPC works - Amazon Virtual Private Cloud"))

예시:

```text
VPC: 10.0.0.0/16

Subnet A: 10.0.1.0/24
Subnet B: 10.0.2.0/24
Subnet C: 10.0.3.0/24
```

이렇게 나눠서:

- 웹 서버 영역
    
- 애플리케이션 서버 영역
    
- DB 영역
    

등으로 분리할 수 있어.

---

## 왜 서브넷으로 나눌까?

주 목적은:

### 1. 보안 분리

예:

- Public Subnet → 인터넷 연결 가능
    
- Private Subnet → 외부 직접 접근 불가
    

보통 구조:

```text
Internet
   ↓
Public Subnet
   ↓
Private Subnet
```

- ALB / Bastion → Public
    
- DB / 내부 서비스 → Private
    

처럼 구성해.

AWS에서 subnet이 public/private인지 결정되는 핵심은  
“인터넷 게이트웨이로 가는 route가 있냐”야. ([AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html?utm_source=chatgpt.com "Subnets for your VPC - Amazon Virtual Private Cloud"))

---

### 2. 장애 분리 (AZ 분산)

Subnet은 반드시 하나의 AZ에만 속해. ([AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html?utm_source=chatgpt.com "Subnets for your VPC - Amazon Virtual Private Cloud"))

예:

```text
ap-northeast-2a → subnet-a
ap-northeast-2c → subnet-c
```

이렇게 여러 AZ에 분산해서 고가용성을 확보해.

> [!info] 여러 AZ에 서브넷을 분산해 고가용성을 확보한다는 의미  
> AWS의 AZ(Availability Zone)는 서로 독립된 데이터센터다.
> 
> 서브넷은 반드시 하나의 AZ에만 속할 수 있기 때문에:
> 
> ```text
> Subnet-A → ap-northeast-2a
> Subnet-C → ap-northeast-2c
> ```
> 
> 처럼 여러 AZ에 나누어 생성한다.
> 
> 이렇게 분산하는 이유는 특정 AZ 장애가 발생하더라도 서비스 전체가 중단되지 않도록 하기 위해서다.
> 
> 예를 들어:
> 
> ```text
> ALB
>  ├─ EC2 (2a subnet)
>  └─ EC2 (2c subnet)
> ```
> 
> 구조에서 `ap-northeast-2a` AZ가 장애나더라도:
> 
> - `2c` AZ의 EC2는 계속 동작
>     
> - ALB가 정상 인스턴스로 트래픽 전달 가능
>     
> - 전체 서비스는 계속 운영 가능
>     
> 
> 즉:
> 
> > 하나의 데이터센터 장애가 전체 서비스 장애로 이어지지 않도록 여러 AZ에 리소스를 분산하는 것
> 
> 이 고가용성(HA, High Availability) 확보의 핵심 개념이다.

---

## 쉽게 비유하면

```text
VPC = 아파트 단지 전체
Subnet = 단지 안의 각 동
EC2 = 각 집
IP = 집 주소
```

- VPC가 큰 네트워크 공간
    
- 서브넷이 구역
    
- EC2 같은 리소스가 실제 배치되는 위치
    

라고 생각하면 이해가 쉬워.

---

## 면접용 한 줄 정의

### VPC

> AWS 상에서 사용자가 정의하는 논리적으로 격리된 가상 네트워크입니다.

### Subnet

> VPC 내부 IP 대역을 더 작은 네트워크 단위로 분리한 영역이며, 리소스를 배치하고 보안 및 가용성을 분리하기 위해 사용합니다.
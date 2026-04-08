**OIDC**는 **인증을 위해 사용되는 신원 확인 방식**이야. 🔐  
정식 이름: **OpenID Connect**

## 한 줄 정의

👉 **“이 요청한 대상이 누구인지 증명해주는 인증 프로토콜”**

## EKS IRSA에서 역할

AWS가 Pod에게 물어봄:

```text
AWS → "너 진짜 이 클러스터 Pod 맞아?"
```

이걸 확인할 때 사용하는 게 **OIDC**.

## 흐름

```text
Pod
 → ServiceAccount 토큰(= OIDC 토큰 전달) 생성
 → AWS가 OIDC Provider로 검증
 → IAM Role 허용
```

## 왜 필요해?

IRSA 구조:

```text
Pod → ServiceAccount → IAM Role → AWS
```

여기서 AWS는 **ServiceAccount 토큰이 진짜인지 확인해야 함**  
→ 그 확인 시스템이 **OIDC** ✔️

## Terraform에서 나오는 이유

`iam_oidc.tf`

- EKS 클러스터 OIDC Provider 생성
    
- AWS가 신뢰할 대상 등록
    

## 핵심 요약

- OIDC = 신원 인증 방식
    
- AWS가 Pod 신뢰 여부 확인
    
- IRSA 필수 구성 요소
    
- ServiceAccount 토큰 검증에 사용
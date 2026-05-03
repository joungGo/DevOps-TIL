**EKS IRSA**는 **Pod가 AWS 권한을 안전하게 사용하는 방법**이야. 🔐

정식 이름: **IAM Roles for Service Accounts**

---

## 한 줄 정의

👉 **Kubernetes Pod에 IAM Role을 직접 연결하는 기능**

---

## 왜 필요해?

기본적으로 Pod는 AWS에 요청하면:

```
Pod → AWS 요청
AWS → "너 누구야?"
```

Pod는 IAM User도 아니고 Role도 아니라서 **권한이 없음** ❌

---

## IRSA 사용하면

```
Pod → ServiceAccount → IAM Role → AWS
```

즉,

- Pod가 특정 [[ServiceAccount]] 사용
    
- ServiceAccount에 IAM Role 연결
    
- 그 Role 권한으로 AWS 접근
    

올바른 정의: [[2. ServiceAccount와 IAM Role의 정의 & 관계]]

---

## 예시

S3 접근하는 Pod

IRSA 없을 때 ❌

- access key 하드코딩 (보안 위험)
    

IRSA 있을 때 ✅

- Pod에 Role만 연결
    
- 키 없음
    
- 자동 인증
    

---

## 구성 요소

1. **IAM Role**
    
2. **Kubernetes ServiceAccount**
    
3. **[[OIDC]] Provider**
    
4. **Pod**
    

---

## 흐름

```
Pod 실행
 → ServiceAccount 사용
 → IAM Role Assume
 → AWS API 호출
```

---

## 왜 OIDC 필요?

AWS가 Pod 신원을 확인해야 함

그래서:

```
EKS OIDC Provider 등록
 → AWS가 "이 Pod는 신뢰 가능" 확인
```

그래서 `iam_oidc.tf` 파일이 등장하는 거야.

---

## 핵심 요약

- IRSA = Pod에 IAM Role 연결
    
- access key 필요 없음
    
- 보안 안전
    
- OIDC Provider 필요
    


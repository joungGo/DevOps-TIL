Capability는 전통적인 **root(UID 0) 전체 권한 모델을 쪼개서, 권한을 세분화한 메커니즘**이다.

---

# 1. 등장 이전: 전통적인 권한 모델

과거 리눅스/유닉스는 매우 단순했다.

## 구조

- root (UID 0) → 모든 권한
    
- 일반 사용자 → 제한된 권한
    

즉,

```text
권한 = root 이냐 아니냐
```

### 문제점

1. root는 너무 강력함  
    → 네트워크 설정, 마운트, 모듈 로드, 모든 파일 접근 가능
    
2. setuid root 프로그램의 위험성  
    → 일부 기능만 필요해도 root 전체 권한 획득
    

예:

- ping은 raw socket 생성 때문에 root 필요
    
- 하지만 root 권한 전체를 부여해야 했음
    

이건 보안상 매우 위험했다.

---

# 2. 등장 이후: Capability 기반 모델

리눅스 2.2부터 도입.

핵심 아이디어:

> root 권한을 여러 개의 작은 권한 단위로 분해

예:

- CAP_NET_ADMIN → 네트워크 설정
    
- CAP_NET_RAW → raw socket 사용
    
- CAP_SYS_ADMIN → 시스템 관리 작업
    
- CAP_SYS_TIME → 시스템 시간 변경
    

이제는:

```text
권한 = 필요한 capability만 부여
```

가능해졌다.

---

# 3. 무엇이 달라졌나?

## 이전

- root → 전능
    
- 일반 사용자 → 제한
    
- 중간 단계 없음
    

## 이후

- 프로세스는 특정 capability만 가질 수 있음
    
- root 없이도 특정 특권 작업 가능
    
- 최소 권한 원칙 적용 가능
    

예:

ping에 CAP_NET_RAW만 부여하면  
root 전체 권한 없이도 실행 가능.

---

# 4. Capability의 구조

각 프로세스는 다음 집합을 가진다:

- Permitted
    
- Effective
    
- Inheritable
    
- Bounding
    
- Ambient
    

핵심은:

- Effective에 있는 capability만 실제로 사용됨
    

---

# 5. 왜 중요한가?

## 보안 강화

- 침해되더라도 전체 root 권한 탈취 아님
    
- 공격 표면 축소
    

## 컨테이너 환경에서 필수

Docker는 기본적으로 많은 capability를 제거한다.

예:

- 컨테이너 안의 root는 진짜 root가 아님
    
- 일부 capability만 가진 제한된 root
    

---

# 6. 요약 비교

|구분|기존 모델|Capability 모델|
|---|---|---|
|권한 단위|root 전체|세분화된 권한|
|보안성|취약|향상|
|최소 권한 원칙|어려움|가능|
|컨테이너 친화성|낮음|높음|

---

# 한 줄 정리

Capability는 기존의 “root는 모든 권한”이라는 단일 모델을 세분화해, 필요한 권한만 프로세스에 부여할 수 있게 만든 리눅스의 권한 분리 메커니즘이다.
**systemd**는 현대 리눅스에서 사용하는 기본 **init 시스템(PID 1)** 이자 서비스 관리자이다.

대표 배포판 대부분에서 사용한다.

---

# 1. 왜 등장했나

과거에는:

- SysV init 스크립트 기반
    
- 순차적 실행
    
- 의존성 관리가 복잡
    

문제점:

- 부팅 느림
    
- 서비스 간 의존성 처리 어려움
    
- 관리 구조가 단순하지만 확장성 낮음
    

이 문제를 해결하기 위해 systemd가 등장했다.

---

# 2. systemd의 핵심 역할

## 1) PID 1 프로세스

커널이 부팅 후 실행하는 첫 사용자 공간 프로세스.

```text
커널 → systemd (PID 1)
```

모든 프로세스의 최상위 부모 역할 수행.

---

## 2) 서비스 관리

서비스를 “unit” 단위로 관리한다.

예:

- nginx.service
    
- sshd.service
    

명령어:

```bash
systemctl start nginx
systemctl stop nginx
systemctl status nginx
```

---

## 3) 병렬 부팅

- 의존성 기반 실행
    
- 가능한 서비스는 동시에 시작
    
- 부팅 속도 향상
    

---

## 4) 의존성 관리

서비스 간 관계 정의 가능:

- Requires=
    
- After=
    
- Wants=
    

즉, 실행 순서를 명확히 제어할 수 있다.

---

# 3. Unit 종류

systemd는 여러 종류의 unit을 관리한다.

- service → 데몬
    
- socket → 소켓 활성화
    
- target → 실행 그룹
    
- mount → 마운트
    
- timer → cron 대체
    

---

# 4. 추가 기능

- 로그 관리 (journald)
    
- cgroup 기반 리소스 관리
    
- 자동 재시작
    
- 소켓 활성화
    
- 타이머 기능
    

단순 init 시스템을 넘어선 “시스템 관리 플랫폼”에 가깝다.

---

# 5. SysV init과 비교

|항목|SysV init|systemd|
|---|---|---|
|실행 방식|순차적|병렬|
|의존성 관리|제한적|명시적|
|서비스 관리|스크립트|unit 파일|
|기능 범위|init 중심|통합 시스템 관리|

---

# 핵심 구조 흐름

```text
커널 부팅
→ systemd (PID 1)
→ 서비스 로드
→ 로그인 서비스 시작
→ 사용자 세션
```

---

# 한 줄 정리

systemd는 커널 이후 실행되는 PID 1 프로세스로, 서비스 관리·부팅 제어·의존성 처리·리소스 관리 등을 담당하는 현대 리눅스의 통합 init 시스템이다.
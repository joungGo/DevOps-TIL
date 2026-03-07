## Roll In / Roll Out

### 1. Roll Out

**새로운 애플리케이션 버전을 배포하는 과정**

즉

```text
기존 Pod → 새로운 Pod로 점진적으로 교체
```

예

```text
nginx:1.0 → nginx:2.0 업데이트
```

Deployment가 수행하는 작업

```text
새로운 Pod 생성
기존 Pod 제거
```

이 과정을 **Rolling Update**라고 한다.

---

### 2. Roll In

쿠버네티스 공식 용어는 아니다.

보통 현장에서 말하는 **Roll In**은 다음 의미로 사용된다.

```text
새로운 버전의 Pod가 점점 트래픽을 받기 시작하는 과정
```

즉

```text
새 Pod → 서비스에 편입
```

---

## 실제 배포 흐름

예

```text
replicas = 3
```

기존 상태

```text
Pod v1
Pod v1
Pod v1
```

Roll Out 시작

```text
Pod v2 생성
Pod v1 제거
```

과정

```text
Pod v2
Pod v1
Pod v1
```

→

```text
Pod v2
Pod v2
Pod v1
```

→

```text
Pod v2
Pod v2
Pod v2
```

---

## 핵심 개념

|용어|의미|
|---|---|
|Roll Out|새로운 버전 배포|
|Roll In|새로운 Pod가 서비스에 편입|

---

## 면접용 한 줄

```text
Roll Out은 새로운 애플리케이션 버전을 점진적으로 배포하는 과정이고,
Roll In은 새로 생성된 Pod가 서비스 트래픽을 받기 시작하는 과정이다.
```
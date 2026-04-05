# ✅ 완전한 최적 Dockerfile (권장)

```dockerfile
# syntax=docker/dockerfile:1.6

# -----------------------------
# Stage 1: Builder
# -----------------------------
FROM python:3.11-slim AS builder

# 작업 디렉토리
WORKDIR /build

# 의존성 파일 먼저 복사 (레이어 캐시 최적화)
COPY requirements.txt .

# pip 패키지 설치
# --mount=type=cache : BuildKit 캐시 사용
# --target=/install  : 패키지를 별도 디렉토리에 설치
# --no-cache-dir     : pip 캐시를 이미지에 남기지 않음
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir --target=/install -r requirements.txt


# -----------------------------
# Stage 2: Runtime
# -----------------------------
FROM python:3.11-slim

# 작업 디렉토리
WORKDIR /app

# builder stage에서 설치된 패키지 복사
COPY --from=builder /install /usr/local

# 애플리케이션 코드 복사
COPY ./app ./app

# 비루트 사용자 생성 (보안)
RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

# 포트 문서화
EXPOSE 8000

# 실행 명령
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

# 1️⃣ `RUN --mount=type=cache` 완전 설명

```dockerfile
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir --target=/install -r requirements.txt
```

이 한 줄은 **3가지 목적**을 동시에 수행합니다:

### ① BuildKit 캐시 사용

```
--mount=type=cache,target=/root/.cache/pip
```

의미:

- pip 다운로드 파일을 빌드 캐시에 저장
    
- 다음 빌드 시 재사용
    

빌드 흐름:

첫 빌드

```
requirements.txt
      ↓
pip download (인터넷)
      ↓
캐시 저장
```

두 번째 빌드

```
requirements.txt 동일
      ↓
캐시에서 바로 사용
      ↓
빠른 빌드
```

👉 이미지 크기 증가 없음  
👉 빌드 속도만 빨라짐

---

### ② pip 캐시 제거

```
--no-cache-dir
```

pip 내부 캐시를 이미지에 남기지 않음

없으면:

```
/root/.cache/pip
   ├── wheel
   ├── http
   └── ...
```

→ 이미지 커짐 ❌

---

### ③ 별도 설치 디렉토리 사용

```
--target=/install
```

기본 pip 설치 위치:

```
/usr/local/lib/python3.11/site-packages
```

하지만 우리는:

```
/install/lib/python3.11/site-packages
```

여기에 설치

이유:  
👉 builder stage에서만 설치  
👉 runtime stage로 필요한 것만 복사

---

# 2️⃣ Dockerfile 경로들 실제 의미

## `/build`

```
WORKDIR /build
```

컨테이너 내부 경로

실제 구조:

```
container filesystem
└── /build
    └── requirements.txt
```

호스트와 무관  
이미지 내부에 저장됨

---

## `/install`

pip 패키지 임시 저장 위치

builder stage 내부

```
/install
└── lib
    └── python3.11
        └── site-packages
```

runtime stage에서:

```
COPY --from=builder /install /usr/local
```

결과:

```
/usr/local/lib/python3.11/site-packages
```

Python이 자동 인식 👍

---

## `/usr/local`

Python 기본 패키지 경로

Python sys.path:

```
/usr/local/lib/python3.11/site-packages
```

그래서 여기로 복사하는 것

---

## `/root/.cache/pip`

pip 다운로드 캐시 위치

BuildKit cache mount 대상

이미지에는 포함되지 않음

---

## `/app`

애플리케이션 코드 위치

```
WORKDIR /app
COPY ./app ./app
```

결과:

```
/app/app/main.py
```

---

# 3️⃣ `AS builder` 의미

```
FROM python:3.11-slim AS builder
```

이건 **stage 이름 지정**

멀티스테이지 빌드 구조:

```
Stage 1 → builder
Stage 2 → runtime
```

builder에서 설치  
runtime에서 복사

```
COPY --from=builder /install /usr/local
```

👉 builder stage 파일만 가져옴

---

# 4️⃣ 멀티스테이지 빌드 구조

```
builder stage
 ├── python
 ├── pip
 ├── build tools
 └── dependencies

          ↓ 필요한 것만 복사

runtime stage
 ├── python
 └── dependencies only
```

결과:

이미지 크기 ↓  
보안 ↑  
배포 속도 ↑

---

# 5️⃣ 이미지 레이어 구조

builder image

```
Layer 1 python:3.11
Layer 2 WORKDIR
Layer 3 COPY requirements
Layer 4 pip install
```

runtime image

```
Layer 1 python:3.11-slim
Layer 2 COPY dependencies
Layer 3 COPY app
Layer 4 USER
```

builder 레이어는 최종 이미지에 포함되지 않음

---

# 6️⃣ BuildKit 활성화

Dockerfile 첫 줄

```
# syntax=docker/dockerfile:1.6
```

또는

```
DOCKER_BUILDKIT=1 docker build .
```

---

# 7️⃣ Terraform + EKS + ECR 관점 장점

이 Dockerfile 사용 시

### 이미지 크기 감소

예시

일반 pip install

```
1.2GB
```

멀티스테이지

```
280MB
```

---

### 배포 속도 증가

EKS pull 시간 감소

```
kubectl rollout faster
```

---

### CI/CD 비용 감소

Build cache 활용

```
빌드 2분 → 20초
```

---

### 보안 향상

- build tool 제거
    
- root 사용자 제거
    
- 최소 runtime image
    

---

# 8️⃣ 최종 구조 시각화

```
Dockerfile
   │
   ├── builder stage
   │     └── pip install → /install
   │
   └── runtime stage
         ├── copy /install → /usr/local
         ├── copy app
         └── run uvicorn
```

---

# 9️⃣ 핵심 요약

### `--mount=type=cache`

👉 pip 다운로드 캐시 재사용 (빌드 빠름)

### `--target=/install`

👉 패키지를 별도 디렉토리에 설치

### `/install`

👉 builder 임시 설치 경로

### `/usr/local`

👉 python 기본 패키지 위치

### `AS builder`

👉 멀티스테이지 빌드 이름

### 멀티스테이지

👉 작은 이미지 + 빠른 배포

---

이 Dockerfile은  
✔ Terraform  
✔ EKS  
✔ ECR  
✔ FastAPI  
✔ CI/CD

환경에서 **실무 최적 구조**입니다.

---

원하시면 다음도 바로 만들어 드립니다:

- Terraform ECR 코드
    
- Terraform EKS deployment
    
- GitHub Actions CI
    
- Docker image build & push pipeline
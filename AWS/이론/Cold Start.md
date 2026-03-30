**Cold start**는
👉 **서비스나 인스턴스가 “처음 시작될 때” 발생하는 지연 시간**을 의미해 ⏱️

즉, 이미 실행 중이면 빠른데
**꺼져 있다가 새로 시작할 때 느려지는 현상**이야.

---

# 1️⃣ 가장 쉬운 예

서버가 이미 켜져 있을 때

```
요청 → 바로 응답 (빠름)
```

서버가 꺼져 있을 때

```
요청 → 서버 생성 → 앱 실행 → 응답 (느림)
```

👉 이 느린 첫 요청 = cold start

---

# 2️⃣ Kubernetes에서 Cold Start

Kubernetes 에서:

1. Pod 없음
2. 요청 들어옴
3. 새 Pod 생성
4. 이미지 pull
5. 컨테이너 실행

→ 이 과정이 느림

---

# 3️⃣ Spot 인스턴스와 관계

Spot 회수됨 → 노드 사라짐 → 새 노드 생성 → pod 재배치

이때:

* EC2 생성
* kubelet join
* pod scheduling
* container start

👉 cold start 발생

---

# 4️⃣ 어디서 많이 발생하나

### 서버리스

AWS Lambda 대표적

* 일정 시간 요청 없음
* 컨테이너 종료됨
* 다시 요청
  → cold start

---

### Kubernetes autoscaling

Horizontal Pod Autoscaler

* 트래픽 증가
* 새 pod 생성
* 초기 지연

---

# 5️⃣ Cold Start 발생 원인

* EC2 생성 시간
* Docker image pull
* 앱 부팅 시간
* JVM startup (특히 느림)
* dependency loading

---

# 6️⃣ Cold Start 줄이는 방법

✔ 최소 노드 유지 (on-demand)
✔ 최소 pod 유지 (`min_replicas`)
✔ 작은 이미지 사용
✔ 이미지 미리 pull
✔ warm pool 사용

---

# 7️⃣ 예시 (Spot만 쓰면)

```
노드 없음
요청 발생
→ 노드 생성 (1~2분)
→ pod 생성
→ 응답
```

사용자 체감:
👉 처음 요청 매우 느림

---

# 핵심 한 줄

👉 **꺼져있던 리소스가 처음 켜질 때 발생하는 지연 시간 = cold start**

---

지금 Spot 쓰려는 이유가

* 비용 절감?
* autoscaling?
* 실습?

어떤 상황인지 알려주면 cold start 영향도 같이 설명해 줄게.

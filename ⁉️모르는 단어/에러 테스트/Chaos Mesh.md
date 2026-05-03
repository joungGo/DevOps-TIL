Chaos Mesh는 Kubernetes 환경에서 카오스 엔지니어링(Chaos Engineering)을 수행하기 위한 오픈소스 플랫폼이야.

쉽게 말하면:  
“Kubernetes 클러스터에 의도적으로 장애를 발생시키는 도구”야.

주로 이런 목적에 사용돼:

- 장애 대응 훈련
    
- 시스템 복원력(Resilience) 검증
    
- Auto Healing 테스트
    
- Alert 검증
    
- AI RCA 데이터 생성
    

---

## 왜 사용할까?

운영 환경에서는 언젠가 반드시 장애가 발생해.

그래서:

- Node 죽으면 어떻게 되는지
    
- 네트워크가 느려지면 어떻게 되는지
    
- DB 연결이 끊기면 어떻게 되는지
    

를 미리 실험해보는 거야.

Chaos Mesh는 Kubernetes-native 방식으로 이런 실험을 자동화해줘.

---

# 주요 기능

## 1. Pod 장애

예:

- Pod kill
    
- Pod restart
    
- Container stop
    

```yaml
kind: PodChaos
spec:
  action: pod-kill
```

---

## 2. Network 장애

예:

- latency 추가
    
- packet loss
    
- network partition
    

```yaml
kind: NetworkChaos
```

예:

- API latency 3초 추가
    
- 패킷 30% 손실
    

---

## 3. CPU / Memory 부하

```yaml
kind: StressChaos
```

예:

- CPU 100% 사용
    
- 메모리 고갈
    

---

## 4. Node 장애

```yaml
kind: DNSChaos
kind: IOChaos
```

예:

- 디스크 지연
    
- DNS 실패
    
- 파일 시스템 오류
    

---

# 구조

Kubernetes CRD 기반으로 동작해.

즉:

```text
kubectl apply -f chaos.yaml
```

하면 Chaos Mesh controller가 장애를 주입해.

구조 예시:

```text
User
  ↓
Chaos YAML
  ↓
Chaos Mesh Controller
  ↓
Target Pod / Node
```

---

# 예시 시나리오

예:  
“comment-service pod를 5분마다 랜덤 삭제”

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: comment-pod-kill
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces:
      - default
    labelSelectors:
      app: comment-service
```

---

# 네 프로젝트와 연결하면

네가 만들려는 AI Incident Investigation Platform에서는 아주 잘 맞는 도구야.

흐름 예시:

```text
Chaos Mesh
  ↓
장애 발생
  ↓
Prometheus Alert
  ↓
Loki / Metrics / Events 수집
  ↓
AI RCA 분석
  ↓
Incident Timeline 생성
  ↓
RCA 평가
```

즉 Chaos Mesh는:  
“학습용/평가용 인시던트를 자동 생성하는 역할”을 하게 돼.

---

# 다른 카오스 도구와 비교

|도구|특징|
|---|---|
|Chaos Mesh|Kubernetes-native, CRD 기반|
|LitmusChaos|Workflow 중심|
|Gremlin|상용 SaaS|
|PowerfulSeal|Python 기반|

---

# 장점

- Kubernetes 친화적
    
- YAML 기반
    
- GitOps 연동 쉬움
    
- 다양한 장애 유형 지원
    
- 실제 운영 환경과 유사한 실험 가능
    

---

# 주의점

운영 환경에서 잘못 사용하면 진짜 장애가 될 수 있어.

그래서 보통:

- staging
    
- test cluster
    
- 제한된 namespace
    

에서 먼저 수행해.
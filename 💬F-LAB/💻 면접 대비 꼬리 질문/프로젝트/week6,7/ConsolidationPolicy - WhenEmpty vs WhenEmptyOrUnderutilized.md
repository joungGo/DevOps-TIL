## `WhenEmpty`로 변경한 이유

### 핵심 차이

| 정책                         | 동작                                                   |
| -------------------------- | ---------------------------------------------------- |
| `WhenEmptyOrUnderutilized` | 노드가 비거나, **통합 가능하다고 판단되면 Pod를 강제 이동(evict)** 후 노드 삭제 |
| `WhenEmpty`                | **노드에 Pod가 하나도 없을 때만** 삭제. Pod 강제 이동 없음              |

---

### 이 설정에서 `WhenEmptyOrUnderutilized`가 문제가 되는 이유

**1. Spot + consolidation 이중 중단**

```
values: ["spot", "on-demand"]   # Spot 허용consolidateAfter: 30s           # 30초 뒤 통합 시도
```

Spot 인스턴스는 AWS가 언제든 회수할 수 있습니다. 여기에 `WhenEmptyOrUnderutilized`까지 더해지면 Pod가 두 가지 원인(Spot 중단 + Karpenter eviction)으로 예고 없이 재시작될 수 있어 안정성이 크게 떨어집니다.

**2. PostgreSQL StatefulSet과 충돌**

`WhenEmptyOrUnderutilized`는 통합을 위해 노드의 Pod를 전부 evict하려 시도합니다. StatefulSet(PostgreSQL)은 PVC가 연결되어 있어 eviction이 느리거나 실패하고, 그 사이 DB 연결이 끊겨 앱 전체가 오류를 냅니다.

**3. HPA와의 race condition**

```
hpa:  minReplicas: 2  maxReplicas: 10
```

Karpenter가 "Pod를 줄여서 노드 통합 가능"이라고 판단해 evict → HPA가 부하 감지해서 다시 스케일업 → Karpenter가 다시 노드를 띄우는 무한루프가 발생할 수 있습니다.

**4. `consolidateAfter: 30s`가 너무 짧다**

운영 환경에서 `WhenEmptyOrUnderutilized` + `30s` 조합은 매우 공격적입니다. 조금이라도 idle해 보이면 30초 만에 Pod를 옮기려 시도합니다. 실습 중 갑자기 Pod가 재시작되면 원인 파악이 어렵습니다.

---

### 정리

```
WhenEmpty = "빈 노드만 삭제" → 실행 중인 Pod에 절대 영향 없음           실습/Spot 혼합 환경에서 안전한 선택
```

`WhenEmptyOrUnderutilized`는 **단일 On-demand, 긴 consolidateAfter(5~10분), PDB 설정 완비** 환경에서 비용 최적화 목적으로 쓰는 게 적합합니다. 이 실습 환경처럼 Spot + StatefulSet + 짧은 consolidateAfter 조합에서는 `WhenEmpty`가 맞습니다.

---

> [!info] Karpenter eviction의 정확한 의미  
> Karpenter의 eviction은 "강제 종료"가 아니라 **Kubernetes Eviction API 기반의 정상적인 재스케줄링 과정**이다.
> 
> **동작 순서**
> 
> 1. Karpenter가 "A 노드의 Pod들을 B 노드에 수용 가능" 판단
>     
> 2. A 노드의 Pod들에 eviction 요청 전송
>     
> 3. Pod들이 B 노드에서 새로 기동 (재스케줄링)
>     
> 4. A 노드가 비면 EC2 terminate
>     
> 
> **핵심 개념**
> 
> - Pod 입장: 종료 → 다른 노드에서 재시작
>     
> - 컨테이너: 반드시 재시작됨
>     
> - 의미: **정중한 이사 요청 (graceful disruption)**
>     

---

> [!warning] 특정 Pod를 통합 대상에서 제외하는 방법  
> Karpenter consolidation 대상에서 Pod를 제외하는 방법은 3가지가 있다.
> 
> **방법 1: karpenter.sh/do-not-disrupt (권장)**
> 
> ```yaml
> metadata:
>   annotations:
>     karpenter.sh/do-not-disrupt: "true"
> ```
> 
> - Karpenter가 절대 eviction하지 않음
>     
> - StatefulSet / DB Pod에 필수
>     
> 
> **방법 2: PodDisruptionBudget (PDB)**
> 
> ```yaml
> apiVersion: policy/v1
> kind: PodDisruptionBudget
> metadata:
>   name: postgres-pdb
> spec:
>   minAvailable: 1
>   selector:
>     matchLabels:
>       app: postgres
> ```
> 
> - 최소 가용 Pod 수 보장
>     
> - replicas=1이면 eviction 불가능
>     
> 
> **방법 3: consolidationPolicy 제어**
> 
> ```yaml
> consolidationPolicy: WhenEmpty
> ```
> 
> - Underutilized consolidation 자체 비활성화
>     
> 
> **Terraform 관점**
> 
> - annotation/PDB = "disruption 정책 정의"
>     
> - consolidationPolicy = "노드 최적화 전략 제어"
>     

---

> [!danger] HPA와 Karpenter 충돌 (oscillation 문제)  
> Karpenter는 "부하"가 아니라 **Pod 수 + resource request 기반으로 판단**한다.
> 
> **충돌 시나리오**
> 
> 1. 트래픽 감소 → HPA: 10 → 3 축소
>     
> 2. 노드 줄이기 위해 Karpenter consolidation 시작
>     
> 3. eviction 진행 중 트래픽 증가
>     
> 4. HPA: 3 → 8 확장
>     
> 5. 노드 부족 → Karpenter 신규 Node 생성
>     
> 6. 다시 트래픽 감소 → 반복
>     
> 
> **문제 본질**
> 
> - consolidation + scale 이벤트가 짧은 시간에 교차
>     
> - 결과: **노드 생성/삭제 반복 (oscillation)**
>     
> 
> **정정**
> 
> - "무한루프" ❌
>     
> - "비효율적인 진동" ⭕
>     

---

> [!tip] consolidateAfter 시간을 늘리면?  
> consolidateAfter는 **조건이 일정 시간 유지될 때만 consolidation 실행**하는 안정화 장치다.
> 
> ```yaml
> consolidationPolicy: WhenEmptyOrUnderutilized
> consolidateAfter: 5m
> ```
> 
> **동작 방식**
> 
> - 조건 유지 → 타이머 진행
>     
> - 조건 깨짐 → 타이머 리셋
>     
> 
> **효과**
> 
> |문제|30s → 5m 변경 효과|
> |---|---|
> |HPA oscillation|✅ 크게 완화|
> |StatefulSet eviction|❌ 영향 없음|
> |Spot interruption|⚠️ 일부 완화|
> 
> **실전 권장**
> 
> - consolidateAfter: 5m
>     
> - - DB Pod에 do-not-disrupt annotation
>         
> 
> → WhenEmptyOrUnderutilized 유지 가능

---

> [!summary] Pod/Node 정리 조건 (Disruption 판단 기준)  
> Karpenter가 노드를 "삭제 가능"으로 판단하는 조건은 다음과 같다.
> 
> **통합 가능 조건**
> 
> 1. 모든 Pod를 다른 노드에 수용 가능
>     
> 2. eviction 불가 Pod 없음
>     
>     - do-not-disrupt annotation → 불가
>         
>     - PDB 제한 → 불가
>         
>     - DaemonSet → 애초에 제외
>         
> 3. 해당 상태가 consolidateAfter 동안 유지
>     
> 
> **빈 노드 조건 (WhenEmpty)**
> 
> - DaemonSet Pod만 남은 경우  
>     → 빈 노드로 간주
>     
> 
> **정리**
> 
> ```
> WhenEmpty:
>   일반 Pod = 0 → 삭제
> 
> WhenEmptyOrUnderutilized:
>   위 조건 OR 통합 가능 → eviction 후 삭제
> ```
> 
> **핵심**
> 
> - DaemonSet은 계산에서 제외
>     
> - 판단 기준 = "request 기반 스케줄링 가능 여부"
>     

---

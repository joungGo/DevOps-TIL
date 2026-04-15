## 1. `consolidateAfter: 30s` 동작 원리

### 타이머 동작 방식

```
[상황: Pod가 줄어들어 Node A가 비거나 통합 가능해짐]

T+0s  : Karpenter가 조건 충족 감지
         "Node A의 Pod를 Node B로 옮길 수 있다"
  │
  ▼
T+0~30s : 대기 (관찰)
  │
  ├─ T+15s에 새 Pod가 Node A에 스케줄됨
  │    → 타이머 리셋 → 다시 T+0s부터
  │
  └─ 30초 동안 변화 없음
       → Node A의 Pod evict
       → Node B에서 재스케줄링
       → Node A 종료
```

### 왜 즉시 삭제하지 않나

```
롤링 업데이트 중 발생하는 일시적 상태:

T+0s : 구버전 Pod 종료 → Node A에 공간 생김
T+5s : Karpenter: "Node A 통합 가능" 감지
T+8s : 신버전 Pod 생성 → Node A에 배치
T+30s: 타이머 리셋 → 삭제 안 함 ✅

만약 즉시 삭제했다면:
T+0s : Pod 종료 → 즉시 Node A 삭제
T+5s : 신버전 Pod 생성 → 배치할 노드 없음 → Karpenter가 새 노드 생성
       → 불필요한 노드 삭제 + 재생성 반복
```

---

## 2. 삭제가 안 되는 예외 상황

### 예외 1: DaemonSet

```yaml
# DaemonSet은 모든 노드에 1개씩 강제 배치
# 예: kube-proxy, aws-node, node-exporter

Node A 상태:
  kube-proxy (DaemonSet)  ← 항상 존재
  aws-node   (DaemonSet)  ← 항상 존재
  사용자 Pod: 0개

→ WhenEmpty 판단 시 DaemonSet은 제외
→ 사용자 Pod 0개 = Empty 조건 충족
→ 삭제 가능 ✅

단, DaemonSet 외에 시스템 Pod가 있으면:
  Node A: [kube-proxy, aws-node, coredns]
                                  ↑
                         DaemonSet 아님, 일반 Pod
  → Empty 조건 불충족 → 삭제 안 됨
```

### 예외 2: PodDisruptionBudget (PDB)

PDB는 **"이 서비스는 최소 N개의 Pod가 항상 살아있어야 한다"** 는 제약이야.

```yaml
# PDB 예시
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: argocd-server-pdb
spec:
  minAvailable: 1        # 최소 1개는 항상 Running
  selector:
    matchLabels:
      app: argocd-server
```

```
[Karpenter가 Node B의 ArgoCD Pod를 evict 시도]

ArgoCD Pod 현재: 1개 Running (Node B)
PDB 조건: minAvailable: 1

Karpenter: "Node B의 Pod를 evict하려면 ArgoCD가 0개가 됨"
           → PDB 위반 → evict 거부
           → Node B 삭제 불가
```

**PDB가 자동 생성되는 컴포넌트들**:

|컴포넌트|PDB 설정 여부|
|---|---|
|ArgoCD|✅ Helm 설치 시 자동 생성|
|Prometheus|✅ kube-prometheus-stack 자동 생성|
|CoreDNS|✅ EKS 관리형 자동 생성|
|사용자 앱|❌ 직접 설정 필요|

---

## 3. 실무에서의 `consolidateAfter` 설정 전략

### 환경별 권장값

```yaml
# 개발 환경 — 빠른 비용 절감 우선
disruption:
  consolidationPolicy: WhenEmptyOrUnderutilized
  consolidateAfter: 30s     # 빠른 삭제, 비용 최소화

# 스테이징 환경 — 중간
disruption:
  consolidationPolicy: WhenEmptyOrUnderutilized
  consolidateAfter: 2m      # 2분, 안정성과 비용의 균형

# 프로덕션 환경 — 안정성 우선
disruption:
  consolidationPolicy: WhenEmpty   # Underutilized는 끄고 Empty만
  consolidateAfter: 5m             # 5분, 충분한 관찰 후 삭제
```

### 실무에서 `WhenUnderutilized`를 끄는 이유

```
WhenUnderutilized = Pod를 강제로 다른 노드로 이동

이동 중 발생하는 일:
  1. 기존 Pod Termination (graceful shutdown: 최대 30초)
  2. 새 노드에서 Pod 시작 (이미지 pull + 기동: 10~30초)
  3. 헬스체크 통과까지 대기

이 40~60초 동안:
  - 해당 Pod로 가는 트래픽 처리 용량 감소
  - 순간적인 응답 지연 발생 가능

→ 프로덕션에서는 비용 절감보다 안정성이 우선
→ WhenEmpty만 사용 (이미 비어있는 노드만 삭제)
```

### `budgets`으로 삭제 속도 제한 (실무 패턴)

```yaml
# 한 번에 너무 많은 노드가 삭제되지 않도록 제한
disruption:
  consolidationPolicy: WhenEmptyOrUnderutilized
  consolidateAfter: 30s
  budgets:
    - nodes: "20%"    # 전체 노드의 20%까지만 동시 삭제
```

```
노드 10개 운영 중 → 한 번에 최대 2개만 삭제
→ 갑작스러운 용량 감소 방지
→ 트래픽 급증 시 버퍼 역할
```

### 점검 시간대에만 Consolidation 허용

```yaml
# 트래픽이 적은 새벽에만 노드 정리
disruption:
  budgets:
    - schedule: "0 2 * * *"    # 새벽 2시에만
      duration: 2h              # 2시간 동안
      nodes: "50%"              # 최대 50%까지
    - nodes: "0%"               # 그 외 시간대는 삭제 금지
```

---

## 4. 현재 프로젝트 설정 평가

```yaml
# 현재 nodepool.yaml
disruption:
  consolidationPolicy: WhenEmptyOrUnderutilized
  consolidateAfter: 30s
```

|항목|현재 설정|프로덕션 권장|
|---|---|---|
|Policy|WhenEmptyOrUnderutilized|WhenEmpty|
|consolidateAfter|30s|5m|
|budgets|없음|`nodes: "20%"`|

**현재 설정은 실습/개발 환경에 최적화**된 값이야. 비용 절감을 최대한 빠르게 확인하기 좋지만, 실제 서비스라면 Pod 이동으로 인한 순간 지연이 발생할 수 있어.

> 실무 요약:
> 
> - **개발/스테이징**: `WhenEmptyOrUnderutilized` + 짧은 시간 → 비용 최소화
> - **프로덕션**: `WhenEmpty` + 긴 시간 + `budgets` → 안정성 우선, 점진적 삭제
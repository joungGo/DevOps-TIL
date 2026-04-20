**PodDisruptionBudget**의 약자입니다.

---

## 한 줄 정의

> "이 Pod는 동시에 최대 N개까지만 중단을 허용한다"는 **중단 허용량 제한 정책**

---

## 왜 필요한가

Kubernetes에서 Pod를 "정상적으로" 중단시키는 상황이 여러 가지 있습니다.

```
- kubectl drain (노드 점검)- Karpenter consolidation (노드 통합)- 롤링 업데이트 (Deployment 배포)- HPA scale down- 노드 업그레이드
```

이 중단들이 **동시에 여러 Pod에 발생하면** 서비스가 순간적으로 끊길 수 있습니다. PDB는 이를 막는 안전장치입니다.

---

## 설정 방법

두 가지 방식 중 하나를 씁니다.

```
# 방식 1: minAvailable — 최소 N개는 항상 살아있어야 함spec:  minAvailable: 1        # replica 3개 중 1개는 항상 유지                         # → 동시에 최대 2개까지 중단 허용
# 방식 2: maxUnavailable — 동시에 최대 N개까지 중단 허용spec:  maxUnavailable: 1      # 동시에 1개만 중단 허용
```

---

## 실제 동작 예시

```
replicas: 3, minAvailable: 2 설정 시Karpenter가 노드 통합하려고 Pod eviction 요청  → 현재 Running Pod = 3개  → 1개 evict 허용 (3 - 1 = 2 = minAvailable 충족)  → 2번째 evict 요청 → 현재 Running = 2개 → 거부 (minAvailable 위반)  → Karpenter는 2번째 Pod가 다른 노드에서 뜰 때까지 대기
```

---

## Karpenter와의 관계

Karpenter는 eviction 전에 PDB를 반드시 확인합니다.

```
PDB 위반 없이 evict 가능 → 통합 진행PDB 위반 발생           → 해당 노드 통합 포기, 다음 노드 탐색
```

그래서 `replicas: 1`인 StatefulSet에 `minAvailable: 1` PDB를 걸면 Karpenter가 그 노드를 **절대 통합 대상으로 선택하지 못합니다.**

---

## 실습 환경에서 적용한다면

```
apiVersion: policy/v1kind: PodDisruptionBudgetmetadata:  name: url-shortener-pdb  namespace: url-shortenerspec:  minAvailable: 1  selector:    matchLabels:      app: url-shortener-api   # HPA로 관리되는 API Pod
```

`minReplicas: 2`이고 `minAvailable: 1`이면 — Karpenter가 동시에 1개씩만 evict하므로 서비스 중단 없이 노드 통합이 가능합니다.
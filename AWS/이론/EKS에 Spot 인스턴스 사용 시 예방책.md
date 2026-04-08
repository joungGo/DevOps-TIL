**1. PodDisruptionBudget (PDB) — 동시 축출 수 제한**

Spot 노드가 회수될 때 한 번에 너무 많은 Pod가 On-Demand로 밀려오지 않도록 제한합니다.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: url-shortener-pdb
spec:
  minAvailable: 2        # 최소 2개는 항상 Running 유지
  selector:
    matchLabels:
      app: url-shortener
```

Spot 노드 drain 시 PDB 조건을 만족할 때까지 다음 Pod 축출을 대기합니다. 한꺼번에 몰리는 것을 물리적으로 막는 첫 번째 방어선입니다.

---

**2. Pod Topology Spread Constraints — On-Demand 한 노드에 쏠림 방지**

On-Demand 노드가 3대일 때 Pod 30개가 한 노드에 몰리지 않도록 노드 간 균등 분산을 강제합니다.

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1                        # 노드 간 Pod 수 차이 최대 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: DoNotSchedule  # 조건 불만족 시 스케줄 거부
      labelSelector:
        matchLabels:
          app: url-shortener
```

Spot이 전부 사라져 On-Demand에만 Pod가 몰릴 때, 특정 On-Demand 노드에 집중되지 않게 분산시킵니다.

---

**3. Resource Requests/Limits 명확히 설정 — 노드 포화 방지**

On-Demand 노드에 Pod가 몰릴 때 requests가 없으면 한 노드에 무한정 스케줄됩니다.

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

requests가 명확해야 스케줄러가 노드 용량을 계산해서 초과 스케줄을 막습니다. requests 없이 운영하면 On-Demand 노드가 OOM으로 죽는 상황이 발생할 수 있습니다.

---

**4. Cluster Autoscaler — On-Demand도 자동 확장**

Spot이 회수되고 On-Demand가 꽉 찼을 때 On-Demand NodeGroup도 자동으로 늘어나도록 설정합니다.

```
On-Demand NodeGroup: min=2, max=10   ← max를 넉넉하게
Spot NodeGroup:      min=0, max=20
```

CA의 `--expander=priority` 옵션으로 스케일아웃 순서를 명시합니다.

```yaml
# priority expander 설정
priorities: |
  10:
    - .*spot.*        # Spot NodeGroup 우선 확장
  1:
    - .*on-demand.*   # Spot 불가 시 On-Demand 확장
```

Spot이 전부 날아가고 On-Demand도 꽉 찼을 때 CA가 On-Demand를 자동으로 추가해 줍니다.

---

**5. Node Termination Handler (NTH) — 2분 경고 포착해서 선제 drain**

AWS가 Spot 회수 2분 전에 보내는 신호를 감지해 해당 노드를 먼저 drain합니다. CA나 스케줄러가 반응하기 전에 선제적으로 Pod를 이동시켜 On-Demand에 갑작스럽게 몰리는 충격을 줄입니다.

```bash
helm install aws-node-termination-handler \
  eks/aws-node-termination-handler \
  --namespace kube-system
```

NTH 없이는 Spot이 갑자기 사라지고 → Pod가 On-Demand로 한꺼번에 재스케줄되는 충격파가 그대로 옵니다.

---

**정리**

| 예방책 | 막는 것 |
|---|---|
| PDB | 동시에 너무 많은 Pod가 죽는 것 |
| Topology Spread | On-Demand 특정 노드 1개에 쏠리는 것 |
| Resource Requests | 노드 용량 초과 스케줄 |
| CA + priority expander | On-Demand도 자동으로 늘어나도록 |
| Node Termination Handler | 갑작스러운 회수 대신 선제 drain |

이 다섯 가지를 함께 쓰면 Spot 대량 회수 시에도 On-Demand 과부하 없이 트래픽을 처리할 수 있습니다.
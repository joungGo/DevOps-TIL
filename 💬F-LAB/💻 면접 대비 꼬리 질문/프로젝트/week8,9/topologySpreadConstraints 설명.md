```yaml
      # 🛡️ Pod를 여러 AZ/노드에 분산 — 단일 AZ 장애 시 서비스 유지
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone   # 🛡️ AZ 간 분산
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: {{ .Release.Name }}-api
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname         # 🛡️ 노드 간 분산
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: {{ .Release.Name }}-api
```

Pod를 여러 AZ/노드에 균등하게 분산시키는 설정입니다.

---

## 기존과의 차이점

**설정 전**

```
Kubernetes 기본 스케줄러는 "실행 가능한 노드"에 배치
→ 운이 나쁘면 모든 Pod가 같은 AZ/노드에 몰릴 수 있음

[AZ-a]  Pod1, Pod2
[AZ-b]  (없음)
[AZ-c]  (없음)

AZ-a 장애 발생 → 서비스 전체 중단
```

**설정 후**

```
분산 규칙에 따라 AZ/노드에 균등 배치

[AZ-a]  Pod1
[AZ-b]  Pod2
[AZ-c]  (없음, replicas=2이므로)

AZ-a 장애 발생 → Pod2가 AZ-b에서 계속 서비스
```

---

## maxSkew란

**Pod 수의 최대 편차 허용값**입니다.

```
maxSkew: 1 의 의미

[AZ-a]  Pod 2개
[AZ-b]  Pod 1개   → 차이: 1 ✅ (허용)

[AZ-a]  Pod 3개
[AZ-b]  Pod 1개   → 차이: 2 ❌ (위반, 새 Pod 스케줄 거부)
```

---

## 첫 번째 규칙 — AZ 간 분산

```yaml
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone   # AZ 기준
  whenUnsatisfiable: DoNotSchedule           # 조건 불만족 시 스케줄 거부
  labelSelector:
    matchLabels:
      app: {{ .Release.Name }}-api
```

- `topologyKey: topology.kubernetes.io/zone` → AZ(ap-northeast-2a/2b/2c)를 기준으로 분산
- `whenUnsatisfiable: DoNotSchedule` → AZ 간 Pod 수 차이가 1을 초과하면 **배치 자체를 거부**
- 핵심 목적: **AZ 장애 시 서비스 유지**

```
AZ 장애 시나리오
[AZ-a]  Pod1  ← 장애
[AZ-b]  Pod2  ← 계속 운영
→ HPA가 Pod3 추가 → AZ-a 복구 전까지 AZ-b 또는 AZ-c에 배치
```

---

## 두 번째 규칙 — 노드 간 분산

```yaml
- maxSkew: 1
  topologyKey: kubernetes.io/hostname        # 노드 기준
  whenUnsatisfiable: ScheduleAnyway          # 조건 불만족 시에도 스케줄 허용
  labelSelector:
    matchLabels:
      app: {{ .Release.Name }}-api
```

- `topologyKey: kubernetes.io/hostname` → 개별 노드를 기준으로 분산
- `whenUnsatisfiable: ScheduleAnyway` → 노드 간 분산이 불가능해도 **일단 배치는 허용**
- 핵심 목적: **노드 장애 시 영향 최소화**

```
노드 장애 시나리오
[Node-1 / AZ-a]  Pod1  ← 장애
[Node-2 / AZ-b]  Pod2  ← 계속 운영

DoNotSchedule이 아닌 ScheduleAnyway인 이유:
  노드가 1개만 남았을 때도 Pod를 올려야 서비스가 유지됨
  → AZ 분산보다 우선순위가 낮으므로 강제하지 않음
```

---

## 두 규칙을 같이 쓰는 이유

```
AZ 규칙만 있을 때
  [AZ-a / Node-1]  Pod1, Pod2   ← 같은 노드에 몰릴 수 있음
  [AZ-b / Node-2]  (없음)
  → AZ는 분산됐지만 Node-1 장애 시 Pod1, Pod2 동시 중단

AZ + 노드 규칙 함께 쓸 때
  [AZ-a / Node-1]  Pod1
  [AZ-b / Node-2]  Pod2
  → AZ 장애, 노드 장애 모두 대응 가능
```

---

## 전체 요약

|규칙|기준|불만족 시|목적|
|---|---|---|---|
|첫 번째|AZ|스케줄 거부|AZ 장애 대응 (강제)|
|두 번째|노드|일단 허용|노드 장애 대응 (권장)|

---

> [!info] replicas: 2 vs topologySpreadConstraints — 역할이 다르다
>
> **`replicas: 2`가 하는 일**: "Pod를 2개 실행해라" — 개수만 지정, 어디에 배치할지는 관여하지 않는다.
>
> **왜 지금 노드당 1개로 보이냐**
> Kubernetes 스케줄러가 기본적으로 같은 Deployment의 Pod를 다른 노드에 두려는 **성향(scoring)** 을 가지고 있기 때문이다. 노드 2개, Pod 2개면 자연스럽게 1개씩 배치되어 분산된 것처럼 보이지만, 이건 **점수 기반의 선호**이지 **강제 규칙이 아니다.**
>
> **보장이 깨지는 순간**
> - Node 1 CPU 90% → 스케줄러가 Node 2에 Pod 2개 몰아넣을 수 있음 (리소스 점수가 분산 점수를 이김)
> - Node 1 장애 후 재스케줄 → Pod 1이 Node 2로 이동, Node 2에 Pod 2개 동시 존재
> - HPA 스케일 업 → 어느 노드에 몇 개씩 갈지 보장 없음
>
> | | `replicas: 2` | `topologySpreadConstraints` |
> |---|---|---|
> | 역할 | Pod 개수 보장 | 배치 위치 보장 |
> | 분산 방식 | 스케줄러 성향 (best effort) | 강제 규칙 (hard constraint) |
> | 노드 장애 후 재배치 | 여유 있는 노드 아무 곳 | 규칙 만족하는 곳만 |
> | AZ 분산 보장 | ❌ | ✅ |
>
> `replicas: 2`는 **"몇 개"** 를 정하고, `topologySpreadConstraints`는 **"어디에"** 를 강제한다.
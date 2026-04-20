# 9. HPA + Karpenter 스케일링 동작 원리 심화

  

> 부하 테스트(`hey -z 5m -c 50`) 실행 기준으로 Pod와 Node가 어떻게 생성·삭제되는지,

> 그리고 각 판단 기준이 코드의 어느 값에 근거하는지 정리.

  

---

  

## 9-1. 부하 해소 후 Pod / Node 삭제 타이밍

  

### 전체 흐름

  

```

부하 종료

│

├─ Metrics Server가 CPU 하락 감지 (15초마다 체크)

│

▼

[HPA] scale-down 안정화 대기: 5분 (300초)

│ CPU < 50%가 5분간 유지되어야 축소 결정

▼

Pod 축소 (10개 → 2개)

│

▼

[Karpenter] 빈 노드 / 통합 가능 노드 감지

│

▼

consolidateAfter: 30초 대기

│

▼

Node 삭제 (10개 → 2개)

  

총 소요: 약 5분 30초

```

  

---

  

### Pod 삭제 기준 (HPA)

  

**감지 주기**: 15초마다 Metrics Server로부터 CPU 수치를 읽어옴

  

**삭제 조건**:

```

현재 평균 CPU < targetCPUUtilizationPercentage (50%)

```

  

**근데 바로 줄이지 않는다 — scale-down 안정화 윈도우**

  

```yaml

# 기본값 (hpa.yaml에 별도 설정이 없으면 자동 적용)

behavior:

scaleDown:

stabilizationWindowSeconds: 300 # 5분

```

  

HPA는 매 15초마다 "지금 몇 개가 적당한가"를 계산해서 **기록**만 한다.

그리고 과거 300초(5분) 기록 중 **가장 많은 Replica를 요구한 값**으로 결정한다.

  

```

과거 5분간 기록 예시:

T-300s: 2개 권장

T-285s: 2개 권장

T-270s: 4개 권장 ← 이 중 가장 큰 값 채택

T-255s: 2개 권장

...

T-0s: 2개 권장

  

최종 결정: 4개 → 아직 축소 안 함

(4개가 5분 윈도우 안에서 나왔으므로)

```

  

**이렇게 하는 이유 → Flapping 방지**

  

```

안정화 윈도우 없을 때:

부하 급증 → CPU 200% → Pod 10개

1초 후 → CPU 48% → Pod 2개 (즉시 축소)

1초 후 → CPU 180% → Pod 10개 (즉시 확장)

→ Pod가 계속 생겼다 죽었다 반복 = Flapping(진동)

```

  

**커스텀 단축 설정**:

```yaml

# hpa.yaml templates/hpa.yaml

behavior:

scaleDown:

stabilizationWindowSeconds: 60 # 5분 → 1분으로 단축

scaleUp:

stabilizationWindowSeconds: 0 # scaleUp은 기본값이 0 (즉시 반응)

```

  

| 설정값 | 장점 | 단점 | 권장 상황 |

|---|---|---|---|

| 300초 (기본) | 안정적, Flapping 없음 | 비용이 5분 더 발생 | 트래픽이 불규칙한 경우 |

| 60초 | 빠른 원복, 비용 절감 | Flapping 위험 | 트래픽이 예측 가능한 경우 |

  

> **scaleUp은 기본 0초** — 부하가 오면 즉시 늘리고, 줄일 때만 신중하게 기다리는 비대칭 구조.

  

---

  

### Node 삭제 기준 (Karpenter)

  

**코드 기준**:

```yaml

# url-shortener/k8s/karpenter/nodepool.yaml

disruption:

consolidationPolicy: WhenEmptyOrUnderutilized

consolidateAfter: 30s

```

  

**두 가지 삭제 조건**:

  

**① WhenEmpty** — 노드에 Pod가 단 하나도 없을 때

```

Node 8: Pod 없음 (DaemonSet 제외)

→ 30초 후 EC2 인스턴스 종료 ✅

```

  

**② WhenUnderutilized** — 이 노드의 Pod들을 다른 노드로 옮길 수 있을 때

```

Node 7: API Pod 1개 (200m CPU requests)

Node 6: 현재 1500m CPU 여유 있음

  

Karpenter 판단:

"Node 7의 Pod를 Node 6으로 옮길 수 있다"

→ Node 7의 Pod를 evict (정중하게 퇴거)

→ Node 6에서 재스케줄링

→ Node 7 비워짐

→ 30초 후 EC2 인스턴스 종료 ✅

```

  

**판단 기준은 `requests` 값** (실제 사용량이 아님):

```yaml

# url-shortener/url-shortener-chart/values.prod.yaml

resources:

requests:

cpu: 200m ← Karpenter가 이 값으로 "옮길 수 있나?" 계산

memory: 256Mi

```

  

**중요한 예외 — 원복이 안 되는 경우**:

  

```

DaemonSet Pod (kube-proxy, aws-node 등)

→ 모든 노드에 반드시 1개씩 존재

→ WhenEmpty 조건 절대 불충족

→ WhenUnderutilized가 핵심인 이유

  

PodDisruptionBudget(PDB) 설정된 Pod

→ 강제 이동 불가 → 해당 노드 삭제 불가

→ Prometheus, ArgoCD 등 일부 시스템 Pod에 적용됨

```

  

---

  

### 타이밍 요약표

  

| 단계 | 소요 시간 | 담당 | 판단 기준 |

|---|---|---|---|

| CPU 하락 감지 | 즉시 (15초 주기) | Metrics Server | CPU < 50% 감지 |

| Pod 축소 결정 | **+5분** (기본) | HPA | 5분 윈도우에서 가장 큰 권장값 채택 |

| Node 통합 가능성 판단 | 즉시 | Karpenter | requests 기준 재배치 가능 여부 |

| Node 삭제 실행 | **+30초** | Karpenter | `consolidateAfter: 30s` |

| **총 소요** | **약 5분 30초** | | |

  

---

  

## 9-2. 부하 테스트 시 Pod / Node 최대 생성 수

  

### 테스트 명령어

```bash

hey -z 5m -c 50 http://<ALB_DNS>/healthz

# -z 5m : 5분간 실행

# -c 50 : 50개 동시 연결 유지

```

  

### Pod 최대 생성 수 (HPA)

  

**HPA 계산식**:

```

목표 Replica = 현재 Replica × (현재 CPU% / 목표 CPU%)

```

  

**설정값**:

```yaml

# values.prod.yaml

hpa:

minReplicas: 2

maxReplicas: 10

targetCPUUtilizationPercentage: 50

  

resources:

requests:

cpu: 200m # HPA는 이 값 기준으로 % 계산

```

  

**스케일 업 과정**:

  

| 시점 | 현재 Pod | 현재 CPU | 계산 | 결과 |

|---|---|---|---|---|

| 부하 시작 | 2개 | 150% | 2 × (150/50) | → 6개 |

| 1단계 후 | 6개 | 80% | 6 × (80/50) | → 10개 |

| 최종 | **10개 (max)** | — | HPA 상한 도달 | 고정 |

  

---

  

### Node 최대 생성 수 (Karpenter)

  

**이론 계산 (오류 가능성 있음)**:

  

```

t3.medium 1개 allocatable: 약 1820m CPU

시스템 DaemonSet 선점 후: 약 1400m CPU 남음

  

API Pod 1개 requests: 200m

→ 노드 1개당 API Pod 7개 수용 가능

→ 10개 Pod ÷ 7 = 2개 노드면 충분 → 이론상 맞지만 실제는 다름

```

  

**실제 결과: 노드 10개 생성된 이유**

  

```

기존 노드 2개는 시스템 Pod들로 이미 포화 상태:

  

각 노드당 점유:

kube-proxy 100m, 32Mi

aws-node (vpc-cni) 25m, 64Mi

ArgoCD 500m, 256Mi

Karpenter 1000m, 1Gi ← 단독으로 절반 이상 점유

ALB Controller 100m, 128Mi

Prometheus Stack 500m, 512Mi

PostgreSQL 100m, 256Mi

Redis 100m, 128Mi

───────────────────────────────

합계: ~2400m CPU, ~2.2Gi

  

→ 기존 노드에 API Pod가 들어갈 공간이 1~2개 뿐

→ HPA가 요구한 나머지 8개 Pod는 Pending 상태

→ Karpenter가 Pod마다 새 노드 생성 (거의 1:1 비율)

→ NodePool CPU 한도까지 도달

```

  

**NodePool 한도와 일치**:

```yaml

# nodepool.yaml

limits:

cpu: "20" # t3.medium 2 vCPU × 10개 = 20 vCPU = 한도

memory: "40Gi"

```

  

```

20 vCPU ÷ t3.medium 2 vCPU = 최대 10개 노드 ← 한도에 딱 맞게 생성됨

```

  

---

  

### 최종 결과표

  

| | Pod 수 | Node 수 |

|---|---|---|

| **부하 전** | 2개 | 2개 |

| **부하 중** | **최대 10개** | **최대 10개** |

| **상한 결정 기준** | `hpa.maxReplicas: 10` | `nodepool limits.cpu: "20"` |

| **부하 후 원복** | 2개 (약 5분 후) | 2개 (약 5분 30초 후) |

  

---

  

### 노드 수 조정 방법

  

노드가 10개까지 생성되어 비용이 부담된다면:

  

**방법 1: NodePool limits 낮추기**

```yaml

# nodepool.yaml

limits:

cpu: "10" # 최대 5개 노드로 제한

memory: "20Gi"

```

  

**방법 2: API Pod requests 높이기 (노드당 Pod 수 감소)**

```yaml

# values.prod.yaml

resources:

requests:

cpu: 500m # 200m → 500m으로 상향

memory: 256Mi

# → 노드 1개당 API Pod 약 2~3개만 배치 가능

# → 기존 노드 여유 공간을 더 효율적으로 활용

```

  

**방법 3: HPA maxReplicas 낮추기**

```yaml

# values.prod.yaml

hpa:

maxReplicas: 5 # 10 → 5로 제한

```

  

> **실무 팁**: requests를 실제 평균 사용량에 맞게 설정하는 것이 노드 수와 비용을 가장 효과적으로 제어하는 방법.

> requests가 너무 낮으면 노드 한 개에 너무 많은 Pod가 몰려 실제 성능이 저하될 수 있음.
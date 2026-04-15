## 1. 각 방식의 트래픽 흐름

### `target-type: instance` (EC2 노드 경유)

```
인터넷
  │
  ▼
ALB
  │  ← EC2 노드 IP + NodePort (예: 30080)로 전달
  ▼
EC2 Node (NodePort: 30080)
  │
  ▼
kube-proxy (iptables 규칙으로 Pod 선택)
  │
  ▼        ← 같은 노드일 수도, 다른 노드일 수도 있음
Pod
```

**핵심**: ALB는 Pod가 어디 있는지 모름. EC2 노드에 던지고, 노드 안의 kube-proxy가 적절한 Pod로 라우팅.

---

### `target-type: ip` (Pod 직접)

```
인터넷
  │
  ▼
ALB
  │  ← Pod IP로 직접 전달 (예: 10.0.10.45)
  ▼
Pod (ENI를 통해 VPC 내 IP 직접 접근)
```

**핵심**: ALB가 Pod IP를 직접 알고 있음. EC2 노드와 kube-proxy를 완전히 우회.

> [!info] Target Group이란?
> 
> ALB(Application Load Balancer)가 트래픽을 전달할 **대상 목록**이다.
> ALB는 직접 서버에 트래픽을 보내지 않고, Target Group에 등록된 대상들에게 분산 전달한다.
> 
> ```
> 인터넷
>   │
>   ▼
>  ALB  ──→  Target Group  ──→  대상 1 (Pod 또는 EC2)
>                          ├──→  대상 2
>                          └──→  대상 3
> ```
> 
> **Target Group의 역할 3가지**
> 
> | 역할 | 설명 |
> |---|---|
> | 대상 목록 관리 | 트래픽을 받을 IP/포트 목록을 보유 |
> | 헬스체크 | 주기적으로 각 대상에 요청을 보내 정상 여부 확인 |
> | 로드밸런싱 | 정상 대상에게만 트래픽을 균등하게 분산 |
> 
> **`target-type`에 따라 등록 대상이 달라짐**
> 
> - `target-type: instance` → **EC2 노드 IP + NodePort** 등록
>   - 노드 10개 = Target 10개
>   - Pod가 아무리 많아도 Target 수는 노드 수와 동일
> 
> - `target-type: ip` → **Pod IP** 개별 등록
>   - Pod 1개 = Target 1개
>   - Pod가 수백~수천 개면 Target도 그만큼 증가
>   - ALB Target Group 기본 한도: **1,000개**
>   - 대규모 클러스터에서 한도 초과 가능
> 
> **헬스체크 동작**
> 
> ```
> Target Group이 30초마다 각 대상에 GET /healthz 요청
>   │
>   ├─ 200 응답 → Healthy → 트래픽 전송 대상 유지
>   └─ 응답 없음 / 5xx → Unhealthy → 트래픽 전송 대상에서 제외
> ```
> 
> Pod가 CrashLoopBackOff 상태가 되면 헬스체크 실패 → Target Group에서 자동 제외 → 해당 Pod로 트래픽 유입 차단


---

## 2. `instance` 방식을 써야 하는 경우

### 이유 1: VPC CNI를 사용하지 않는 환경

```
EKS VPC CNI (aws-node) 가 있어야 Pod에 VPC IP가 할당됨
  └─ ip 모드 전제 조건

온프레미스 쿠버네티스, 또는 다른 CNI 플러그인 사용 시
  (Calico, Flannel, Weave 등)
  └─ Pod IP가 VPC 외부 IP → ALB가 직접 접근 불가
  └─ instance 모드만 가능
```

### 이유 2: NodePort가 이미 존재하는 레거시 구조

```
기존 시스템이 NodePort 기반으로 설계되어 있고
ALB를 앞에 붙이는 경우

NodePort: 30080 → 기존 방화벽/보안 정책이 이 포트 기준으로 설정됨
→ ip 모드로 바꾸면 NodePort가 불필요해져서 구조 변경 필요
→ 레거시 호환성을 위해 instance 유지
```

### 이유 3: 대규모 클러스터에서 Target Group 등록 한도

```
target-type: ip
  → ALB Target Group에 Pod IP를 개별 등록
  → Pod 1개 = Target 1개

Pod 수가 매우 많으면 (수백~수천 개):
  ALB Target Group 한도: 1,000개 (기본)
  → Pod가 많으면 한도 초과 가능

target-type: instance
  → 노드 IP만 등록
  → 노드 10개 = Target 10개
  → Pod가 아무리 많아도 Target은 노드 수와 동일
  → 대규모 클러스터에서 Target 수 부담 없음
```

---

## 3. 방식별 상세 비교

|항목|`instance`|`ip`|
|---|---|---|
|**트래픽 경로**|ALB → NodePort → kube-proxy → Pod|ALB → Pod 직접|
|**레이턴시**|높음 (홉 2개)|낮음 (홉 1개)|
|**헬스체크 대상**|노드 IP + NodePort|Pod IP + containerPort|
|**헬스체크 정확도**|낮음 (노드는 살아있어도 Pod가 죽었을 수 있음)|높음 (Pod 상태 직접 감지)|
|**Target 수**|노드 수|Pod 수|
|**대규모 Pod 환경**|유리 (Target 적음)|불리 (Target 많음)|
|**VPC CNI 필요 여부**|불필요|필요|
|**EKS 권장**|❌|✅|
|**Cross-zone 홉**|발생 가능|없음|

---

## 4. 헬스체크 정확도 차이가 중요한 이유

```
[instance 모드 문제 상황]

Node는 정상 동작 중
  └─ ALB 헬스체크: ✅ (노드에 응답)

Pod는 OOM으로 CrashLoopBackOff 상태
  └─ ALB는 모름 → 계속 트래픽 전송

결과: 503 에러가 ALB 레벨에서 감지 안 됨
     kube-proxy가 죽은 Pod로 라우팅 시도 → 에러
```

```
[ip 모드]

Pod가 CrashLoopBackOff 상태
  └─ ALB 헬스체크: ❌ (Pod에 직접 응답 없음)
  └─ 즉시 Target에서 제외
  └─ 살아있는 Pod에만 트래픽 전송

결과: 장애 Pod로 트래픽 유입 없음
```

---

## 5. 지금 프로젝트에서 `ip`를 쓰는 이유

```
EKS + VPC CNI 조합
  └─ Pod마다 VPC IP 자동 할당 (ip 모드 전제 조건 충족)

HPA + Karpenter로 Pod/Node가 자주 생성·삭제됨
  └─ ip 모드: Pod 생성/삭제 시 ALB Target 자동 갱신
  └─ instance 모드: 노드 단위라 Pod 상태 즉시 반영 어려움

Pod 수: 최대 10개 → Target Group 한도(1,000개)와 거리가 멀어 ip 모드 부담 없음
```

**결론**: 지금처럼 EKS + VPC CNI + 오토스케일링 조합에서는 `ip` 모드가 정확도, 레이턴시, 오토스케일링 연동 모든 면에서 유리해. `instance` 모드는 VPC CNI를 못 쓰거나, Pod 수가 극단적으로 많거나, 레거시 호환이 필요한 특수한 경우에만 선택하는 거야.
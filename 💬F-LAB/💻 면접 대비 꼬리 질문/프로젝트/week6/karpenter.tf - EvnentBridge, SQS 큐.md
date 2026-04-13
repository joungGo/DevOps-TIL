
### Spot 인스턴스란?

AWS EC2는 두 가지 구매 방식이 있습니다.

```
On-demand: 정가 지불, 언제든 사용 가능, 중단 없음
Spot:       남는 자원을 최대 90% 할인, 단 AWS가 필요하면 강제 회수
```

Spot은 저렴하지만 **AWS가 2분 전 통보 후 인스턴스를 강제 종료**할 수 있습니다. 이 2분 안에 Pod를 안전하게 다른 노드로 옮기는 것이 Spot 인터럽션 처리의 핵심입니다.

---

### 전체 흐름

```
AWS: "이 Spot 인스턴스 2분 후 회수할게"
    ↓
EventBridge: 이벤트 감지 → SQS 큐에 메시지 전달
    ↓
Karpenter: 큐 폴링 → 메시지 수신
    ↓
Karpenter: 해당 노드 Pod 드레인 (다른 노드로 이동)
    ↓
AWS: Spot 인스턴스 종료
```

---

### SQS 큐 (`queue_name = "Karpenter-${var.project_name}"`)

```
Karpenter-urlshortener
```

AWS 이벤트를 받아서 쌓아두는 메시지 큐입니다. Karpenter가 이 큐를 지속적으로 폴링해서 메시지를 꺼내갑니다.

```
EventBridge ──→ SQS 큐 ──→ Karpenter 수신
               (메시지 보관)
```

SQS가 중간에 있는 이유는 Karpenter가 이벤트를 놓치지 않기 위해서입니다. Karpenter Pod가 재시작 중이어도 메시지가 큐에 남아있기 때문입니다.

---

### EventBridge 규칙 3가지

모듈이 자동으로 만드는 EventBridge 규칙입니다.

**1. Spot 인터럽션 경고**

```json
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Spot Instance Interruption Warning"]
}
```

Spot 회수 **2분 전** AWS가 발생시키는 이벤트입니다. 가장 중요한 이벤트입니다.

**2. EC2 상태 변경**

```json
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"]
}
```

인스턴스가 `terminated`, `stopped` 등으로 상태가 바뀔 때 발생합니다. Spot 회수가 실제로 완료됐을 때 Karpenter가 해당 노드를 클러스터에서 제거합니다.

**3. AWS Health 이벤트**

```json
{
  "source": ["aws.health"],
  "detail-type": ["AWS Health Event"]
}
```

AWS가 계획된 유지보수, 하드웨어 교체 등을 미리 알려주는 이벤트입니다. Spot이 아닌 On-demand도 해당될 수 있습니다.

---

### Karpenter가 메시지를 받으면 하는 일

```
1. 해당 노드에 새 Pod 스케줄링 중단 (cordon)
        ↓
2. 기존 Pod를 다른 노드로 이동 (drain)
        ↓
3. 필요하면 새 노드 프로비저닝
        ↓
4. AWS가 Spot 인스턴스 종료
```

2분이라는 시간 안에 이 과정이 완료되어야 서비스 중단 없이 처리됩니다.

---

### enable_spot_termination = false로 하면?

```
AWS: Spot 회수 이벤트 발생
    ↓
아무도 감지 안 함
    ↓
2분 후 인스턴스 강제 종료
    ↓
해당 노드의 Pod 전부 죽음 → 서비스 중단
```

Spot을 사용한다면 반드시 `true`로 설정해야 합니다.

> [!WARNING] Karpenter Pod 배치 주의사항
> Karpenter Pod가 자신이 관리하는 Spot 노드에 올라가면, Spot 인터럽션 발생 시 Karpenter 자신이 죽어버려 이후 노드 프로비저닝이 불가능해집니다.
>
> 실무에서는 Karpenter Pod를 Terraform이 관리하는 **안정적인 기본 노드(On-demand)에만 배치**되도록 강제합니다.
>
> ```yaml
> # cluster-addons/karpenter/application.yaml
> helm:
>   values: |
>     nodeSelector:
>       karpenter.sh/nodepool: ""  # Karpenter가 만든 노드 제외
>     tolerations:
>       - key: "CriticalAddonsOnly"
>         operator: "Exists"
> ```
>
> **배치 우선순위**
> - Karpenter Pod → 기본 노드 (Terraform EKS NodeGroup, On-demand)
> - 일반 앱 Pod → Karpenter가 추가한 노드 (Spot/On-demand)


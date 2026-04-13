### 상황

`terraform destroy` 후 `terraform apply`를 다시 실행하면 Karpenter Controller IAM Role이 새로 생성되면서 ARN이 변경됩니다.

```bash
# 구 ARN (이전 클러스터)
arn:aws:iam::716174522908:role/KarpenterController-20260412080046592800000014

# 신 ARN (새 클러스터)
arn:aws:iam::716174522908:role/KarpenterController-20260413094918032900000018
```

새 ARN을 `cluster-addons/karpenter/application.yaml`에 반영하고 `kubectl apply`를 실행했지만 Karpenter가 여전히 구 ARN으로 AWS API를 호출해서 아래 에러가 발생했습니다.

```json
{
  "level": "ERROR",
  "message": "ec2 api connectivity check failed",
  "error": "...operation error STS: AssumeRoleWithWebIdentity,
            StatusCode: 403,
            api error AccessDenied: Not authorized to perform sts:AssumeRoleWithWebIdentity"
}
```

---

### 핵심 개념

**Deployment**는 Pod를 관리하는 K8s 오브젝트입니다.

```
Deployment (관리자)
    └── Pod (실제 실행 중인 컨테이너)
            └── 환경변수
                    AWS_ROLE_ARN: arn:...구ARN  ← 시작할 때 고정됨
                    AWS_REGION: ap-northeast-2
```

**ServiceAccount**는 Pod가 AWS API를 호출할 때 사용할 IAM Role을 지정하는 오브젝트입니다.

```yaml
ServiceAccount
  annotation:
    eks.amazonaws.com/role-arn: arn:...새ARN  ← annotation 변경 가능
```

Pod는 **시작할 때 딱 한 번** ServiceAccount annotation을 읽어서 환경변수로 주입합니다. 이후 annotation이 바뀌어도 이미 실행 중인 Pod의 환경변수는 바뀌지 않습니다.

> 출근할 때 사원증을 받으면 하루 종일 그 사원증으로 일합니다. 사원증 정책이 바뀌어도 퇴근 후 다시 출근(재시작)해야 새 사원증을 받습니다.

---

### `kubectl apply -f application.yaml`이 하는 일

`application.yaml`의 `kind: Application` 때문에 이 파일은 ArgoCD Application 오브젝트가 됩니다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application        ← ArgoCD CRD로 등록된 타입
metadata:
  name: karpenter
  namespace: argocd
```

K8s는 `kind` 값으로 어떤 오브젝트를 만들지 결정합니다.

```
kind: Deployment    → Deployment 오브젝트 생성
kind: Service       → Service 오브젝트 생성
kind: Application   → ArgoCD Application 오브젝트 생성 (ArgoCD CRD)
```

ArgoCD가 클러스터에 설치되면서 `Application`이라는 새로운 오브젝트 타입을 K8s에 등록(CRD)합니다. 그래서 `kubectl apply`로 이 파일을 적용하면 ArgoCD가 감지하고 관리를 시작합니다.

```
kubectl apply -f application.yaml
    ↓
kind: Application 확인 → ArgoCD CRD 타입
    ↓
ArgoCD Application 오브젝트 생성/수정
    ↓
ArgoCD 감지 → spec.source (Git 레포, chart, targetRevision) 읽음
    ↓
Helm chart 설치/업데이트 실행
    ↓
helm values의 role-arn으로 ServiceAccount annotation 업데이트
```

ArgoCD는 평소에 GitHub만 감시하지만, `kubectl apply`로 직접 Application 오브젝트를 수정하면 그 설정을 즉시 반영합니다.

---

### `kubectl apply` 후에도 ARN이 반영되지 않은 이유

```
application.yaml 수정 (새 ARN)
    ↓
kubectl apply
    ↓
ArgoCD Application 오브젝트 업데이트
    ↓
ArgoCD → Helm values 변경 감지
    ↓
ServiceAccount annotation 업데이트 (새 ARN)  ← 여기까지만 자동
    ↓
Deployment 스펙 변경 없음
    ↓  role-arn은 Deployment 스펙이 아닌
       ServiceAccount annotation에 붙기 때문
    ↓
Pod 재시작 안 함
    ↓
기존 Pod는 구 ARN 환경변수 유지 → 인증 실패
```

---

### 해결 방법

Pod를 강제로 재시작해서 새 ARN을 읽게 합니다.

```bash
# Pod 재시작
kubectl rollout restart deployment karpenter -n karpenter

# 재시작 완료 확인
kubectl rollout status deployment karpenter -n karpenter

# 새 ARN 적용 확인
kubectl describe pod -n karpenter -l app.kubernetes.io/name=karpenter \
  | grep AWS_ROLE_ARN
```

재시작 후 흐름:

```
새 Pod 시작
    ↓
ServiceAccount annotation 읽음
    eks.amazonaws.com/role-arn: 새ARN
    ↓
환경변수 주입
    AWS_ROLE_ARN=새ARN
    ↓
AWS STS 인증 성공 → Karpenter 정상 동작
```
### 1. ArgoCD 설치 시 생성되는 것들

ArgoCD를 설치하면 여러 Pod가 뜹니다. 각각 역할이 다릅니다.

```bash
kubectl get pods -n argocd
# argocd-server                  ← UI/API 서버
# argocd-repo-server             ← Git 레포 클론/감시
# argocd-application-controller  ← 클러스터 상태 비교/동기화
# argocd-redis                   ← 캐시
# argocd-dex-server              ← 인증
```

---

### 2. Application 오브젝트 등록

`kubectl apply -f application.yaml`로 ArgoCD에 감시할 대상을 등록합니다.

```yaml
spec:
  source:
    repoURL: https://github.com/f-lab-edu/Url-Shortener-EKS-Platform
    targetRevision: feat/week6to7
    path: url-shortener/url-shortener-chart
  destination:
    server: https://kubernetes.default.svc
    namespace: url-shortener
```

ArgoCD는 이 정보를 보고 **"이 Git 레포의 이 브랜치를 감시해서 이 클러스터 네임스페이스와 일치시켜라"** 라고 인식합니다.

---

### 3. repo-server가 Git을 주기적으로 폴링

ArgoCD는 **Push 방식이 아닌 Pull 방식**입니다. GitHub가 ArgoCD에 알려주는 게 아니라, ArgoCD가 주기적으로 GitHub에 가서 확인합니다.

```
argocd-repo-server
    ↓ 3분마다 (기본값)
GitHub 레포 clone/fetch
    ↓
최신 커밋 SHA 확인
    ↓
이전에 확인한 SHA와 다름 → 변경 감지
```

```bash
# 폴링 주기 변경 (기본 3분)
argocd-cm ConfigMap:
  timeout.reconciliation: 180s
```

---

### 4. application-controller가 클러스터 상태와 비교

```
Git 상태 (desired state)          클러스터 실제 상태 (actual state)
──────────────────────            ──────────────────────────────
values.prod.yaml                  kubectl get deployment
  image.tag: sha-abc123     vs      image.tag: sha-old456
  replicas: 2                       replicas: 2
```

```
비교 결과:
  image.tag 다름 → OutOfSync 상태
  replicas 같음  → Synced
```

---

### 5. Sync 실행 (자동 또는 수동)

```yaml
# application.yaml
syncPolicy:
  automated:
    prune: true      # Git에서 삭제된 리소스 클러스터에서도 삭제
    selfHeal: true   # 클러스터가 Git과 달라지면 자동으로 되돌림
```

`automated` 설정이 있으면 OutOfSync 감지 즉시 자동 Sync합니다.

```
OutOfSync 감지
    ↓
Helm template 렌더링
    ↓
kubectl apply (내부적으로)
    ↓
클러스터 상태 = Git 상태
    ↓
Synced
```

---

### 전체 흐름

```
Git에 push (values.prod.yaml image.tag 변경)
    ↓
argocd-repo-server: 3분마다 폴링
    ↓
새 커밋 SHA 감지
    ↓
argocd-application-controller: 클러스터 상태와 비교
    ↓
image.tag 다름 → OutOfSync
    ↓
automated sync → Helm template 렌더링 → kubectl apply
    ↓
새 이미지로 Pod 재배포
    ↓
Synced
```

---

### GitOps의 핵심

```
Git = 진실의 원천 (Source of Truth)

누군가 kubectl로 직접 클러스터를 수정해도
    ↓
selfHeal: true 설정이면
    ↓
ArgoCD가 감지하고 Git 상태로 되돌림
```

클러스터의 모든 변경은 Git을 통해서만 이루어져야 한다는 원칙입니다.
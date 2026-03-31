
GitOps는 Git을 단일 소스로 사용하여 클러스터 상태를 자동 동기화하고, 변경 이력 관리와 롤백, 자동 배포를 가능하게 하는 운영 방식입니다.

---

## GitOps란

GitOps는 **Git 저장소를 인프라와 애플리케이션 설정의 단일 진실 소스로 두고, Git 상태를 기준으로 실제 클러스터를 자동 동기화하는 운영 방식**입니다.  
즉, 모든 변경은 Git commit을 통해 수행되고, Git에 정의된 상태가 실제 운영 상태가 됩니다.

쉽게 말하면 이렇습니다.

```
기존 방식:  개발자가 직접 kubectl apply, terraform apply 실행
GitOps 방식: Git에 push만 하면 → 자동으로 클러스터에 반영
```

---

## 핵심 원칙 4가지

**① 선언적(Declarative)** "어떻게 하라"가 아니라 "이런 상태여야 한다"를 Git에 기술합니다. Terraform의 `.tf` 파일, K8s의 `yaml` 파일이 모두 선언적입니다.

**② Git이 유일한 진실** 클러스터의 실제 상태는 항상 Git의 내용과 같아야 합니다. 누군가 콘솔에서 직접 바꿨다면 그건 틀린 상태입니다.

**③ 변경은 PR로만** 직접 `kubectl apply`나 `terraform apply`를 치는 게 아니라, Git PR → 리뷰 → merge → 자동 반영 흐름을 강제합니다.

**④ 자동 동기화** ArgoCD, Flux 같은 GitOps 도구가 Git을 지속적으로 감시하다가 클러스터 상태가 Git과 다르면 자동으로 맞춥니다.

---

## 기존 방식 vs GitOps

```
기존 방식
개발자 A → kubectl apply -f deployment.yaml  (직접 실행)
개발자 B → kubectl apply -f deployment.yaml  (또 직접 실행)
→ "지금 클러스터가 어떤 상태인지" 아무도 확실히 모름
→ 누가 언제 바꿨는지 추적 불가

GitOps
개발자 A → PR 생성 → 리뷰 → merge
              ↓
           ArgoCD가 감지 → 자동으로 클러스터에 반영
→ Git 히스토리 = 클러스터 변경 히스토리
→ 누가 언제 왜 바꿨는지 완전 추적 가능
```

---

## 왜 중요한가

**1. 감사(Audit) 가능** 모든 변경이 Git commit으로 남습니다. "언제 누가 replicas를 3→10으로 바꿨나"를 git log로 즉시 확인 가능합니다.

**2. 빠른 롤백** 장애 발생 시 Git revert 한 번으로 이전 상태로 돌아갑니다. `kubectl rollout undo`보다 훨씬 안전합니다.

**3. 드리프트 방지** 누군가 콘솔에서 몰래 설정을 바꾸면 ArgoCD가 감지해서 Git 상태로 자동 복구합니다. "콘솔 클릭 → 그게 반영된 줄 알았는데 사실 안 됨" 같은 상황이 없어집니다.

**4. 협업과 리뷰** 인프라 변경도 코드 리뷰와 동일한 프로세스를 탑니다. PR에서 `terraform plan` 결과가 자동으로 댓글로 달리고, 팀원이 리뷰 후 merge합니다.

**5. 재해 복구** 클러스터가 완전히 날아가도 Git에 있는 내용으로 `terraform apply` 한 번이면 동일한 환경을 재현할 수 있습니다.

---

## Week4 구조와 연결하면

지금 배우고 있는 내용이 이미 GitOps의 재료입니다.

```
Terraform 코드  →  Git에 올라가 있음  →  PR로 인프라 변경
Helm Chart      →  Git에 올라가 있음  →  PR로 앱 배포 변경
```

여기에 ArgoCD(또는 GitHub Actions)를 붙이면 완전한 GitOps 파이프라인이 됩니다.

```
Git push
  → GitHub Actions가 terraform plan 결과를 PR 댓글로 게시
  → 리뷰 후 merge
  → terraform apply 자동 실행
  → EKS 클러스터 상태 자동 반영
```

Week4에서 `-auto-approve`를 로컬에서 쓰지 말라고 한 이유도 결국 GitOps 원칙과 같습니다. 사람이 직접 실행하는 게 아니라 Git → CI/CD → 자동 실행이 표준이기 때문입니다.
## Access Key를 사용해야 하는 경우

### 1. AWS 접근이 OIDC를 지원하지 않는 경우

OIDC는 GitHub Actions 전용입니다. 아래 환경에서는 사용 불가합니다.

```
- Jenkins
- GitLab CI (자체 호스팅)
- CircleCI (일부 설정)
- 로컬 스크립트 자동화
```

이런 경우 Access Key를 Secret에 저장해서 사용합니다.

```yaml
# GitHub Secret에 저장
AWS_ACCESS_KEY_ID: AKIA...
AWS_SECRET_ACCESS_KEY: xxxx...
```

---

### 2. AWS가 아닌 외부 서비스 자격증명

OIDC는 AWS, GCP, Azure 등 클라우드 서비스에서만 지원합니다. **외부 SaaS나 서드파티 서비스**는 OIDC를 지원하지 않아서 Secret에 저장할 수밖에 없습니다.

실무에서 자주 쓰이는 예시입니다.

|Secret|용도|
|---|---|
|`SLACK_WEBHOOK_URL`|배포 결과 슬랙 알림|
|`SONAR_TOKEN`|SonarQube 코드 품질 분석|
|`DOCKERHUB_TOKEN`|Docker Hub push (ECR 대신 사용 시)|
|`NPM_TOKEN`|Private npm 패키지 publish|
|`SENTRY_DSN`|Sentry 에러 트래킹 연동|
|`DATADOG_API_KEY`|Datadog 모니터링 연동|
|`TELEGRAM_BOT_TOKEN`|배포 알림 봇|

```yaml
# 실무 워크플로우 예시
- name: 슬랙 알림
  run: |
    curl -X POST ${{ secrets.SLACK_WEBHOOK_URL }} \
      -d '{"text": "배포 완료"}'

- name: SonarQube 분석
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
  run: sonar-scanner
```

---

### 3. 조직 내 레포지토리가 빈번하게 생성/삭제되는 경우

OIDC 신뢰 정책은 레포지토리를 명시적으로 지정합니다.

```hcl
condition {
  variable = "token.actions.githubusercontent.com:sub"
  values   = ["repo:f-lab-edu/Url-Shortener-EKS-Platform:ref:refs/heads/main"]
}
```

레포가 10개, 20개로 늘어나면 매번 Terraform 코드를 수정하고 `terraform apply`를 다시 실행해야 합니다.

```
레포 추가될 때마다:
  tf 코드 수정 → PR → 리뷰 → apply
  
레포 삭제될 때:
  tf 코드에서 제거 → PR → 리뷰 → apply
```

조직 전체를 허용하는 방법이 있긴 합니다.

```hcl
values = ["repo:f-lab-edu/*"]  # 조직 전체 허용
```

하지만 이렇게 하면 조직 내 모든 레포가 AWS에 접근 가능해져서 **보안이 약해집니다.**

반면 Access Key는 Secret만 등록하면 끝납니다.

```
레포 추가될 때:
  GitHub Secret에 AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY 등록
  → 끝
```

---

### 4. AWS Organizations를 통한 세밀한 제어

**AWS Organizations**는 여러 AWS 계정을 하나의 조직으로 묶어 중앙에서 관리하는 서비스입니다.

```
AWS Organizations
├── Root
│   ├── 개발 계정 (dev)
│   ├── 스테이징 계정 (staging)
│   └── 프로덕션 계정 (prod)
```

Organizations에서는 **SCP(Service Control Policy)** 로 Access Key 사용을 세밀하게 제어할 수 있습니다.

```json
// SCP 예시: 특정 리전에서만 사용 가능
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestedRegion": "ap-northeast-2"
    }
  }
}
```

```json
// SCP 예시: 특정 IP에서만 Access Key 사용 가능
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "NotIpAddress": {
      "aws:SourceIp": ["203.0.113.0/24"]  // 회사 IP만 허용
    }
  }
}
```

OIDC는 GitHub 레벨에서 제어하지만, Organizations SCP는 **AWS 레벨에서 강제 적용**되어 IAM 정책보다 우선순위가 높습니다.

---

### 5. Access Key의 3개 토큰 구조와 Session Token 만료 정책

Access Key 방식에는 두 가지 종류가 있습니다.

**장기 자격증명 (IAM User Access Key)**

```
AWS_ACCESS_KEY_ID     = AKIA...   # 영구 키
AWS_SECRET_ACCESS_KEY = xxxx...   # 영구 시크릿
```

만료 없이 영구적으로 유효해서 유출 시 위험합니다.

**단기 자격증명 (STS Temporary Credentials)**

```
AWS_ACCESS_KEY_ID     = ASIA...   # 임시 키 (ASIA로 시작)
AWS_SECRET_ACCESS_KEY = xxxx...   # 임시 시크릿
AWS_SESSION_TOKEN     = yyyy...   # 세션 토큰 ← 이 3개가 모두 있어야 인증됨
```

3개가 세트로 동작하며 **하나라도 없으면 인증 실패**합니다.

```
AWS_ACCESS_KEY_ID + AWS_SECRET_ACCESS_KEY 만으로는 인증 불가
반드시 AWS_SESSION_TOKEN 까지 3개가 있어야 함
```

**Session Token 만료 정책 설정**

```bash
# 임시 자격증명 발급 시 만료 시간 지정
aws sts assume-role \
  --role-arn arn:aws:iam::716174522908:role/my-role \
  --role-session-name my-session \
  --duration-seconds 3600   # 1시간 (최소 15분, 최대 12시간)
```

IAM Role 자체에도 최대 세션 시간을 설정할 수 있습니다.

```hcl
resource "aws_iam_role" "github_actions" {
  name                 = "urlshortener-github-actions-role"
  assume_role_policy   = ...
  max_session_duration = 3600  # 최대 1시간으로 제한
}
```

|설정|범위|특징|
|---|---|---|
|`--duration-seconds`|요청 시 지정|Role의 max_session_duration 초과 불가|
|`max_session_duration`|Role에 고정|이 Role을 assume하는 모든 대상에 적용|

---

### 정리

|상황|권장 방식|
|---|---|
|레포가 고정적, 클라우드 리소스 접근|OIDC|
|레포가 빈번하게 생성/삭제|Access Key (STS 임시 자격증명)|
|조직 레벨 보안 정책 적용|Organizations SCP + Access Key|
|외부 SaaS 연동|Access Key (장기 자격증명)|

실무에서는 OIDC가 베스트 프랙티스이지만, **조직 규모와 운영 방식에 따라 STS 임시 자격증명 + Organizations 조합이 더 현실적인 경우도 있습니다.**
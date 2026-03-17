# ⛵ Helm 완벽 가이드

---

## 🔷 Helm이란?

**Helm**은 Kubernetes의 **패키지 매니저(Package Manager)** 입니다. Linux의 `apt`, `yum`, macOS의 `brew`와 유사한 역할을 Kubernetes 환경에서 수행합니다.

Helm을 사용하면 복잡한 Kubernetes 애플리케이션을 **Chart**라는 단일 패키지로 묶어 손쉽게 설치, 업그레이드, 롤백, 삭제할 수 있습니다.

[🔗 공식 사이트: helm.sh](https://helm.sh/)

---

### ✅ Helm의 핵심 개념

|용어|설명|
|---|---|
|**Chart**|Kubernetes 리소스들을 하나의 패키지로 묶은 단위 (앱 설치 패키지)|
|**Release**|Chart를 클러스터에 설치한 인스턴스 (동일 Chart로 여러 Release 가능)|
|**Revision**|Release의 업그레이드/변경 이력 번호|
|**Repository**|Chart들을 저장하고 배포하는 저장소|
|**Values**|Chart 템플릿에 주입되는 설정 값 (`values.yaml`)|

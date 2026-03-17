**🔍 왜 Helm을 배우나? — kubectl apply로 충분하지 않나?**

    Week2에서 kubectl apply -f 로 YAML 7개를 순서대로 적용했습니다.

    지금은 서비스가 1개라 괜찮지만, 서비스가 늘어나면 문제가 생깁니다.

kubectl apply의 한계:

      - 환경(dev/prod)마다 YAML을 따로 관리해야 함

      - 이미지 태그 하나 바꾸려면 YAML 파일을 직접 수정해야 함

      - 어떤 버전이 배포됐는지 이력 추적이 어려움

      - 롤백하려면 이전 YAML을 직접 찾아서 다시 apply해야 함

    Helm이 해결하는 것:

      - YAML을 템플릿화 → 환경별 values 파일 하나만 바꾸면 됨

      - helm install / upgrade / rollback 으로 배포 이력 자동 관리

      - 여러 YAML을 하나의 Chart로 묶어 한 번에 배포/삭제

    → EKS 프로덕션 환경에서는 Helm이 사실상 표준 배포 도구입니다.
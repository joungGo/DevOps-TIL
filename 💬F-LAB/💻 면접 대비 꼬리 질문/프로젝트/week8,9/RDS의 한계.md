  

| 항목           | RDS PostgreSQL    | Aurora PostgreSQL | Aurora Serverless v2 | StatefulSet (유지) |
| ------------ | ----------------- | ----------------- | -------------------- | ---------------- |
| 관리 부담        | 낮음                | 낮음                | 매우 낮음                | 높음               |
| Read Replica | 코드 1줄 추가          | 자동 (최대 15개)       | 자동                   | 수동 구성            |
| Multi-AZ     | 옵션                | 기본 6-way 복제       | 기본                   | 미지원              |
| 최소 비용        | ~$13/월 (t3.micro) | ~$58/월            | 사용량 기반               | EC2 비용만          |
| EKS 통합       | VPC 내부 연결         | VPC 내부 연결         | VPC 내부 연결            | 네이티브             |

트래픽이 일정 수준 이상이면 Aurora PostgreSQL을 선택함. Aurora는 Primary 장애 시 자동 페일오버(30초 이내), 최대 15개 Read Replica를 지원하며 클러스터 볼륨이 자동 확장되는 기능이 있음.

>RDS는 replica를 수동으로 생성하거나 생성하는 api를 호출하는 로직을 작성해야 함.
>또한, replica 생성 수 max가 5개임.
>
>RDS의 최대 단점은 Primary와 Replica 간 데이터 동기화(WAL) 간 Lag가 필수적으로 발생한다는 점임.
>하지만, Aurora는 이를 해결해줌.
>- RDS: 각 인스턴스(Primary, Replica) 당 하나의 스토리지
>- Aurora: 공유 스토리지 사용

[[RDS vs Aurora]]


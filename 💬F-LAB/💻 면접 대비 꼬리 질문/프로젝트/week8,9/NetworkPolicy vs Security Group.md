우선순위 경쟁이 아닙니다. **둘 다 통과해야** 트래픽이 도달합니다.

```
트래픽 흐름

앱 Pod → RDS
    ↓
1단계: NetworkPolicy 검사 (Kubernetes 레벨)
    ↓ 통과
2단계: Security Group 검사 (AWS VPC 레벨)
    ↓ 통과
3단계: RDS 도달
```

- 하나라도 막으면 연결 차단
- Security Group이 허용해도 NetworkPolicy가 막으면 안 됨
- NetworkPolicy가 허용해도 Security Group이 막으면 안 됨

**RDS.tf의 Security Group과 NetworkPolicy가 다르다면:**

```
예) Security Group: 5432 허용 (모든 소스)
    NetworkPolicy:  5432 허용 (RDS 방향만)

→ 둘 다 허용하는 교집합만 통과
→ 더 제한적인 쪽이 실질적인 제어 기준이 됨
```


## Performance Insights가 뭔가요?
**RDS Performance Insights(PI)** 는 DB의 “느려짐/병목”을 볼 수 있게 **DB 엔진 내부 성능 데이터를 수집·시각화**해주는 기능입니다. Enhanced Monitoring이 **OS(호스트) 레벨**이라면, PI는 **DB 내부(쿼리/대기 이벤트) 레벨**에 초점이 있어요.

이 리포지토리에서는 아래 설정으로 켜져 있습니다.

```113:116:/Users/jounghyeon/개발/Url-Shortener-EKS-Platform/terraform/rds.tf
  performance_insights_enabled          = true
  performance_insights_retention_period = 7 # 보관 기간
```

## 어떤 걸 볼 수 있나요? (핵심)
- **DB Load**: 시간대별로 DB가 얼마나 바쁜지
- **Top SQL / Top waits**: 어떤 쿼리/대기 이벤트(락, I/O, CPU 등)가 부하를 만드는지
- **병목 원인 분해**: “CPU가 문제인지, I/O인지, 락 대기인지” 같은 원인 분석

## Enhanced Monitoring/CloudWatch랑 차이
- **Performance Insights**: 쿼리·대기 이벤트 중심(문제 원인 추적에 강함)
- **Enhanced Monitoring**: CPU/메모리/디스크/네트워크 같은 OS 메트릭 중심(인프라 상태 파악에 강함)

	두 가지 방식이 있습니다.

---

## Pull 방식 (기본)

```
Prometheus가 주기적으로 대상에게 직접 요청

Prometheus ──────────────→ 앱 Pod (/metrics)
           ← 메트릭 응답 ──
           (30초마다 반복)
```

- Prometheus가 수집 주체
- 앱은 `/metrics` 엔드포인트만 노출하면 됨
- 현재 이 프로젝트에서 사용 중인 방식

---

## Push 방식 (Pushgateway)

```
앱이 직접 Prometheus에 메트릭을 전송

앱 ──── 메트릭 push ────→ Pushgateway ←── pull ── Prometheus
```

- 앱이 수집 주체
- Pushgateway라는 중간 저장소가 필요
- Prometheus는 Pushgateway를 pull

---

## 언제 Push 방식을 쓰나

Pull 방식이 불가능한 경우에 사용합니다.

|상황|이유|
|---|---|
|배치 Job|실행 후 종료되어 Prometheus가 pull할 타이밍이 없음|
|방화벽 뒤 앱|Prometheus가 접근할 수 없는 네트워크에 있을 때|
|수명이 짧은 Pod|30초 수집 주기보다 Pod 수명이 짧을 때|

---

## Pull 방식이 기본인 이유

```
Push 방식의 문제점

앱이 다운됨 → Prometheus가 모름 (push가 안 올 뿐)
앱이 정상   → Prometheus가 모름 (push 안 하면 구분 불가)

Pull 방식
앱이 다운됨 → Prometheus가 /metrics 요청 → 응답 없음 → 장애 감지 ✅
```

> Pull 방식은 Prometheus가 능동적으로 상태를 확인하므로 **앱 다운 감지가 자연스럽게 됩니다.**
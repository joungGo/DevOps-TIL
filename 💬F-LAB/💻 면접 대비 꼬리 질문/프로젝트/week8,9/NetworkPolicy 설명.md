NetworkPolicy는 **Pod 간, 외부 간 트래픽을 화이트리스트 방식으로 제어**하는 Kubernetes 리소스입니다.

>NetworkPolicy는 **"침해당했을 때 피해 범위를 해당 Pod 안으로 가두는"** 역할입니다. 보안 사고는 막는 게 아니라 터졌을 때 확산을 막는 것이 핵심입니다.

```yaml
{{- if .Values.networkPolicy.enabled }}
# 🔒 NetworkPolicy — url-shortener 네임스페이스 트래픽 화이트리스트
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ .Release.Name }}-netpol
spec:
  podSelector:
    matchLabels:
      app: {{ .Release.Name }}-api
  policyTypes:
    - Ingress
    - Egress

  ingress:
    # 🔒 ALB에서 오는 트래픽만 허용 (aws-load-balancer-controller가 관리)
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - port: 8000
    # 🔒 Prometheus scrape 허용
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - port: 8000

  egress:
    # 🔒 RDS 접근 허용
    - ports:
        - port: 5432
          protocol: TCP
    # 🔒 Redis 접근 허용
    - to:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: redis
      ports:
        - port: 6379
          protocol: TCP
    # 🔒 DNS 허용 (필수)
    - ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
{{- end }}
```


---

## 기존과의 차이점

**NetworkPolicy 적용 전 (기본 상태)**

```
인터넷
  → ALB → url-shortener-api ✅
  
같은 클러스터 내 임의의 Pod
  → url-shortener-api ✅  (차단되지 않음)

url-shortener-api
  → RDS ✅
  → Redis ✅
  → 클러스터 내 다른 모든 서비스 ✅  (제한 없음)
  → 외부 임의의 서버 ✅  (제한 없음)
```

> 기본적으로 Kubernetes는 모든 트래픽을 허용합니다. 앱 Pod이 침해당하면 클러스터 내 다른 서비스나 외부로 자유롭게 통신할 수 있습니다.

---

**NetworkPolicy 적용 후**

```
인터넷
  → ALB → url-shortener-api ✅  (허용)

같은 클러스터 내 임의의 Pod
  → url-shortener-api ❌  (차단)

Prometheus(monitoring)
  → url-shortener-api:8000 ✅  (허용)

url-shortener-api
  → RDS:5432 ✅  (허용)
  → Redis:6379 ✅  (허용)
  → CoreDNS:53 ✅  (허용)
  → 클러스터 내 다른 서비스 ❌  (차단)
  → 외부 임의의 서버 ❌  (차단)
```

> 명시된 경로만 허용, 나머지는 전부 차단됩니다. 앱 Pod이 침해당해도 공격자가 이동할 수 있는 범위가 제한됩니다.

---

## 적용 대상

```yaml
podSelector:
  matchLabels:
    app: {{ .Release.Name }}-api    # url-shortener-api Pod에만 이 정책 적용
```

---

## Ingress (들어오는 트래픽)

### ALB → 앱 Pod

```yaml
- from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
  ports:
    - port: 8000
```

```
인터넷 → ALB → kube-system의 aws-load-balancer-controller
                                    ↓
                        url-shortener-api:8000
```

- ALB는 Pod IP로 직접 트래픽을 포워딩 (target-type: ip)
- 이 트래픽은 `kube-system` 네임스페이스에서 시작한 것으로 인식됨
- `kube-system`에서 오는 8000번 포트 요청만 허용

**기존과의 차이**: 기존에는 어느 네임스페이스에서든 8000 포트로 접근 가능했습니다. 적용 후에는 `kube-system`에서 오는 요청만 허용됩니다.

---

### Prometheus → 앱 Pod

```yaml
- from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring
  ports:
    - port: 8000
```

```
monitoring 네임스페이스의 Prometheus
  → 30초마다 url-shortener-api:8000/metrics/ scrape
```

- Prometheus가 `monitoring` 네임스페이스에 있으므로 해당 네임스페이스에서 오는 요청만 허용
- 이 규칙이 없으면 NetworkPolicy가 Prometheus scrape를 차단 → 메트릭 수집 안 됨

**기존과의 차이**: NetworkPolicy 없이도 Prometheus scrape는 동작했습니다. 적용 후에는 이 규칙을 명시하지 않으면 Prometheus가 차단되므로 반드시 추가해야 합니다.

---

## Egress (나가는 트래픽)

### RDS 접근

```yaml
- ports:
    - port: 5432
      protocol: TCP
```

- `to` 없이 `ports`만 지정 → 목적지 무관하게 5432 포트로 나가는 트래픽 허용
- RDS는 VPC 내 외부 서비스라 Pod IP로 선택할 수 없기 때문에 포트로만 제어
- RDS Primary, Replica 모두 5432이므로 하나의 규칙으로 커버

**기존과의 차이**: 기존에는 앱 Pod에서 어떤 포트로든 외부로 통신 가능했습니다. 적용 후에는 5432(RDS), 6379(Redis), 53(DNS) 외의 포트로 나가는 트래픽은 모두 차단됩니다.

---

### Redis 접근

```yaml
- to:
    - podSelector:
        matchLabels:
          app.kubernetes.io/name: redis
  ports:
    - port: 6379
      protocol: TCP
```

- Redis는 같은 네임스페이스의 Pod이므로 `podSelector`로 정확히 지정 가능
- RDS와 달리 목적지를 Redis Pod으로 한정 → 더 엄격한 제어

**기존과의 차이**: 기존에는 6379 포트로 클러스터 내 어떤 Pod에도 접근 가능했습니다. 적용 후에는 `app.kubernetes.io/name: redis` 라벨을 가진 Pod에만 접근 가능합니다.

---

### DNS 허용

```yaml
- ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
```

```
앱이 도메인을 IP로 변환할 때
  → CoreDNS(kube-system)에 DNS 쿼리 전송

DNS가 막히면
  → RDS 엔드포인트 resolve 불가 → DB 연결 실패
  → Redis 서비스명 resolve 불가 → 캐시 연결 실패
```

- UDP 53: 일반 DNS 쿼리
- TCP 53: 응답이 512바이트 초과할 때 자동으로 TCP로 재시도

**기존과의 차이**: 기존에는 DNS가 별도 설정 없이 동작했습니다. Egress 정책이 생기는 순간 DNS도 차단되므로, 이 규칙을 빠뜨리면 도메인 기반 연결이 전부 끊깁니다. **가장 흔한 실수입니다.**

---

## 전체 비교 요약

|트래픽|적용 전|적용 후|
|---|---|---|
|ALB → 앱:8000|✅|✅|
|Prometheus → 앱:8000|✅|✅ (명시 필요)|
|임의 Pod → 앱|✅|❌|
|앱 → RDS:5432|✅|✅|
|앱 → Redis:6379|✅|✅ (Redis Pod만)|
|앱 → DNS:53|✅|✅ (명시 필요)|
|앱 → 외부 HTTP(80/443)|✅|❌|
|앱 → 클러스터 내 다른 서비스|✅|❌|

---


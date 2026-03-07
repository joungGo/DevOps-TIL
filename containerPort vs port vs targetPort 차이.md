## 1. containerPort

**Pod 내부 컨테이너가 사용하는 포트**

즉

```text
컨테이너 애플리케이션이 실제로 열고 있는 포트
```

예

```yaml
containers:
  - name: my-app
    image: nginx
    ports:
      - containerPort: 80
```

의미

```text
컨테이너 내부에서 80번 포트 사용
```

주의

```text
containerPort는 문서적 의미가 강함
실제 네트워크 연결은 Service가 담당
```

---

## 2. port

**Service가 노출하는 포트**

즉

```text
Service IP로 접근할 때 사용하는 포트
```

예

```yaml
ports:
  - port: 80
```

의미

```text
Service:80 으로 접근
```

---

## 3. targetPort

**Service가 트래픽을 전달할 Pod 포트**

즉

```text
Service → Pod로 전달될 때 사용하는 포트
```

예

```yaml
ports:
  - port: 80
    targetPort: 8080
```

의미

```text
Service:80 → Pod:8080 전달
```

---

## 전체 흐름

```text
Client
  │
  ▼
Service:port
  │
  ▼
Pod:targetPort
  │
  ▼
Container:containerPort
```

예

```yaml
Service
port: 80
targetPort: 8080

Pod
containerPort: 8080
```

요청 흐름

```text
Client → Service:80 → Pod:8080 → Container
```

---

## 실제 YAML 예시

```yaml
apiVersion: v1
kind: Service

spec:
  selector:
    app: my-app

  ports:
    - port: 80         # Service 접근 포트
      targetPort: 8080 # Pod 포트
```

Pod

```yaml
containers:
  - name: app
    image: my-app
    ports:
      - containerPort: 8080
```

---

## 면접용 핵심 정리

```text
containerPort는 컨테이너 내부 애플리케이션 포트이고,
port는 Service가 외부에 노출하는 포트이며,
targetPort는 Service가 트래픽을 전달할 Pod의 포트이다.
```
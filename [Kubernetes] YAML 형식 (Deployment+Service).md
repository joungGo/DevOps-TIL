쿠버네티스에서 **가장 기본적으로 많이 사용하는 YAML 형식 (Deployment + Service)** 예시입니다.  
각 코드에 주석을 달았습니다.

```yaml
# ─────────────────────────────────────────
# Deployment: Pod를 관리하고 원하는 개수만큼 유지
# ─────────────────────────────────────────
apiVersion: apps/v1          # 사용할 쿠버네티스 API 버전
kind: Deployment             # 리소스 종류 (Deployment)

metadata:
  name: my-app-deployment    # Deployment 이름
  labels:
    app: my-app              # 리소스를 식별하기 위한 라벨

spec:
  replicas: 3                # 실행할 Pod 개수 (3개 유지)

  selector:
    matchLabels:
      app: my-app            # 이 라벨을 가진 Pod들을 관리

  template:                  # 실제 생성될 Pod의 템플릿
    metadata:
      labels:
        app: my-app          # Pod에 붙는 라벨 (selector와 반드시 일치)

    spec:
      containers:
        - name: my-app-container      # 컨테이너 이름
          image: nginx:latest         # 사용할 컨테이너 이미지

          ports:
            - containerPort: 80       # 컨테이너 내부에서 사용하는 포트

          resources:                  # 리소스 제한 (실무에서 많이 사용)
            requests:
              cpu: "100m"             # 최소 보장 CPU
              memory: "128Mi"         # 최소 보장 메모리
            limits:
              cpu: "500m"             # 최대 CPU
              memory: "256Mi"         # 최대 메모리
```

---

```yaml
# ─────────────────────────────────────────
# Service: Pod에 접근하기 위한 네트워크 엔드포인트
# ─────────────────────────────────────────
apiVersion: v1               # Service는 v1 API 사용
kind: Service                # 리소스 종류 (Service)

metadata:
  name: my-app-service       # Service 이름

spec:
  type: ClusterIP            # 서비스 타입 (기본값, 클러스터 내부 접근)

  selector:
    app: my-app              # 이 라벨을 가진 Pod로 트래픽 전달

  ports:
    - port: 80               # Service가 노출하는 포트
      targetPort: 80         # Pod 컨테이너의 포트
      protocol: TCP          # 통신 프로토콜
```

---

## 면접에서 말할 때 핵심 정리

**1. Deployment**

- Pod를 직접 관리하지 않고 **Deployment를 통해 Pod를 관리**
    
- 주요 기능
    
    - Pod 개수 유지 (`replicas`)
        
    - Rolling Update
        
    - 자동 복구
        

**2. Service**

- Pod는 **IP가 자주 바뀌기 때문에**
    
- **고정된 네트워크 접근 지점을 제공하는 리소스**
    
- Label Selector로 Pod를 찾아 **로드밸런싱**
    

---

## 실제 동작 흐름 (면접에서 좋아하는 설명)

1. Deployment가 Pod 3개 생성
    
2. Pod에 `app: my-app` 라벨이 붙음
    
3. Service가 `selector: app: my-app`으로 Pod 탐색
    
4. Service IP로 들어온 요청을 **Pod들로 라운드로빈 로드밸런싱**
    

---

원하면 **면접에서 진짜 많이 묻는 Kubernetes YAML 5가지도 정리해 줄게**

1. Deployment
    
2. Service
    
3. ConfigMap
    
4. Secret
    
5. Ingress
    

이 5개가 **실무 + 면접에서 가장 많이 사용하는 YAML**이다.
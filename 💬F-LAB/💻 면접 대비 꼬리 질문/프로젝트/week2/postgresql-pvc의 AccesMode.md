**AccessMode**는 👉 **하나의 스토리지를 여러 Pod가 “어떻게” 사용할 수 있는지 정하는 옵션**이야.

Kubernetes PVC에서 설정함.

---

## 📦 AccessMode 종류 3가지

### 1️⃣ ReadWriteOnce (RWO)

👉 **한 노드에서만 읽기/쓰기 가능**

- 가장 많이 사용됨 (DB용)
    
- 여러 Pod 가능하지만 **같은 노드에 있을 때만**
    
- 다른 노드에서는 mount 불가
    

```
Pod A (node1)  → OK
Pod B (node1)  → OK
Pod C (node2)  → ❌
```

✔ PostgreSQL / MySQL / Redis 기본 선택

---

### 2️⃣ ReadOnlyMany (ROX)

👉 **여러 Pod가 읽기만 가능**

- 쓰기 불가
    
- config / static 파일 공유용
    

```
Pod A → read
Pod B → read
Pod C → read
write → ❌
```

---

### 3️⃣ ReadWriteMany (RWX)

👉 **여러 Pod가 동시에 읽기/쓰기 가능**

- 공유 스토리지 (NFS 등 필요)
    
- 웹 서버 여러 replica에서 사용
    

```
Pod A → read/write
Pod B → read/write
Pod C → read/write
```

---

## 🎯 PostgreSQL이 RWO 쓰는 이유

DB는 동시에 여러 노드에서 쓰면 데이터 깨짐 ⚠️  
그래서 **하나만 쓰게 제한해야 함**

[[PVC - 다른 노드에서 다른 노드의 파드에 대해 읽고, 쓰기를 하려면 어떻게 해야할까?]]

---

## YAML 예시

```yaml
accessModes:
  - ReadWriteOnce
```

---

## 면접 한 줄 답변 🎤

👉 **AccessMode는 PVC를 여러 Pod가 동시에 읽기/쓰기 할 수 있는 범위를 정의하는 옵션이며, RWO·ROX·RWX 세 가지가 있다. DB는 보통 ReadWriteOnce를 사용한다.**

더 핵심 비교 필요하면 말해줘:

- RWO vs RWX 언제 쓰는지
    
- minikube에서 RWX 안되는 이유
    
- StatefulSet + AccessMode 관계
## replica 구성은 어떻게 해?

👉 **각 replica가 자기 디스크를 갖게 해야 함**  
→ 그래서 **StatefulSet + volumeClaimTemplates** 사용

구조:

```
postgres-0 → pvc-postgres-0 → disk-0
postgres-1 → pvc-postgres-1 → disk-1
postgres-2 → pvc-postgres-2 → disk-2
```

즉 replica가 **공유 디스크 쓰는 게 아니라**  
👉 **각자 독립 디스크 + DB replication 사용**

---

### StatefulSet replica 예시

```yaml
spec:
  replicas: 3
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
```

이렇게 하면 자동 생성됨:

```
postgres-storage-postgres-0
postgres-storage-postgres-1
postgres-storage-postgres-2
```

---


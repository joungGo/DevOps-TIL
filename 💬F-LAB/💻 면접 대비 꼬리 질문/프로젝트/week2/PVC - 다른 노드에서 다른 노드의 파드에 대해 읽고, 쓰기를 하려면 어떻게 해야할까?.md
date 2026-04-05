## 다른 노드에 있는 Pod도 읽고/쓰기 하려면?

👉 **`ReadWriteMany (RWX)` + 공유 스토리지 필요**

즉 단순히 AccessMode만 바꾸는 게 아니라  
**여러 노드가 동시에 접근 가능한 스토리지**를 써야 함.

대표 예시:

- NFS
    
- CephFS
    
- EFS (AWS)
    
- Azure Files
    

```yaml
accessModes:
  - ReadWriteMany
```

구조:

```
node1 ─ pod A ─┐
               ├── shared storage (NFS)
node2 ─ pod B ─┘
```

⚠️ 하지만 DB(PostgreSQL)는 RWX 쓰면 데이터 깨질 위험 큼  
→ DB는 보통 **RWO + replica 방식** 사용

[[PVC - replica 설계]]

---


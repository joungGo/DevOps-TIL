
### Kubernetes - Minikube

- 이미지 이름: `url-shortener-image:v1`

---
### Helm

- Deployment(API): `url-shortener-api`
- Service(API): `url-shortener-svc`
- Ingress(ingress.enabled=true일 때): `url-shortener-ingress`
- StatefulSet(Postgres): `url-shortener-postgres`
- Service(Postgres): `url-shortener-postgres-svc`
- PVC(Postgres): `url-shortener-postgres-pvc`
- Secret: `url-shortener-secret`
- Postgres Pod 이름(StatefulSet): `url-shortener-postgres-0` (replicas=1 기준)
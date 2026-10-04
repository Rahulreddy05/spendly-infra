# spendly-infra

Deployment for Spendly: Docker Compose for running the whole stack on one
machine, and Kubernetes manifests (Kustomize) for clusters.

Expects the app repos as siblings:

```
IdealProjects/
├── spendly-api/
├── spendly-web/
└── spendly-infra/   ← you are here
```

## Docker Compose (whole stack, one command)

```bash
cp .env.example .env        # fill in secrets: openssl rand -base64 48
docker compose up --build
open http://localhost:8080
```

Services: `db` (Postgres 16) → `migrate` (runs Prisma migrations, exits) →
`api` (port 4000) → `web` (nginx, port 8080, proxies `/api` to the API).

## Kubernetes

```
k8s/
├── base/                 # Deployments, Services, HPAs, PDBs, Ingress, NetworkPolicies
└── overlays/
    ├── local/            # 1 replica each, in-cluster Postgres, http://spendly.localhost
    └── prod/             # managed Postgres, TLS via cert-manager, registry images
```

### Local cluster (Docker Desktop → Settings → Kubernetes → Enable)

```bash
# Images: build locally; Docker Desktop's cluster can use them directly.
docker build -t spendly-api:local ../spendly-api
docker build -t spendly-web:local ../spendly-web

# An ingress controller (once per cluster).
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.0/deploy/static/provider/cloud/deploy.yaml

cd k8s/overlays/local
cp secrets.example.env secrets.env && cp db-secrets.example.env db-secrets.env   # fill in
kubectl apply -k .
kubectl -n spendly get pods -w
open http://spendly.localhost
```

### What the manifests do

| Concern | How |
| --- | --- |
| Zero-downtime deploys | Rolling updates with `maxUnavailable: 0`, readiness gates, `preStop` drain |
| Migrations | `migrate` init container runs `prisma migrate deploy` before the new API serves |
| Scaling | HPA on CPU (API 2–8, web 2–6), scale-down stabilisation to avoid flapping |
| Availability | PodDisruptionBudgets; replicas spread across nodes |
| Health | startup/liveness on `/health/live`, readiness on `/health/ready` (checks DB) |
| Right-sizing | CPU/memory requests and limits on every container |
| Security | `restricted` Pod Security Standard, non-root, read-only root FS, all capabilities dropped, no SA token |
| Network | Default-deny ingress; only the ingress controller reaches web/API; only the API reaches Postgres |
| Secrets | `Secret`s from git-ignored env files (prod: use External Secrets / Sealed Secrets) |
| Routing | One origin: Ingress sends `/api` to the API and `/` to the web app — no CORS |

### Plaid (bank linking)

- Keys: `PLAID_CLIENT_ID` and `PLAID_SECRET` go in the Secret (sandbox keys locally,
  production keys only in the prod Secret). Non-secret settings are in the API ConfigMap.
- Webhooks: `PLAID_WEBHOOK_URL` must be the public `https://<host>/api/v1/webhooks/plaid`.
  Plaid signs every webhook (ES256 JWT); the API verifies it, so the endpoint needs no other auth.
- OAuth banks: register `PLAID_REDIRECT_URI` (`https://<host>/oauth-return`) and the
  Android package name `com.rahulreddy05.spendly` under Dashboard → API → Allowed redirect URIs / Android package names.
- Locally there are no webhooks; use **Sync now** in the app.
- The web CSP allows Plaid Link (`cdn.plaid.com`); the API needs outbound HTTPS to `*.plaid.com`.

## Standards

See [ENGINEERING_STANDARDS.md](ENGINEERING_STANDARDS.md).

# Spendly Infra

@ENGINEERING_STANDARDS.md

Deployment for Spendly: `docker-compose.yml` (full local stack) and Kustomize
manifests in `k8s/` (`base` + `overlays/local`, `overlays/prod`). App code lives
in the sibling repos `../spendly-api` and `../spendly-web`.

## Rules for this repo

- Any new API env var: add it to `k8s/base/api-config.yaml` (non-secret) or the
  overlays' `secrets.example.env` (secret), and to `docker-compose.yml` / `.env.example`.
- Every workload keeps: requests+limits, probes, restricted securityContext,
  PDB, HPA (if stateless), NetworkPolicy coverage.
- Never commit `secrets.env`, `db-secrets.env` or `.env` — they are git-ignored.
- Validate before committing:
  `kubectl kustomize k8s/overlays/<name> | kubeconform -strict -summary`
  and `docker compose config --quiet`.

# Homelab session — Context

> Scope: anything that runs on the homelab Kubernetes cluster.
> Out of scope: Mac Studio inference, GCP frontend.

## What lives here

- Airflow (`crawl_<community>` DAGs, `daily_digest` DAG)
- Postgres 16 (single instance, hostPath PV)
- Elasticsearch 8 (single node, Korean analyzer)
- Argo CD (watches `manifests/` repo)

## Data model (read-only contract)

```
posts(id, community, source_url, title, body,
      posted_at, fetched_at, nsfw, summary)
```

If you need to change this, write a migration in `db/migrations/` and update
the master CLAUDE.md too.

## External endpoints this machine talks to

- **Mac Studio LLM** — `http://mac-studio.tailnet:8080/v1/...`
  - Used by `daily_digest` DAG for NSFW classification and summarization.
- **GCP VM** — read-only access to Postgres/ES via Tailscale, for the frontend.

## Conventions for this session

- Manifests live in the `manifests/` repo, NOT in app repos.
- GitHub Actions on app repos must commit a tag bump into `manifests/` so Argo
  picks it up — never `kubectl apply` directly.
- `kubectl` context is `homelab-prod` by default.
- Resource limits: profile the pod first, then set requests = p95, limits = 2×.

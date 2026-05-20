# GCP VM session — Context

> Scope: the public-facing web frontend and reverse proxy on the GCP free-tier VM.
> Out of scope: data ingestion, LLM inference, K8s.

## What lives here

- Next.js frontend (systemd service, listens on `127.0.0.1:3000`)
- nginx (TLS termination, reverse proxy to the Next.js service)
- Let's Encrypt certs via certbot, auto-renew on a daily timer
- Tailscale (joins the home tailnet so the app can reach Postgres / ES)

## Data sources

- **Postgres** (`homelab.tailnet:5432`) — primary list view, read-only user.
- **Elasticsearch** (`homelab.tailnet:9200`) — search endpoint, read-only.

If either source is unreachable, the frontend should serve a friendly empty
state, not 500.

## Conventions for this session

- This is the only machine with a public IP. Everything else stays internal.
- Don't store secrets in `.env` files on the VM. Pull at boot from
  a small `gcloud secrets` lookup.
- App deploys via `git pull && npm run build && systemctl restart app.service`.
  This box is intentionally outside the K8s/Argo loop.
- Keep nginx config in version control; copy from `infra/gcp/nginx.conf` on
  any change.

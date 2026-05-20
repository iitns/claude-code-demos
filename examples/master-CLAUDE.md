# Personal Community Aggregator — Master Context

> The single source of truth for the whole project. Each per-machine session
> gets a slimmed-down copy derived from this file.

## Goal

A personal aggregator that:

1. Crawls a fixed set of online communities on a schedule.
2. Lands posts into Postgres and indexes them into Elasticsearch.
3. Runs an NSFW classifier and a two-pass summarizer on a local LLM.
4. Sends a daily digest to Telegram.
5. Exposes a web frontend (list + search) to me on the public internet.

## Infrastructure

| Location       | Box              | Role                                                  |
|----------------|------------------|-------------------------------------------------------|
| Home network   | MacBook (dev)    | Where I drive Claude Code and Codex from.             |
| Home network   | Homelab (K8s)    | Crawlers, Postgres, Elasticsearch, Airflow.           |
| Home network   | Mac Studio       | Local LLM inference (NSFW + summary).                 |
| External       | GCP free-tier VM | Public web frontend, reverse proxy, health checks.    |

All four machines are stitched together by Tailscale, so any node is reachable
from any other by its short hostname.

## Data model (Postgres)

```
posts(
  id            bigserial primary key,
  community     text not null,
  source_url    text not null unique,
  title         text not null,
  body          text,
  posted_at     timestamptz not null,
  fetched_at    timestamptz not null default now(),
  nsfw          boolean,         -- null until classifier runs
  summary       text             -- null until digest DAG runs
);
```

## Platforms

- **Airflow** (homelab) — two DAG families:
  - `crawl_<community>` — per-community crawler, hourly
  - `daily_digest` — runs at 06:00 local, calls Mac Studio LLM, pushes Telegram
- **Postgres 16** (homelab) — single instance, hostPath PV
- **Elasticsearch 8** (homelab) — single node, Korean analyzer
- **Local LLM** (Mac Studio) — exposed as an HTTP endpoint over Tailscale
- **Frontend** (GCP VM) — Next.js app, reads Postgres for lists, ES for search
- **Argo CD** (homelab) — watches `manifests/` repo, syncs cluster

## Deployment loop

App repos → GitHub Actions → image registry + bump tag commit into
`manifests/` repo → Argo CD watches `manifests/` → cluster reconciles.

## Conventions

- Secrets via Kubernetes Secret resources, never in env vars in the manifest
  YAML. All values originate from `.env.local` on the MacBook.
- All times stored as UTC, rendered in `Asia/Seoul` at the frontend.
- Per-community parsing is generic by default — only specialize when a site
  forces it.
- Don't change shared schema without a migration script in `db/migrations/`.

## Where work runs

- **Homelab session** — K8s manifests, Airflow DAGs, Postgres, Elasticsearch.
- **Mac Studio session** — local LLM service, model swaps, inference endpoint.
- **GCP session** — frontend deploy, nginx, certs, public DNS.

Each per-machine session gets a derived CLAUDE.md scoped to just its slice.

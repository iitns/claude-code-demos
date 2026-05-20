# Mac Studio session — Context

> Scope: the local LLM serving stack on the Mac Studio.
> Out of scope: anything outside this box.

## What lives here

- llama.cpp server (or equivalent) bound to `0.0.0.0:8080` on Tailscale only.
- A pool of GGUF model files in `~/models/`.
- `launchd` agent that keeps the server up across reboots.

## Endpoints exposed

| Path                | Used by                  | Notes                                |
|---------------------|--------------------------|--------------------------------------|
| `POST /v1/classify` | Airflow `daily_digest`   | Returns `{ nsfw: bool }` for a post. |
| `POST /v1/summary`  | Airflow `daily_digest`   | Per-community or cross-community.    |

Both are JSON in, JSON out. Don't break the contract — the DAG depends on it.

## Conventions for this session

- Don't pull models down at request time. Pre-stage in `~/models/`.
- Model swaps require restarting the launchd service —
  `launchctl kickstart -k gui/$(id -u)/local.llm.server`.
- This box has no public exposure. All callers are on Tailscale.
- If GPU memory is tight, prefer a smaller quant over a smaller model.

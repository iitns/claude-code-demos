# Overview
- A project to prepare a presentation that shares my Zero-to-One experience using Claude Code.

# Goal
- A 15–30 minute slide deck, in English.
  - Can be split into two ~15-minute halves.

# Audience
- Junior to Senior software engineers, including one engineering manager.
- Most likely they've only used Claude Code through their company account for company work.
- They use Claude Code daily as a baseline.

# Ground rules
- All slides must be in English.
- The script must also be prepared in English.
- Focus on sharing learnings and knowledge.
- Do not get distracted by engineering decisions (why this tool over that one). Stay focused on the *process* of building things.

# Format
- HTML.
  - Some image files or videos will be attached later by me. Keep placeholders for the media.
- Keep the script aside (separate from the deck itself).
- Keep the text as low as possible. Lean on pictures and diagrams instead.
- Any framework is fine — pick whatever expresses the content best. It'll be hosted on the internal company GitHub, so anything reasonable works.

# Content

## Overview
- Show what we're going to do.
- What outcome was needed, what had to be in place to achieve it, what infrastructure I already had, and how Claude Code was used inside that.
- What helped most: planning the work at a high level, splitting work by machine, and splitting tasks across agents so they don't overlap.

## Motivation
- Over meals or quick breaks I scroll YouTube Shorts, but I also like checking what's going on out there — so I find myself digging through posts.
- Because communities are scattered, hopping between sites is time-consuming and tedious. I wanted everything in one place.
- Once you have the data collected, you find new things to do with it.

## Infra
- Setup:
  - A homelab running Kubernetes, with Argo CD already configured.
  - A Mac Studio for local LLM inference.
  - A free-tier VM running on GCP for health checks (and other always-on needs).
- They all needed to be reachable from each other, so a VPN was required.
- For this section, draw a diagram: the dev MacBook, the homelab, and the Mac Studio sit in the home network; GCP sits in the external network; everything is connected through a VPN.
  - The diagram will be reused later — make it good. I'll later overlay Claude Code and Codex attaching to these boxes via SSH; I want the visual to look like an "agent riding on top of the computer" doing the work directly.

## Architectural design
- Provided the context in CLAUDE.md: what's needed, what infra I have, which platforms are required, which data stores are needed.
- Specified the platforms to use: Airflow, Postgres, Elasticsearch, a frontend page.
- Asked Claude Code to split the tasks by machine so I could parallelize the work across multiple sessions.

## Operations
- DB
  - Provided the crawl schema and the storage schema. Helps Claude understand both the crawler code and the DB schema.
  - Claude executes queries for DB schema changes, data pruning, and normalization/denormalization.
- Deployments: synced with GitHub and Argo CD for automated deployment.
  - A code push triggers a GitHub Action, which builds the Docker image and pushes it to the Docker/GitHub registry. Argo CD then syncs.
  - But sometimes the agents need to SSH into servers to manually update and deploy.

## Working with multiple agents
- Claude Code and Codex.
  - Claude Code: infrastructure and operations.
  - Codex: writing the Airflow DAGs and the frontend app. Early on, Codex had to wait until Claude Code finished the infra setup.
- An animation built from the infra diagram would help here:
  - From the dev MacBook, when I direct Claude Code, it SSHs into other machines like the homelab or Mac Studio to do the work.
  - Codex pulls the code from the GitHub repo, modifies it, and pushes it back. Once the pushed code is built and stored in the registry, Argo CD syncs it and the new code gets deployed.

## A few notes
- Many things didn't work at first.
  - Servers came up on Kubernetes fine, but resources were under-allocated.
  - In Elasticsearch, the term-extraction option wasn't set, so search wasn't working properly.
  - Airflow DAGs weren't parallelized, or were misconfigured so that a single failure broke the next step.
  - Logs weren't being captured properly.
  - Crawls weren't collecting properly, etc.
- For per-community collection logic, I didn't hand-write parsers — I just gave the URLs and Claude parsed the post lists on its own.
  - Time-zone issues did come up, so I refined the timestamp storage format, fixed HTML selectors for body extraction, and so on.

## After collection
- I can now use a local LLM to produce a daily summary.
- For each community I pull the top-10 most popular posts, summarize them, and then summarize those summaries — so I get yesterday's topics delivered to Telegram.
- I don't have to spend time catching up to know what happened; the content from the domains I care about comes to me.

## Applications
- I can ingest financial news to produce a summarized news brief — this needs a data-collection pipeline.
- I can translate overseas news and turn it into YouTube videos.

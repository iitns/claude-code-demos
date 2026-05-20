# Speaker Script — Zero to One with Claude Code

**Total runtime:** ~25–28 minutes
**Split point:** end of Section 5 (Operations) — natural break for two 15-min halves.
**Tone:** conversational. Don't read line by line — use these as memory anchors.

---

## Title (~30s)

> Hi everyone. I'm Janghan. Today I want to share a side project I built — a personal aggregator that crawls a bunch of online communities and sends me a morning digest. The system itself isn't the point of the talk. The point is *how* I built it, working with Claude Code and Codex as collaborators. So this is a Zero-to-One story, but the focus is on the workflow, not the code.

---

# PART ONE — Design & infrastructure (~14 min)

## 1. Overview (~2 min)

### 1.1 What we're going to build
> Let me set the destination first. By the end of this talk you'll know what got built, with what pieces I already had, and where Claude Code actually fit in. We'll move through motivation, infra, the design phase, ops, the agent split, and lessons. There's an outcome section and an "applications" section at the end.

### 1.2 What did I need?
> Three things I wanted. Collect posts from multiple online communities on a schedule. Store them somewhere I can list and search. Summarize them and deliver a digest to my phone. Notice these are *outcomes* — not engineering choices. Keeping the outcome separate from the implementation made the conversation with Claude Code much cleaner.

### 1.3 What I already had
> I didn't start from zero on infra. I had a homelab running Kubernetes with Argo CD already syncing from GitHub. I had a Mac Studio that was mostly idle, perfect for local LLM inference. I had a free-tier GCP VM that's always on. And of course Claude Code — and a willingness to actually let it drive.

### 1.4 What helped most
> If you remember only one slide from this talk, make it this one. Four things made the difference.
>
> First — plan big, then split into *agent-shaped* tasks. The trick isn't that you plan; it's *how* you slice. Each task should end up being either an operation, or a piece of code — never both at once. That keeps each agent's purpose unambiguous, and it's also how I dodged content rot as the project grew.
>
> Second — split by machine. Same principle, applied to hardware. One session per box keeps state local. Decisions for the homelab don't pollute decisions for the Mac Studio.
>
> Third — split by agent *type*. Ops tasks go to Claude Code, because it lives in the terminal. Code tasks go to Codex, because it lives in the repo workflow. Right tool, right lane.
>
> Fourth — and this is the humility one — ship the bare minimum first, then iterate. My v1 goal, literally, was "works enough that I could demo this from a United flight." Wi-fi flaky, no help from anyone. It mostly *didn't* work that first version. But every refinement after was driven by a real failure I'd seen — not by speculating about what might go wrong. That's the loop that kept things moving.

## 2. Motivation (~2 min)

### 2.1 Why I built this
> Quick personal motivation. This isn't a work project. It's a scratch-your-own-itch one. That matters because it gave me freedom to experiment, to fail, to iterate without anyone breathing down my neck.

### 2.2 The everyday scene
> Picture lunch break. Coffee. Phone in hand. I'd watch some shorts, sure, but I also wanted to know what was *actually* happening that day in the communities I follow — tech, finance, hobbies. They all live in different places. [Point at the screenshot.] That's the aggregator's home view now. Posts from every community in one stream, ranked by recency.

### 2.3 The frustration
> Two pains. Fragmented — every community is its own site. And time-expensive — I'd hunt through threads to find the few that actually mattered. The thought was simple: if I just had everything in one place, I could decide what's worth my attention.

### 2.4 The bonus realization
> Here's the part I didn't fully appreciate at the start. Once the data was collected, new uses appeared on their own. Daily digests. Trend signals. Eventually translation and video. The collection pipeline became a foundation, not an endpoint.

## 3. Infra (~2.5 min)

### 3.1 The machines I had
> Three locations, one VPN, no rented servers. Before any code, this is the physical reality. The shape of the hardware shaped how Claude Code worked across the whole project.

### 3.2 The hardware
> Each box has exactly one job. Homelab is compute. Mac Studio is inference. GCP is the always-reachable foothold from the outside world. Separating responsibilities by hardware made it much easier to reason about state.

### 3.3 How they're connected
> [Walk through the diagram.] On the left, my home network — the MacBook, the homelab, the Mac Studio. On the right, external — the GCP VM. They're all stitched together by Tailscale in the middle. Notice the little agents riding on top of each box. That's the visual: the agent is the thing actually doing work *on* the box. The MacBook is where Claude Code lives, and from there it SSHs into any machine as if it were local.

### 3.4 Why the VPN matters
> Without a flat addressable network, multi-machine agent work is painful. The VPN is what makes "one agent, many machines" actually work. One SSH config, every node reachable, no public exposure of homelab services. This was load-bearing for the whole project.

## 4. Architectural Design (~4 min)

### 4.1 From idea to design doc
> Before any code, I wrote out the context in a CLAUDE.md file. Same pattern this very talk is built with, by the way — these slides were also bootstrapped from a CLAUDE.md.

### 4.2 What I gave Claude Code up front
> Four things. The goal — collect, store, search, summarize, notify. The inventory — the hardware I just showed. The platform picks — Airflow, Postgres, Elasticsearch, a small web frontend. And constraints — stay on existing infra, no paid SaaS. Notice: I picked the platforms. I didn't ask Claude what to use. That's where you waste hours debating. Hand it the bag of tools and say "design for these."

### 4.3 The system, end-to-end
> [Walk diagram.] Two flavors of Airflow DAG run on the homelab. The crawler DAGs pull from each community and land rows in Postgres. From Postgres they're indexed into Elasticsearch. The frontend lives on the GCP VM — it reads Postgres directly for the listings, and hits Elasticsearch for the search feature. A separate digest DAG pulls yesterday's window from Postgres, calls the local LLM on the Mac Studio for NSFW classification and summarization, and pushes the result to Telegram — all from within the same DAG. The Mac Studio is just an inference endpoint here, not an orchestrator.

### 4.4 The split that unblocked me
> Here's the prompt that changed everything: "split this plan by machine, so I can run sessions in parallel." Suddenly I had a homelab session, a Mac Studio session, and a GCP session — each with its own scope, its own context, its own todo list. The token budget thanks you. Your sanity thanks you. And the sessions can actually run in parallel.

### 4.5 How the split actually happens
> Concretely: I keep one master CLAUDE.md in the repo root. It captures the whole project — goal, infra inventory, data schema, platforms, conventions, and a short "where work runs" section.
>
> Then I run one prompt — literally "split this by machine and produce one CLAUDE.md per session" — and I get three derived files. Homelab, Mac Studio, GCP.
>
> Each session only loads its own slice. The homelab session never sees the nginx config. The Mac Studio session never sees Argo CD. The GCP session never sees the data schema.
>
> Three concrete benefits. Faster context — no irrelevant tokens taking up space. Less content rot — changes to the master don't accidentally regenerate stale assumptions in the per-machine files until I explicitly ask. And clearer accountability — when something goes wrong, I know exactly which file the agent was reading from.

## 5. Operations (~3 min)

### 5.1 Running it day-to-day
> Design is a one-off. Ops is forever. This is where Claude Code stopped being a code generator and became an actual operator.

### 5.2 Database work
> The schema lives in CLAUDE.md. Migrations are generated by Claude, I review, they run against Postgres. Pruning, normalization, dedupe — all just queries Claude writes and I approve. Big win: crawl code and query code stayed in sync because both were generated from the same documented schema.

### 5.3 Deployments — the happy path
> [Walk diagram.] This is a GitOps loop, and the two-repo structure matters. The app code lives in one repo. The Kubernetes manifests live in a *separate* repo. When I push to the app repo, GitHub Actions builds the image and does two things — pushes the image to the registry, and commits the new image tag back into the manifest repo. Argo CD is only watching the manifest repo; it doesn't poll the registry. It sees the new commit, syncs the cluster, the pod rolls. The cluster's desired state always lives in Git, which means re-syncs, rollbacks, and audits are just git operations.

### 5.4 The other path — SSH
> Not everything fits the Git-driven loop. Sometimes I need to bump a stuck pod's resources, run a one-off DB hotfix, swap an LLM model on the Mac Studio, or tweak nginx on the GCP VM. Claude Code with SSH access is, honestly, the cleanest on-call experience I've had. Because of the VPN, every machine is one hop away.

---

### *[SPLIT POINT — natural break for two 15-min halves]*

---

# PART TWO — Multi-agent workflow & outcomes (~13 min)

## 6. Multi-Agent (~4 min)

### 6.1 Two agents, different jobs
> This is the part of the talk I find most fun. I had two coding agents working at the same time, and the trick was giving them different jobs.

### 6.2 The division of labor
> Claude Code did infrastructure and ops. SSH into machines, edit manifests, run kubectl, patch DAGs in place, debug pods. Codex did application code. Airflow DAG implementations, the frontend app, pure-code repos, PRs into GitHub. Early on Codex had to wait — Claude needed to set the infra up first. But once the foundation was there, both ran in parallel.

### 6.3 What it looks like in motion
> [Walk the animated diagram.] Step one — Claude Code SSHs from the MacBook into the homelab or the Mac Studio for ops work. Step two — in parallel, Codex pushes to the app repo. Step three — Actions builds the image and pushes it to the registry. Step four — the same Action commits a tag bump into the manifest repo. That's the GitOps trigger. Step five — Argo CD is only watching the manifest repo; it sees the new commit. Step six — the cluster reconciles, pod rolls. The MacBook is the orchestrator. I'm just routing tasks between the two agents.

### 6.4 Why the split worked — and where it bites
> Let me do this honestly, because it's not free.
>
> Upsides first. No collision — they touch different layers, never the same file. Right tool for the layer — Claude lives in the terminal, Codex lives in the repo workflow. Easier review — Codex's diffs hit GitHub, Claude's changes hit the cluster. Recoverable — Argo CD is the truth, so if anything drifts, I just re-sync. And one practical one — token budget. Daily and weekly rate limits hit half as fast when two providers share the load.
>
> Downsides. Two of them. First — if you accidentally let both agents touch the same area, you get conflicts. Either a git merge conflict, or worse, drift between what each agent *thinks* the world looks like. Second — they don't share memory. Each agent's notes and memory live in its own session. If Claude just patched a deployment, Codex has no idea unless it goes and looks.
>
> The mitigation is the same in both cases: don't trust either agent's mental model. Have it re-read the source of truth — the code repo, the live cluster, the actual state — at the start of any non-trivial change.

## 7. Hiccups & Lessons (~3 min)

### 7.1 It did not work the first time
> Honest section. This is not a "prompted once and shipped" story.

### 7.2 What broke early on
> Four real failures. Kubernetes pods came up but were under-provisioned — OOMs and throttling. Elasticsearch was missing a term-extraction option, so a chunk of my docs were invisible to queries. Airflow DAGs weren't parallelized, so one community failing cascaded into the next day's run. And crawls were silent — no logs, so failed collections looked like empty days. None of these were surprising in hindsight. They were just invisible until I looked. Claude helped triage each one — but the human still has to *look*.

### 7.3 What worked surprisingly well
> Five wins, and the last three were genuine surprises.
>
> *Generic crawlers* — I gave Claude only the URLs. It figured out list parsing for each community on its own, without per-site code.
>
> *Schema-driven prompts* — once the DB schema lived in CLAUDE.md, the crawler code and the query code stopped drifting apart.
>
> *Recovers broken systems* — one of the homelab nodes rebooted on its own. Claude couldn't tell me *why* the host went down — that's out of its reach. But it noticed a Postgres data disk failed to remount, edited /etc/fstab, brought the mount back, restarted Postgres. End-to-end recovery without me touching the keyboard.
>
> *Handles gnarly K8s commands and manifests* — the long kubectl incantations, the fiddly manifest fields like hostPath mounts, GPU mounts, secrets. These are the parts I always have to look up. Claude just does them.
>
> *Reads error logs and patches the code* — on a failure, it tails the logs, isolates the offending stack frame, and edits the code directly. No need to translate the failure into a prompt.

### 7.4 Refinements that mattered
> Timestamp normalization — store UTC, render local. Selector tuning — extract body, not chrome. DAG isolation — one community fails, the rest keep running. Right-sizing resources by observing actual usage. And UI polish — search bar wired up to Elasticsearch, dark mode, filters by community and tag. Each one was a 30-minute conversation, not a 3-day project. That's the speed multiplier you get from this workflow.

## 8. Outcome (~3 min)

### 8.1 The result that lands on my phone
> Here's the payoff. From scattered communities to one morning digest.

### 8.2 Local LLM — two jobs
> [Walk diagram.] Once the local LLM was sitting on the Mac Studio, it was easy to give it more than one job.
>
> Job one — classification. For every new post, the model looks at the title and the body and decides whether it's NSFW. The flag goes back into Postgres alongside the post.
>
> Job two — summarization. Same model, two passes. Per-community first, then a meta-summary across all of them. The NSFW filter feeds the top-10 selection, so the digest never includes anything I'd rather not have on my phone over breakfast.
>
> The lesson here: the local LLM stopped being a "summarizer." It became a generic inference endpoint I could keep adding tasks to.

### 8.3 What lands on my phone
> One Telegram message in the morning. Top topics across my chosen communities, with links if I want to dive deeper. No catch-up scrolling, no FOMO. I'll show you a real screenshot here on the actual demo day.

### 8.4 What changed for me
> Four things.
>
> Time — no more 20-minute lunch-hour scroll.
>
> Coverage — I see more communities than I used to visit manually.
>
> Signal — pre-ranked by what humans found engaging, not by an algorithmic feed.
>
> And one that changed how I read the web. Discovery via digest. I don't dig through whole threads anymore. The Telegram digest surfaces the popular posts, and I only follow the ones that look worth my time. The model flipped — from hunting, to triaging a curated list.

## 9. Applications (~2 min)

### 9.1 What this unlocks next
> The collection pipeline is the foundation. Everything else is a remix.

### 9.2 Adjacencies
> Two examples. A finance digest — swap the source list for financial communities and news outlets, reuse the entire stack. And translated video shorts — pull foreign-language stories, translate, generate narration, stitch into short videos. Same upstream pipeline, new downstream module.

### 9.3 Takeaways
> Five things to remember.
>
> Plan first, code second — CLAUDE.md is your contract with the agent.
>
> Split by machine — one session per box keeps state clean.
>
> Split by agent — Claude Code for ops, Codex for code.
>
> Trust the loop — Argo CD deploys and SSH hotfixes can coexist.
>
> Value the data layer — once collection runs, every new feature is days, not months.

### 9.4 Closing
> What would *you* build with this pattern? The Kubernetes piece is optional. The agent-split is the durable part. Thanks — happy to take questions.

---

## Timing buffer & cuts

- If running long: skip 9.1 and merge 9.2 directly into 9.3.
- If *really* long: cut 2.4 — the realization re-lands at the start of §9.
- If audience is junior-heavy: spend longer on the animated diagram in §6.3 — that's the conceptual core.
- If audience is senior-heavy: spend longer on §4.5 (CLAUDE.md fan-out) and §6.4 (downsides) — those land hardest with experienced engineers.

## Demo backup (if there's time)

- Live Telegram message from this morning.
- Live frontend (the search UI over Elasticsearch on the GCP VM).
- A `kubectl get pods` from a Claude Code session — to show ops in action.
- `cat examples/master-CLAUDE.md` followed by one of the derived files — make the fan-out concrete.

## Numbers to mention if anyone asks

- ~4 machines, 1 tailnet, ~6 weeks from idea to first Telegram message.
- ~20 minutes per refinement on average — the iteration loop is the win.
- Free GCP tier is enough for the frontend; everything else is hardware I already owned.

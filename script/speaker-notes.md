# Speaker Script — Zero to One with Claude Code

**Total runtime:** ~25–28 minutes
**Split point:** end of Section 5 (Operations) — natural break for two 15-min halves.
**Tone:** conversational. Don't read line by line — use these as memory anchors.

---

## Title (~45s)

> Hi everyone. I'm Janghan. Today I want to share a side project I built — a personal aggregator that crawls a bunch of online communities and sends me a morning digest.
>
> Before we start, one quick framing note. **This talk is about how I used Claude Code to go from zero to one — the workflow, the prompts, the way work was split across agents and machines. It is *not* a discussion of the engineering decisions inside the project.** I'm not going to debate Postgres over MongoDB, Airflow over Prefect, two-pass summarization, and so on. Those choices are in the talk, but I'm treating them as given. If you want to dig into any of them, come find me after — happy to chat. For the next 25 minutes, the lens is squarely on *the build process with Claude*, not the architecture itself.

---

# PART ONE — Design & infrastructure (~14 min)

## 1. Overview (~2.5 min)

### 1.1 What we're going to build
> Let me set the destination first. By the end you'll know what got built, with what pieces I already had, and where Claude Code actually fit in. We move through motivation, infra, the design phase, ops, the agent split, and lessons. There's an outcome section and an "applications" section at the end.

### 1.2 What did I need?
> Three things I wanted. Collect posts from multiple online communities on a schedule. Store them somewhere I can list and search. Summarize them and deliver a digest to my phone.

### 1.3 What I already had
> I didn't start from zero on infra. I had a homelab running Kubernetes with Argo CD already syncing. I had a Mac Studio that was mostly idle — perfect for local LLM inference. I had a free-tier GCP VM that's always on with a public IP, so it's the natural home for anything that needs to be reachable from outside the home network. And of course Claude Code *and* Codex — two coding agents.

### 1.4 What helped most
> Four things to call out. The top row is the conceptual pair — *how to think*. The bottom row is the concrete mechanics — *how to execute*.
>
> **One — plan big, then split into smaller chunks.** Plan the whole system, then slice the plan into agent-shaped tasks: one session per machine, one task per agent, and each chunk is either an *operation* or a piece of *code* — never both. Smaller surface area means cleaner state and fewer context-bleed surprises. This is how I ran homelab, Mac Studio, and GCP work in parallel without them tangling, and it's the single biggest reason content didn't rot as the project grew.
>
> **Two — ship the bare minimum first, then iterate.** This is the humility one. My v1 goal, literally, was "works enough that I could demo this from a United flight." Wi-fi flaky, no help from anyone. It mostly *didn't* work that first version. But every refinement after was driven by a real failure I'd seen — not by speculating about what might go wrong.
>
> **Three — pre-write a concrete CLAUDE.md.** The more specific your spec, the closer the first output. You *can* refine by ping-ponging with Claude or Codex, but that burns time and tokens fast — and the gap between "rough idea" and "Claude can act on this" is exactly what a planning model is good at closing. So draft the CLAUDE.md ahead of time. Honestly: I use ChatGPT or Gemini for that draft pass, then hand the result to Claude. Different tools, complementary jobs.
>
> **Four — keep agents' lanes from crossing.** When you're running multiple agents in parallel — Claude Code on one side, Codex on the other — make sure their work areas don't overlap. Two agents editing the same file or touching the same cluster object gets you merge conflicts at best, silent drift between mental models at worst. Draw the lanes explicitly, and re-check them when scope changes.

## 2. Motivation (~2 min)

### 2.1 Why I built this
> Quick personal motivation. This isn't a work project. It's a scratch-your-own-itch one. That matters because it gave me freedom to experiment, to fail, to iterate without anyone breathing down my neck.

### 2.2 The everyday scene
> Picture lunch break. Coffee. Phone in hand. I'd watch some shorts, sure, but I also wanted to know what was *actually* happening that day in the communities I follow — tech, finance, hobbies. They all live in different places. [Point at the screenshot.] That's the aggregator's home view now. Posts from every community in one stream, ranked by recency.

### 2.3 The frustration
> Two pains. Fragmented — every community is its own site. And time-expensive — I'd hunt through threads to find the few that actually mattered. The thought was simple: if I just had everything in one place, I could decide what's worth my attention.

### 2.4 The bonus realization
> Here's the part I didn't fully appreciate at the start. Once the data was collected, new uses appeared on their own — two so far. The daily digest, summarized by the local LLM overnight. And a sense of which topics are heating up across communities. I'm being honest on this slide: there's more downstream I haven't built yet — translation and video pipelines come to mind — but I'll talk about those at the end, not here.

## 3. Infra (~2.5 min)

### 3.1 The machines I had
> Three locations, one VPN, no rented servers. Before any code, this is the physical reality. The shape of the hardware shaped how Claude Code worked across the whole project.

### 3.2 The hardware
> Each box has exactly one job. Homelab is compute. Mac Studio is inference. GCP is the always-reachable foothold from the outside world. Separating responsibilities by hardware made it much easier to reason about state.

### 3.3 How they're connected
> [Walk through the diagram.] On the left, my home network — the MacBook, the homelab, the Mac Studio. On the right, external — the GCP VM. They're all stitched together by Tailscale in the middle. Notice the little agents riding on top of each box. That's the visual — the agent is the thing actually doing work *on* the box. The MacBook is where Claude Code and Codex live, and from there they reach any machine as if it were local.
>
> One detail worth calling out: I didn't install Tailscale on any of these boxes by hand. I had Claude Code SSH into each machine in turn — homelab, Mac Studio, GCP VM — run the install, authenticate the node, and join it to the tailnet. So even the VPN itself was bootstrapped through the same agent-on-the-box pattern you see in this diagram. The fastest way to wire up multi-machine infra turned out to be: let the agent do the wiring.

### 3.4 Why the VPN matters
> Without a flat addressable network, multi-machine agent work is painful. Tailscale is what makes "one agent, many machines" actually work. One SSH config, every node reachable, no public exposure of homelab services. This was load-bearing for the whole project.

## 4. Architectural Design (~4.5 min)

### 4.1 From idea to design doc
> Before any code, I wrote out the context in a CLAUDE.md file. Same pattern this very talk is built with, by the way — these slides were also bootstrapped from a CLAUDE.md.

### 4.2 What I gave Claude Code up front
> Four things. The goal — collect, store, search, summarize, notify. The inventory — the hardware I just showed. The platform picks — Airflow, Postgres, Elasticsearch, a small web frontend. And constraints — stay on existing infra, no paid SaaS. Notice: I picked the platforms. I didn't ask Claude what to use. That's where you waste hours debating. Hand it the bag of tools and say "design for these."

### 4.3 The system, end-to-end
> [Walk diagram.] Two flavors of Airflow DAG run on the homelab. The crawler DAGs pull from each community and land rows in Postgres. From Postgres they're indexed into Elasticsearch. The frontend lives on the GCP VM — it reads Postgres directly for the listings, and hits Elasticsearch for the search feature. A separate digest DAG pulls yesterday's window from Postgres, calls the local LLM on the Mac Studio for NSFW classification *and* summarization, and pushes the result to Telegram — all from within the same DAG. The Mac Studio is just an inference endpoint here, not an orchestrator.

### 4.4 The split that unblocked me
> Here's the prompt that changed everything: "split this plan by machine, so I can run sessions in parallel." Suddenly I had a homelab session, a Mac Studio session, and a GCP session — each with its own scope, its own context, its own todo list. The token budget thanks you. Your sanity thanks you. And the sessions can actually run in parallel.

### 4.5 How the split actually happens
> Concretely: I keep one master CLAUDE.md in the repo root. It captures the whole project — goal, infra inventory, data schema, platforms, conventions, and a short "where work runs" section.
>
> Then I run one prompt — literally "split this by machine and produce one CLAUDE.md per session" — and I get three derived files. Homelab, Mac Studio, GCP.
>
> Each session only loads its own slice. The homelab session never sees the nginx config. The Mac Studio session never sees Argo CD. The GCP session never sees the data schema.
>
> Three concrete benefits. Faster context — no irrelevant tokens. Less content rot — changes to the master don't accidentally regenerate stale assumptions in the per-machine files until I explicitly ask. And clearer accountability — when something goes wrong, I know exactly which file the agent was reading from.

## 5. Operations (~3 min)

### 5.1 Running it day-to-day
> Design is a one-off. Ops is forever. This is where Claude Code stopped being a code generator and became an actual operator.

### 5.2 Database work
> [Point at the snippet.] This is the entire data-model section in CLAUDE.md. Notice what's *not* there — I didn't write a single line of SQL. I just listed the fields I needed: title, URL, community, a cross-community global_id, fetched timestamp, vote counts, summary, is_nsfw, and a couple of null-until-filled-later columns. Claude picked the column types, picked sensible indexes, and dropped a migration into `db/migrations/` on its own. The point of putting this in CLAUDE.md isn't to hand-craft the schema — it's to give every future session the same starting context, so the crawler code and the query code never drift apart. When I need a new field later, I add a line to the list and ask for another migration.

### 5.3 Deployments — the happy path
> [Walk diagram.] This is a GitOps loop, and the two-repo structure matters. The app code lives in one repo. The Kubernetes manifests live in a *separate* repo. When I push to the app repo, GitHub Actions builds the image and does two things — pushes the image to the registry, *and* commits the new image tag back into the manifest repo. Argo CD is only watching the manifest repo; it doesn't poll the registry. It sees the new commit, syncs the cluster, the pod rolls. The cluster's desired state always lives in Git, which means re-syncs, rollbacks, and audits are just git operations.

### 5.4 The other path — SSH
> Not everything fits the Git-driven loop. Sometimes I need to bump a stuck pod's resources, run a one-off DB hotfix, swap an LLM model on the Mac Studio, or tweak nginx on the GCP VM. Claude Code with SSH access is, honestly, the cleanest on-call experience I've had. Because of Tailscale, every machine is one hop away.

---

### *[SPLIT POINT — natural break for two 15-min halves]*

---

# PART TWO — Multi-agent workflow & outcomes (~13 min)

## 6. Multi-Agent (~4.5 min)

### 6.1 Two agents, different jobs
> This is the part of the talk I find most fun. I had two coding agents working at the same time, and the trick was giving them different jobs.

### 6.2 The division of labor
> Claude Code did infrastructure and ops. SSH into machines, edit manifests, run kubectl, patch DAGs in place, debug pods. Codex did application code. Airflow DAG implementations, the frontend app, pure-code repos, PRs into GitHub. Early on Codex had to wait — Claude needed to set the infra up first. But once the foundation was there, both ran in parallel.

### 6.3 What it looks like in motion
> [Walk the animated diagram. Six steps.]
>
> Step one — Claude Code SSHs from the MacBook into the homelab or the Mac Studio for ops work.
>
> Step two — in parallel, Codex pushes to the app repo.
>
> Step three — Actions builds the image and pushes it to the registry.
>
> Step four — the same Action commits a tag bump into the *manifest* repo. That's the GitOps trigger.
>
> Step five — Argo CD is only watching the manifest repo. It sees the new commit.
>
> Step six — the cluster reconciles, pod rolls.
>
> The MacBook is the orchestrator. I'm just routing tasks between the two agents.

### 6.4a Why the split worked (upsides slide)
> Five reasons the lane split paid off.
>
> *No collision* — different layers, never the same file.
>
> *Right tool for the layer* — Claude lives in the terminal, Codex lives in the repo workflow.
>
> *Easier review* — Codex's diffs hit GitHub where review is easy; Claude's changes hit the cluster where Argo can compare against desired state.
>
> *Recoverable* — Argo CD is the truth. If anything drifts, I just re-sync.
>
> And one practical one most people don't expect — *token budget spread*. Daily and weekly rate limits hit half as fast when two providers share the load. If everything goes through one agent, you cap out mid-task and you're stuck. Splitting across Claude Code and Codex roughly doubles your runway.

### 6.4b …and where it bites (downsides slide)
> Now the downsides, because the split isn't free. Two of them.
>
> *Overlap = conflict.* If you accidentally let both agents touch the same file or the same cluster object, you get conflicts: either a git merge conflict, or worse, drift between what each agent *thinks* the world looks like.
>
> *No shared memory.* Each agent's notes live in its own session. If Claude just patched a deployment, Codex has no idea unless it goes and looks.
>
> *The mitigation is the same in both cases:* don't trust either agent's mental model. Have it re-read the source of truth — the code repo, the live cluster, the actual state — at the start of any non-trivial change.
>
> Land the callout: **two specialized agents beat one generalist — *as long as you keep their lanes from crossing***. The "as long as" matters. Without that discipline, the split actively hurts.

## 7. Hiccups & Lessons (~3 min)

### 7.1 It did not work the first time
> Honest section. This is not a "prompted once and shipped" story.

### 7.2 What broke early on
> Four real failures, and they cluster nicely on a 2x2.
>
> *K8s under-provisioned* — pods came up but resources were too tight. OOMs and throttling.
>
> *Silent crawls* — logs missing, so failed collections looked like empty days.
>
> *Elasticsearch misses* — term extraction option wasn't set, so a chunk of my docs were invisible to queries.
>
> *Airflow DAG fragility* — no parallelism, so one community failing cascaded into the next day's run.
>
> None of these were surprising in hindsight. They were just invisible until I looked. Claude helped triage each one — but the human still has to *look*.

### 7.3 What worked surprisingly well
> Five wins. Top row was the original surprise; bottom row is where Claude exceeded my expectations on the ops side.
>
> *Generic crawlers* — I gave Claude only the URLs. It figured out list parsing for each community on its own, without per-site code.
>
> *Schema-driven prompts* — once the schema was documented in CLAUDE.md, the crawler code and the query code stopped drifting apart.
>
> *Recovers broken systems* — one of the homelab nodes rebooted on its own. Claude couldn't tell me *why* the host went down — that's out of its reach. But it noticed a Postgres data disk failed to remount, edited `/etc/fstab`, brought the mount back, restarted Postgres. End-to-end recovery without me touching the keyboard.
>
> *Handles gnarly K8s commands and manifests* — the long kubectl incantations, the fiddly manifest fields like hostPath mounts, GPU mounts, secrets. These are the parts I always have to look up. Claude just does them.
>
> *Reads error logs and patches the code* — on a failure, it tails the logs, isolates the offending stack frame, and edits the code directly. No need for me to translate the failure into a prompt.

### 7.4 Refinements that mattered
> Five small refinements that compounded.
>
> *Timestamp normalization* — store UTC, render local.
>
> *HTML selector tuning* — extract body, not chrome or ads.
>
> *DAG isolation* — one community fails, the rest keep running.
>
> *Resource right-sizing* — observe actual usage, then bump.
>
> *UI polish* — the backend got steady, so I layered on quality-of-life features: a search bar wired to Elasticsearch, dark mode, filters by community and tag.
>
> Each one was a 30-minute conversation, not a 3-day project. That's the speed multiplier you get from this workflow.

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
> [Point at the screenshot.] This is an actual morning digest. The header is the date. Then the day's headlines as bullets — usually five to eight across all communities. Below that, per-community highlights. The whole thing fits on one screen, no scrolling needed. If a topic is interesting, I tap through to the original post. The point isn't that the digest is comprehensive — it's that it's enough to decide whether to dig in.

### 8.4 What changed for me
> Four things.
>
> *Time* — no more 20-minute lunch-hour scroll.
>
> *Coverage* — I see more communities than I used to visit manually.
>
> *Signal* — pre-ranked by what humans found engaging, not by an algorithmic feed.
>
> And one that changed how I read the web. *Discovery via digest.* I don't dig through whole threads anymore. The Telegram digest surfaces the popular posts, and I only follow the ones that look worth my time. The model flipped — from hunting, to triaging a curated list.

## 9. Closing (~2 min)

### 9.1 What this unlocks next
> The collection pipeline is the foundation. Everything else is a remix.

### 9.2 Takeaways
> Five things to remember.
>
> *Plan first, code second.* CLAUDE.md is your contract with the agent.
>
> *Split tasks into smaller chunks.* One session per box keeps state clean.
>
> *Keep agents' lanes from crossing.* If you run multiple agents in parallel, separate their areas cleanly — overlap creates conflicts and silent drift.
>
> *Engineering knowledge still matters.* Our background is the edge non-engineers don't have — agents amplify it, they don't replace it.
>
> *Data is the moat.* Once collection runs, every new feature is days, not months.

### 9.3 Closing — interactive CTA
> [Question slide. Read it out, then *pause*.]
>
> "What would *you* build with this pattern?"
>
> Let the question land. Don't rush past it. If someone in the room has an idea worth surfacing, ask them to share it.
>
> When you're ready to wrap, **click the button on the slide** (not the arrow key — the arrow advances to "Thank you"). The question floats up, two cards slide in from below.
>
> *Finance digest* — swap the source list for financial communities and news outlets, reuse the entire summarization stack.
>
> *Translated video shorts* — pull foreign-language stories, translate, summarize, generate narration, stitch into short videos.
>
> Both are mid-build, not theoretical. Anchor the closing: the pattern is reusable. The data layer is the moat. Then click forward to the Thank you slide.

---

## Timing buffer & cuts

- If running long: cut 2.4 — the realization re-lands at the start of §8.4.
- If *really* long: collapse §6.4a + §6.4b into a single mention ("upsides clear, downsides are overlap and no shared memory, mitigation is re-read the source of truth").
- If audience is junior-heavy: spend longer on the animated diagram in §6.3 — that's the conceptual core.
- If audience is senior-heavy: spend longer on §4.5 (CLAUDE.md fan-out) and §6.4b (downsides + mitigation) — those land hardest with experienced engineers.

## Demo backup (if there's time)

- Live Telegram message from this morning.
- Live frontend (the search UI over Elasticsearch on the GCP VM).
- A `kubectl get pods` from a Claude Code session — show ops in action.
- `cat examples/master-CLAUDE.md` followed by one of the derived files — make the fan-out concrete.

## Numbers to mention if anyone asks

- ~4 machines, 1 tailnet, ~6 weeks from idea to first Telegram message.
- ~30 minutes per refinement on average — the iteration loop is the win.
- Free GCP tier is enough for the frontend; everything else is hardware I already owned.

## Closing-CTA reminder

⚠️ On the final "What would you build" slide, **use the mouse / trackpad to click the button**, not the arrow key. The arrow advances to "Thank you" without revealing the two adjacency cards.

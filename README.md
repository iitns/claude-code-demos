# Zero to One with Claude Code — Slides

Single-file HTML presentation built with [Reveal.js](https://revealjs.com/) for a 15–30 minute talk
on building a personal content aggregator using Claude Code + Codex.

## How to view

Just open `index.html` in Chrome. No server needed — everything is inlined.

```bash
open index.html   # macOS
```

> Internet connection is required on first load to fetch Reveal.js and Mermaid from CDN.
> After that, the browser will cache them.

## Files

- **`index.html`** — the entire presentation (slides, styles, both SVG diagrams, all speaker notes inline)
- **`script/speaker-notes.md`** — standalone English speaker script for rehearsal (also embedded inline as Reveal speaker notes — press `s` in the deck)
- **`CLAUDE.md`** — project brief (input to Claude Code, not part of the deck)

## Reveal.js keyboard shortcuts

- `→` / `space` — next slide
- `←` — previous slide
- `s` — open speaker notes view (separate window)
- `f` — fullscreen
- `o` — overview mode
- `esc` — exit overview
- `?` — show all shortcuts

## Media placeholders

Look for dashed boxes labeled "Image placeholder" / "Screenshot placeholder". Three to fill in before the talk:

- §2 Motivation — phone scrolling / multiple tabs image
- §5 Operations — CLAUDE.md schema block + psql session
- §8 Outcome — Telegram digest message

Edit them directly in `index.html` — search for `media-placeholder`.

## Two-half split

Designed to break cleanly after **Section 5 (Operations)**:

- **Part 1 (~13 min)** — Overview, Motivation, Infra, Architecture, Operations
- **Part 2 (~12 min)** — Multi-agent, Lessons, Outcome, Applications

See `script/speaker-notes.md` for the marked split point.

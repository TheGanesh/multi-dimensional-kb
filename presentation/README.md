# Commerce Knowledge Base — Leadership Presentation

**File**: `Commerce-KB-Leadership.pptx` (13 slides, 16:9 widescreen, speaker notes on every slide)

A leadership briefing on the Commerce Knowledge Base framework: what problem it solves,
why it was built this exact way (markdown + RAG + MCP), the proof it works, and how it
scales to other domains.

## Slide map

| # | Slide | The message |
|---|-------|-------------|
| 1 | Title | One trustworthy source of truth for humans and AI agents — with the headline numbers |
| 2 | The problem | Knowledge rot was chronic; AI agents made it dangerous (confident wrongness at machine speed) |
| 3 | The idea | Treat knowledge exactly like software: source of truth, deterministic builds, tests, CI, ownership |
| 4 | How it works | The three-layer pipeline (seed → AI ingest → sources → deterministic generator → consumers) and its three guarantees |
| 5 | **Why markdown** | Humans + git + AI all read it natively; ~10× cheaper in tokens; Google OKF standard; zero lock-in |
| 6 | **Why RAG** | Deterministic answers first, measured search second; retrieval quality is a CI-gated number, not a demo |
| 7 | **Why MCP** | The Linux Foundation standard socket for AI agents — one server, every AI tool; 9 deliberate tools |
| 8 | Trust & freshness | Nightly change detection (≤24h vs 7-day SLO, zero cross-team footprint); the TDD-always-wins precedence rule |
| 9 | Proof | The five-exam evaluation stack; "a perfect score is a red flag"; golden-dataset story regeneration |
| 10 | Due diligence | Every pillar maps to a named industry standard (OKF, LLM-Wiki, MCP, Backstage, llms.txt, RAG eval, data mesh) |
| 11 | Scaling | Federation: shared framework package, one repo per domain, one-page registry — standards global, enforcement local |
| 12 | Cost & discipline | $0-LLM CI, zero standing infrastructure, numeric triggers for every deferred component |
| 13 | Roadmap & ask | Extract the framework package → onboard the 2nd domain → close the loop with real traffic |

## Presenting tips

- Slides 5–7 are the heart of the "why this architecture" argument — budget the most time there.
- Every slide has speaker notes with the anticipated leadership questions and answers
  (e.g., "why not Confluence + search?", "why not buy a vector DB?", "why not our own API?").
- The companion deep-dive is the 11-chapter book in [`../documentation/`](../documentation/README.md).

**Owner**: Commerce Architecture Team

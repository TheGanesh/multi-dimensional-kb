# Simplification 02 — Tier 2: Slim by Default

> **Audience**: Maintainers choosing defaults. These four items ship in their **slim**
> form by default; the heavier form is restored **only when our own eval or telemetry
> demands it**. That is the tier's rule: the framework's measuring instruments (E1/E2,
> usage logs, the story judge) decide — not intuition. Each item names its restore
> condition explicitly.

---

## S2.1 — BM25-Only by Default; the Dense/ONNX Tier Becomes Opt-In

**Current state.** Hybrid (BM25 α=0.4 + MiniLM ONNX dense β=0.6 → weighted RRF) is the
default mode. The ~90 MB ONNX model is committed in-repo; `[embeddings]` is part of the
recommended install; embeddings warm-cache in ~20 s
([Chapter 03](../03-mcp-and-retrieval.md#hybrid-search-pipeline)).

**What it costs.** The model weight rides in **every clone of every domain repo**
(federation multiplies it: commerce, customer, order… each carrying 90 MB of identical
weights in git history forever), plus `onnxruntime`/`tokenizers` in every environment,
the embedding cache, the warm-up, and a second scoring path through every retrieval
change. All of it serves exactly **one** of the four query routes — `exploratory` — since
analytical/navigational/factual queries are answered by deterministic graph lookups
([Chapter 03](../03-mcp-and-retrieval.md#smart-query-routing)).

**The change.** Default profile = BM25-only. The dense tier stays fully implemented in
`kb-framework` behind the `[embeddings]` extra — one pip install away, with the model
delivered by the framework package/artifact rather than committed per-domain.

**The proof.**

1. *This corpus is BM25's home turf.* The dominant vocabulary is exact-term:
   `shoppingcartms`, `addItemsToCart`, `/shopping-cart/v1/carts/{cartId}/items`, Kafka
   topic names, error codes. BM25 is the strongest retriever for exact terms — the
   framework's own docs say so ("best for exact terms: service names, operationIds").
   Dense embeddings earn their keep on *paraphrase* queries, which are a minority slice
   of a KB whose consumers are scripted workflows asking structured questions.
2. *The curriculum we absorbed teaches this exact ordering.* The zoomcamp
   [next-steps lesson](https://github.com/DataTalksClub/llm-zoomcamp/blob/main/02-vector-search/lessons/10-next-steps.md) is
   explicit: v1 = text search, v2 = add vectors **only when evaluation proves text
   search misses**, v3 = hybrid — and it warns that vector-first advice
   disproportionately comes from vector-DB vendors. Their measured gains agree: hybrid +
   RRF over keyword search moved Hit Rate 0.917 → 0.925
   ([reranking lesson](https://github.com/DataTalksClub/llm-zoomcamp/blob/main/06-best-practices/lessons/03-reranking.md)) — real,
   but single-digit.
3. *We currently cannot prove the dense tier's value.* The 0.98 Recall@5 headline was
   measured on the saturated 54-query suite
   ([known correction #2](../06-freshness-and-evaluation.md#corrections-to-the-original-design)).
   Until the honest E1 v2 set exists, the dense tier is carried on faith. The correct
   order of operations is: build the honest eval **first**, then let it decide.

**Restore condition (explicit).** Build the E1 v2 ground-truth set; run E2 on the
*exploratory slice* in both modes. **If hybrid beats BM25-only by ΔRecall@5 ≥ 0.03 (or
ΔMRR ≥ 0.05), the dense tier returns as default** — now with a number attached that
justifies its 90 MB and its dependencies forever after. If the delta is smaller, the
slim default stands, and the number defends *that* instead. Either way the decision
stops being arguable.

**Trade-off & mitigation.** Paraphrase-style queries ("how does checkout repricing
work") may degrade in the interim. Mitigation: the zero-result / low-score panel in the
usage report (S2.4) surfaces exactly these queries — they become the evidence for the
restore condition rather than silent losses.

**Verification.** Publish the two-mode E2 table in the release scorecard; record the
decision + delta in an ADR so the question is never re-litigated without new data.

---

## S2.2 — Two Access Channels, Not Three

**Current state.** Consumer workflows carry three retrieval procedures
([Chapter 04](../04-consumer-workflows.md#tri-channel-kb-access)): Phase 1 recursive
markdown crawling from `Commerce-Services-Summary.md`, Phase 1.5 `kb context`/`kb search`
RAG, Phase 2 MCP tools — plus precedence rules and a degradation ladder across all three.

**What it costs.** Every consumer workflow prompt must encode three procedures, their
precedence, and their failure handling — that's prompt surface repeated in every
workflow, forever. Worse, the crawl channel is *expensive at inference time*: recursive
`read_file` pulls whole pages (navigation, tables, sections the task doesn't need) into
context, exactly the token waste the
[llms.txt movement](https://www.mintlify.com/blog/what-is-llms-txt) exists to eliminate.
And a consumer that silently degrades to crawling produces **unmeasured** answer quality
— no retrieval metrics apply to an agent wandering through links.

**The change.** Two channels: **`kb` CLI** (offline floor — no server needed) and
**MCP** (served, richest). The markdown tree remains fully navigable for *humans*
(`index.md`, `Commerce-Services-Summary.md` stay); it simply stops being a scripted
retrieval procedure in workflow prompts.

**The proof.**

1. *The crawl channel is dominated on every axis it claims.* Its stated value is
   "always available (offline)" — but `kb context` is equally offline (CLI-native, no
   server, [Chapter 02](../02-getting-started.md#recipes)), and strictly better for
   agents: token-budgeted, citation-tagged, ranked. The only scenario where crawling
   wins is "`kb` CLI not installed" — which is one `pip install` away and is a setup
   error to fail loudly on, not an operating mode to silently degrade into.
2. *Fewer channels = measurable channels.* E2/E3 metrics cover CLI and MCP retrieval.
   Nothing covers free-form crawling. Removing the unmeasurable channel means every
   retrieval path a consumer can take is under a gate — which is the framework's core
   promise.
3. *We can prove this cut safely — with machinery we already built.* This is the
   textbook use of the golden-dataset story judge
   ([`/kb-evaluation-judge`](../04-consumer-workflows.md#golden-dataset-evaluation--kb-evaluation-judge)):
   regenerate the golden stories with the crawl channel removed from the workflow
   prompts and compare scorecards. If scores hold, the channel was weight; if they
   drop, we learned exactly what the crawl uniquely contributed and can encode *that*
   into `kb context` instead.

**Restore condition (explicit).** Golden-dataset story scores degrade with the crawl
channel removed, and the gap is attributable to content `kb context` cannot reach.

**Trade-off & mitigation.** Environments with neither CLI nor MCP now hard-fail.
Mitigation: the workflow's failure message contains the one-line install command —
loud failure with remedy beats silent quality loss.

**Verification.** Run `/kb-evaluation-judge` on the golden features with two-channel
prompts; diff aggregate scores vs baseline; shrink the workflow prompt sections and
count the removed tokens.

---

## S2.3 — Drop the T1 Event Trigger; Nightly Fingerprint Scan Instead

**Original design (since corrected).** The freshness model as first drafted included a
T1 event trigger: a ~5-line CI step added to **each of 45 service repos** that notifies
the KB's dirty-list on merge to main.
[Chapter 06](../06-freshness-and-evaluation.md#the-freshness-model--detect-refresh-backstop)
now reflects this cut — its detect stage is the nightly scan described below.

**What it costs.** The cost is not the 5 lines — it's *organizational*: 45 PRs into
repos owned by other teams, 45 approvals, security review of a cross-repo trigger
mechanism, and permanent maintenance (every service pipeline migration must preserve the
hook). Federation multiplies it: onboarding the customer domain means doing it again
across *their* service repos. T1 quietly turns "adopt the KB" into "change every team's
CI," which is precisely the kind of friction that stalls platform adoption.

**The change.** Delete T1. The nightly T2 job **computes the dirty set itself**: shallow
`git pull` of the declared service repos, hash the ingest-relevant files, diff against
`.fingerprints.json` (which [already exists](../01-architecture.md#freshness-tracking)
for exactly this purpose), re-ingest what changed. T3 (TTL backstop) is unchanged.

**The proof.**

1. *The SLO has 7× headroom without T1.* The freshness SLO is "changed service
   re-ingested ≤ 7 days." A nightly scan delivers ≤ 24 h worst-case — comfortably
   inside the target. T1's only contribution is shrinking *detection* latency from
   hours to minutes, but detection isn't the bottleneck: **ingestion** (the LLM re-scan)
   runs nightly either way. Paying 45 repos' worth of organizational friction to
   accelerate the part that wasn't slow is a bad trade.
2. *Polling beats events at this change rate.* Event-driven wins when changes are
   frequent and freshness budgets are tight (minutes). Here: a service changes at most
   a few times per week in ways that affect the KB, and the budget is days. The
   standard engineering guidance — poll when change rate is low and staleness budget is
   generous — lands squarely on polling.
3. *The scan is cheap and self-contained.* 45 shallow pulls + file hashing is minutes
   of CI time, $0 LLM, zero footprint outside the KB repo — and it's the same code path
   `.fingerprints.json` maintenance already requires. One team owns the whole freshness
   mechanism end to end.

**Restore condition (explicit).** The freshness SLO tightens to same-day, **or** the
dashboard shows the nightly scan repeatedly missing the ≤ 7-day SLO (e.g., scan job
instability), **or** repo count grows to where nightly scanning exceeds its CI window.

**Trade-off & mitigation.** No same-day re-ingest; a silently failing scan job could
mask changes. Mitigation: the scan job alerts on failure (it's *our* CI, we own the
alert), and T3/D2 remains the independent backstop.

**Verification.** Measure scan wall-clock over the 45 repos; plot achieved
change→ingest latency on the dashboard for a month — it should sit well under 7 days,
which *is* the proof the SLO never needed T1.

---

## S2.4 — Usage Telemetry as JSONL + `kb usage-report` Before Postgres + Grafana

**Original design (since corrected).** The Phase 1 monitoring stack was first specified
as `postgres:17` + `grafana/grafana` containers, the two-table schema, MCP
instrumentation, and eight dashboard panels.
[Chapter 06](../06-freshness-and-evaluation.md#phase-1-usage-telemetry--jsonl-first-zero-docker)
now reflects this cut — JSONL first, containers at graduation.

**What it costs.** Two always-on containers with volumes, credentials, backup and
uptime ownership, Grafana provisioning — likely a corporate ticket or two — standing up
**before a single query has ever been logged** and before anyone has demonstrated they
will watch a dashboard. Infrastructure without an audience is pure carrying cost.

**The change.** The MCP server and `kb` CLI append one JSON line per query/tool call to
a local `usage/*.jsonl` file — **same fields as the `conversations` schema** — and a new
`kb usage-report` command aggregates weekly: queries by tool, p95 latency, **zero-result
queries** (the content-gap backlog), never-retrieved pages. The report is committed like
`audit-report.md`. Postgres + Grafana become the *graduation*, not the starting point.

**The proof.**

1. *The irreducible Phase 1 needs are three, and a file serves all of them.* (a) Log
   every query. (b) Surface zero-result queries — the single highest-value signal
   ([Chapter 06](../06-freshness-and-evaluation.md#instrumentation--outputs) calls it
   the goldmine). (c) Count what's used. None of these requires SQL, concurrent
   writers at scale, or live charts on day one. Append-only JSONL + a report command
   delivers all three at zero infrastructure — and matches the framework's existing
   ethos (`log.md` append-only operational history,
   [LLM-Wiki pattern](../01-architecture.md#append-only-operation-log)).
2. *The heavy stack's real customer is Phase 2.* The Postgres tables earn their keep
   when the **online judge** arrives (async workers writing verdicts, human-vs-judge
   agreement queries, sampling) — which is a Phase 2 component
   ([Chapter 07](../07-phase-2-target-architecture.md#4-online-evaluation-loop)). The
   zoomcamp monitoring module that inspired the stack was built for a *live public
   application with real user traffic*; an internal Phase 1 tool has neither yet.
3. *Graduation is a loader, not a redesign.* Because the JSONL fields mirror the
   `conversations` schema exactly, promoting to Postgres is
   `COPY`/insert of historical files — nothing about the instrumentation call sites
   changes. The slim start costs no rework later.

**Restore condition (explicit).** Any of: the online judge (E3-online) starts; more
than one person asks for usage numbers more often than the weekly report; multiple
long-running MCP server instances make file-append coordination annoying.

**Trade-off & mitigation.** No live dashboards; no cross-domain SQL until graduation.
Mitigation: per-process JSONL files merged at report time (no lock contention); the
weekly report is committed, so trends are diffable in git — crude, but reviewable.

**Verification.** After two weeks of logging: the usage report must produce a non-empty
zero-result list (that list's usefulness is the whole argument); then time the
JSONL→Postgres loader once against a copy to prove graduation is trivial.

---

## Tier 2 Scorecard

| Item | Slim Default | Heavy Form Returns When |
|------|--------------|--------------------------|
| S2.1 | BM25-only retrieval | E2 on honest eval set shows ΔRecall@5 ≥ 0.03 for hybrid |
| S2.2 | CLI + MCP channels | Golden story scores drop without the crawl channel |
| S2.3 | Nightly fingerprint scan | SLO tightens to same-day, or scan misses SLO |
| S2.4 | JSONL + `kb usage-report` | Online judge starts, or dashboard demand is real |

The common thread: **every restore condition is a measurement the framework already
knows how to take.** Nothing here is decided by opinion twice.

---

**Next**: [03 — Tier 3: Defer Until Trigger](./03-tier-3-defer-until-trigger.md) | **Back**: [01 — Tier 1: Cut Outright](./01-tier-1-cut-outright.md) | **Up**: [Simplification README](./README.md)

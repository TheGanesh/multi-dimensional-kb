# Simplification 03 — Tier 3: Defer Until Trigger

> **Audience**: Maintainers resisting the urge to build ahead of need. Everything here
> is **already designed** (Chapters 05–07) — the design work is done and paid for.
> The discipline this chapter adds: none of it gets *built* until its named trigger
> fires. Designed-but-unbuilt is the cheapest state a component can be in; built-but-
> unneeded is the most expensive.

---

## S3.1 — Federation Gateway MCP + FED-03…05

**What is deferred.** The cross-domain gateway MCP (`kb_route`,
`kb_federated_search`, `kb_read`) and the heavier federation checks — cross-domain link
resolution (FED-03), version-floor enforcement (FED-04), SLO monitoring (FED-05)
([Chapter 05](../05-scaling-and-federation.md#mcp-topology--scoped-servers--one-gateway)).

**What remains meanwhile.** `registry.yaml` with the commerce entry, and two checks:
FED-01 (registry schema) and FED-02 (bundle conformance). The registry file can live in
the `kb-framework` repo until a second domain exists — a dedicated `kb-registry` repo
for a one-row table is ceremony.

**Why deferring is safe — the proof.**

1. *A gateway with one domain behind it is an alias.* Every query it could route has
   exactly one destination — the commerce MCP server. Until a second domain publishes a
   bundle, the gateway adds a network hop, an auth surface, and an operational
   component while changing zero answers.
2. *Cross-domain demand is currently hypothetical.* No usage data shows consumers
   asking cross-domain questions — because usage telemetry (S2.4) is only now starting.
   Building the gateway first inverts the evidence order the framework preaches.
3. *Nothing rots while waiting.* The `kb://<domain>/…` URI scheme (the actual
   *prerequisite* for federation) ships now and costs nothing to carry; dangling
   cross-domain references are already tracked by AUD-25. When the gateway arrives,
   the data model is ready — deferral loses no ground.

**The trigger (build when ALL are true).**
- ≥ 2 domains are `active` in `registry.yaml`, **and**
- usage logs (S2.4) show real cross-domain questions, **or** a consumer workflow
  concretely blocks on cross-domain context (e.g., a commerce story needing customer
  profile contracts).

---

## S3.2 — E4 Extraction-Fidelity Automation

**What is deferred.** The nightly automated sampling harness that diffs `sources/` YAML
claims against code-derived ground truth
([Chapter 06](../06-freshness-and-evaluation.md#e4--extraction-fidelity-eval-new)).

**What replaces it meanwhile — and this is the better move anyway.** Attack the
*surface* instead of automating the *audit*: extend `/ingest-service` to parse OpenAPI
specs **deterministically** wherever they exist (paths, methods, operationIds, request/
response schemas), leaving the LLM only what specs cannot express (outbound call chains,
Kafka usage, feature flags). Every fact moved from "LLM-extracted" to
"deterministically parsed" no longer needs auditing at all — it inherits the generator's
determinism guarantee. Plus a **quarterly manual spot-check**: one engineer, five random
services, one hour, diff the YAML against the code by eye; log findings in `log.md`.

**Why deferring is safe — the proof.**

1. *Shrinking the attack surface beats instrumenting it.* If OpenAPI coverage reaches
   most inbound APIs, the majority of `inboundApis[]` facts become deterministic — the
   nightly harness would then be built to audit a minority slice. Do the surface
   reduction first; size the harness (if still needed) to what actually remains
   LLM-asserted.
2. *The trust model already contains the failure.* LLM-only facts carry
   `confidence: inferred` → consumers render them as `⚠️ KB-INFERRED` and never let
   them override TDD ([Chapter 04](../04-consumer-workflows.md#key-rules)). An
   extraction error's blast radius is *flagged-and-reviewable*, not silent — which
   buys time for the quarterly cadence to catch drift.
3. *A manual spot-check produces the sizing data the harness needs.* Building the
   automation first means guessing sample rates and thresholds. Two quarters of manual
   findings tell us the actual error rate — and whether a nightly harness is justified
   at all.

**The trigger (build the automation when ANY is true).**
- A quarterly spot-check finds a *material* extraction error (wrong endpoint, invented
  dependency) that the `inferred` tagging did not contain, **or**
- OpenAPI coverage stalls below ~50% of inbound APIs (LLM surface stays large), **or**
- a downstream incident is traced to a wrong KB fact.

---

## S3.3 — E5 Trajectory Evaluation

**What is deferred.** The per-release trajectory judging of MCP tool-call sequences
against the rubric (≤3 searches, no duplicate args, refinements only)
([Chapter 06](../06-freshness-and-evaluation.md#e5--trajectory-eval-mcp-tool-use)).

**What remains meanwhile.** Trajectory *logging* only — the `tool_name` field in the
S2.4 JSONL. Logging is nearly free and builds the dataset that judging will need.

**Why deferring is safe — the proof.**

1. *Today's consumers have nothing to judge.* The consumer workflows
   ([Chapter 04](../04-consumer-workflows.md)) are **scripted**: they call a fixed,
   documented sequence of tools (Phase 1 → 1.5 → 2). A trajectory judge evaluating a
   hardcoded sequence measures the script's author, not an agent's behavior — every
   run scores identically. The rubric was designed (via zoomcamp's
   [agent-evaluation lesson](https://github.com/DataTalksClub/llm-zoomcamp/blob/main/04-evaluation/lessons/14-agent-evaluation.md))
   for *agents making tool choices*.
2. *The rubric gets better by waiting.* After S1.2 (25 tools → 9), the rubric is
   simpler; after real agentic traffic exists, thresholds (how many searches is
   normal?) can be set from observed distributions instead of guessed.

**The trigger.** Consumers become genuinely agentic — an LLM *chooses* which KB tools
to call (e.g., the gateway MCP is live, or IDE agents free-query the KB) rather than
executing a scripted channel sequence.

---

## S3.4 — The Entire Phase 2 Stack (Hold the Line)

**What is deferred.** pgvector central index, Kestra orchestration, OpenSearch, LLM
query rewriting, online judge infrastructure — everything in
[Chapter 07](../07-phase-2-target-architecture.md).

**Why this needs saying at all.** Chapter 07 already gates each component behind
measured triggers — but the risk it cannot gate is *enthusiasm*: central infrastructure
is more interesting to build than YAML hygiene, and platform teams habitually build
Phase 2 because Phase 1 got boring, not because a trigger fired. This item exists to
make "we held the line" an explicit, reportable decision.

**The proof that holding is correct.**

1. *Every Phase 2 component solves a scale problem we measurably do not have.*
   ~1,300 chunks load in < 2 s; one domain is active; there is no cross-domain traffic;
   there is no online traffic to judge. The
   [migration-trigger table](../07-phase-2-target-architecture.md#migration-triggers--when-to-actually-do-this)
   is the contract — none of its rows are true today.
2. *The one exception is already scheduled correctly.* The online judge is flagged
   "adopt FIRST — nearly free once real traffic exists." Its precondition is *traffic*,
   which S2.4 telemetry will demonstrate — so even the exception waits on a
   measurement, not a mood.
3. *Deferral costs nothing because the contracts are frozen.* Bundles, `kb://` URIs,
   and MCP tool shapes are identical in both phases — that was the point of designing
   Phase 2 fully. Nothing built today must be rebuilt then.

**The trigger.** Exactly the Chapter 07 table — reviewed quarterly against dashboard
numbers, with a one-line verdict recorded in `log.md` ("Phase 2 review: no trigger
fired"). That log line is cheap and makes the discipline auditable.

---

## S3.5 — Workflow Consolidation: 10 → 7

**What is deferred-and-then-done.** Three workflow merges, executed opportunistically
with the next framework release rather than as a standalone project:

| Merge | Rationale |
|-------|-----------|
| `/explain-impact` → folds into `/generate-kb` | Impact pages are derived artifacts of the same generation pass; a separate workflow implies a separate lifecycle that doesn't exist |
| `/audit-kb`'s LLM semantic analysis → on-demand flag (`/audit-kb --semantic`) | The deterministic audit (`kb check` after S1.5) runs always and gates CI; the LLM analysis is exploratory, costs tokens, and belongs behind an explicit ask |
| `/ingest-auxiliary` absorbs any future one-off ingest variants | Prevents the ingest tier from regrowing one workflow per file type |

**Why this is safe — the proof.** None of the three changes what gets produced — they
change how many named entry points a new team member must learn. Ten workflows was the
inventory when each was young; the tier structure (ingest → generate → lifecycle) is
the real mental model, and 7 entries fit it exactly (4 ingest, 1 generate, 2
lifecycle/audit).

**The trigger.** The next `kb-framework` minor release touching workflow definitions —
ride along, don't schedule separately.

---

## Tier 3 Scorecard

| Item | State Today | Build When |
|------|-------------|-----------|
| S3.1 Gateway + FED-03…05 | Designed; registry stub only | 2+ domains AND logged cross-domain demand |
| S3.2 E4 automation | OpenAPI-first extraction + quarterly manual spot-check | Spot-check finds material error / coverage stalls |
| S3.3 E5 judging | Logging only | Consumers become agentic |
| S3.4 Phase 2 stack | Fully designed, quarterly trigger review in `log.md` | Chapter 07 trigger table |
| S3.5 Workflows 10→7 | Pending | Next framework release touching workflows |

---

**Next**: [04 — Do Not Cut](./04-do-not-cut.md) | **Back**: [02 — Tier 2: Slim by Default](./02-tier-2-slim-by-default.md) | **Up**: [Simplification README](./README.md)

# Simplification 04 — Do Not Cut

> **Audience**: Anyone tempted to extend the simplification past its safe boundary —
> including future maintainers reading tiers 1–3 and asking "why stop there?" These
> eight elements are load-bearing: each one either *is* the purpose, *is* the proof, or
> is the foundation that makes the tier 1–2 cuts safe. For each: what it carries, what
> concretely breaks without it, and which simplifications lean on it.

---

## K1 — `seed.yaml` as the Single Declared Input

**What it carries.** The entire "nothing is auto-discovered" property: every source
feeding the KB is declared, reviewable, and diffable in one file
([Chapter 01](../01-architecture.md#seedyaml--the-single-input)).

**What breaks without it.** Auto-discovery (scanning whatever repos/wikis are
reachable) makes KB contents a function of *environment* instead of *declaration* —
provenance (C11) becomes unverifiable, the fingerprint scan (S2.3) loses its definition
of "the declared repos," and a wiki page nobody vetted becomes agent-served truth.

**Leaned on by.** S2.3 (the nightly scan iterates `seed.yaml`), the ownership story
(the file is what CODEOWNERS protects), onboarding (`kb init` scaffolds it).

---

## K2 — Deterministic Generation + the Sync Gate (on Pages)

**What it carries.** The guarantee that `markdown/` is a pure function of `sources/`:
regeneration is byte-identical, so the committed bundle provably matches its inputs and
"stale generated pages" cannot reach `main`
([Chapter 01](../01-architecture.md#three-layer-architecture)).

**What breaks without it.** Hand-edited or drifted pages — the classic wiki failure
mode the whole framework exists to escape. The trust tiers collapse (nothing is
`authoritative` if generation isn't reproducible), and S1.3 becomes *unsafe*: deriving
chunks at build time is only sound because the pages they derive from are themselves
deterministically pinned.

**Leaned on by.** S1.3 (chunks-as-artifacts), FED-02, the leadership claim "CI is
$0-LLM and deterministic."

---

## K3 — Stable `kb://` URIs

**What it carries.** Identity. Every evaluation metric, every cross-domain link, every
provenance chain, and every Phase 2 upsert keys on the URI
([Chapter 01, Pattern 2](../01-architecture.md#pattern-2-stable-resource-identity)).

**What breaks without it.** The zoomcamp lesson is blunt: *"if you can't uniquely
identify a document, you can't tell whether search retrieved the right one"*
([ground-truth lesson](https://github.com/DataTalksClub/llm-zoomcamp/blob/main/04-evaluation/lessons/02-ground-truth.md)) — E1/E2
become unmeasurable, RRF merging collides (the zoomcamp generic `rrf()` keyed on text
is the cautionary bug), federation links dangle unresolvably, and the Phase 2 loader
cannot upsert idempotently.

**Leaned on by.** Literally every tier — S2.1's restore condition is *measured against
URIs*; S3.1's federation *is* URI resolution.

---

## K4 — The Frontmatter Contract

**What it carries.** Machine-readable type, provenance, confidence, staleness, and the
`disclosure` one-liner on every page — the entire OKF conformance story and the
mechanism agents use to decide relevance without reading bodies
([Chapter 01](../01-architecture.md#frontmatter-contract)).

**What breaks without it.** Progressive disclosure dies (agents must read whole pages —
the token waste S2.2 fights returns at 10× scale); `confidence: inferred` tagging dies
(S3.2's containment argument evaporates); `stale_after` dies (T3 backstop blind); OKF
conformance — the headline standards-alignment claim — is unfalsifiable.

**Leaned on by.** S2.2, S3.2, T3 freshness, FED-02, the gateway's disclosure-first
design, the leadership defense.

---

## K5 — The Audit Score Gate (≥ 60, 0 Errors)

**What it carries.** The single number that makes "KB health" enforceable in CI and
comparable across domains and time
([Chapter 03](../03-mcp-and-retrieval.md#ci-pipeline)).

**What breaks without it.** Quality regressions merge silently; the federation
knowledge-as-a-product claim ("every bundle ships with quality guarantees",
[Chapter 05](../05-scaling-and-federation.md#governance--data-mesh-applied-to-knowledge))
loses its enforcement; FED-05 has nothing to read. Note S1.5 *consolidates the runner*,
not the gate — the score and its threshold survive unchanged.

**Leaned on by.** S1.5 (the merged runner exists to compute this gate faster), T2
auto-PRs (unattended merges are only safe because this gate fronts them).

---

## K6 — E1/E2: Versioned Ground Truth + Retrieval Gate in CI

**What it carries.** The measuring instrument. Versioned golden queries with held-out
test split, Recall@K/MRR gates on every PR
([Chapter 06](../06-freshness-and-evaluation.md#e1--synthetic-ground-truth-generation)).

**What breaks without it.** **Every tier 2 decision becomes a guess.** S2.1's
BM25-vs-hybrid verdict, S2.2's golden-story comparison, chunk-size tuning, boost
tuning — all of them are E1/E2 measurements. Cut the instrument and the simplification
program's core claim ("we measured our way out of every component we don't run")
inverts into "we guessed our way out." This is the one component whose *absence* makes
simplification reckless rather than disciplined.

**Leaned on by.** S2.1, S2.2, S1.1's verification, every restore condition in tier 2,
Chapter 07's migration triggers.

---

## K7 — MCP as the Serving Protocol

**What it carries.** Vendor-neutral agent access — the Linux Foundation standard every
major model vendor speaks
([Chapter 05](../05-scaling-and-federation.md#defending-the-approach--theory--citations-for-leadership)).

**What breaks without it.** A custom API means custom integration in every IDE/agent —
the exact per-consumer glue MCP was adopted to delete — and the strongest third of the
leadership defense disappears. Note S1.2 cuts *tool count*, never the protocol: 9 tools
over MCP is a simplification; a bespoke REST API with 9 endpoints would be a regression.

**Leaned on by.** S1.2 (which is a *within-MCP* consolidation), S3.1 (the gateway is
just another MCP server), all consumer workflows.

---

## K8 — CODEOWNERS-Enforced Domain Ownership

**What it carries.** The federation's entire governance model: domain teams own their
sources through their normal PR workflow — Backstage's model, data mesh's first
principle ([Chapter 05](../05-scaling-and-federation.md#the-federation-model)).

**What breaks without it.** Central-team bottleneck (the platform team reviews every
domain's knowledge changes — unscalable) or ungoverned edits (anyone changes anything —
untrustable). Both destroy the "owned by domain teams, maintained in a common shape"
requirement that motivated federation in the first place. It also unravels
accountability plumbing: FED-05 SLO violations route to *owners*; T3 notifications
route to *owners*; without ownership records, both fall back to the shared-inbox
anti-pattern the design explicitly corrected
([correction #6](../06-freshness-and-evaluation.md#corrections-to-the-original-design)).

**Leaned on by.** S2.3 (scan findings routed to owners), FED-05, the entire Chapter 05
governance section.

---

## The Boundary, Stated Once

Tiers 1–3 remove **redundant implementations** (LSA, three runners), **derived copies**
(committed chunks), **premature infrastructure** (gateway, Postgres, Phase 2), and
**organizational friction** (45-repo CI hooks). K1–K8 are none of those — they are the
framework's *identity* (declared inputs, determinism, stable IDs, typed metadata), its
*proof* (audit gate, E1/E2), and its *interface* (MCP, ownership).

A useful smell test for any future cut proposal: if removing X makes some measurement
impossible, some provenance unverifiable, or some owner unreachable — X is a K, not an
S, and the answer is no.

---

**Back**: [03 — Tier 3: Defer Until Trigger](./03-tier-3-defer-until-trigger.md) | **Up**: [Simplification README](./README.md) | **Framework docs**: [Main README](../README.md)

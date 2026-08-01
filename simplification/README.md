# Simplification — Minimum Viable Framework

> **Audience**: Maintainers deciding what to build (and *not* build) next, and reviewers
> who suspect the framework has accumulated more surface area than its purpose requires.
> They are right — this folder is the audited cut list.

---

## Why This Folder Exists

The framework's purpose is one sentence:

> **Accurate, fresh, provable service knowledge that feeds solution intent, technical
> design, and story generation — owned by domain teams, consumable by agents.**

Everything below is measured against that sentence. Chapters 01–07 describe the *full*
design; this folder identifies which parts of it are load-bearing and which parts are
weight. Every recommendation follows the same discipline:

1. **Evidence first** — the claim is proven with numbers from our own docs, our own eval
   machinery, or cited industry practice; never by taste.
2. **Reversible** — each cut keeps a documented, cheap path back.
3. **Measurable** — each cut ships with a verification step that proves nothing broke.

The strongest leadership argument this produces:
*"we measured our way out of every component we don't run"* — which defends the framework
better than any feature list.

---

> **Status (2026-08-01)**: Chapters **05–07** have been corrected to reflect these
> recommendations (deferred gateway/FED checks, detect/refresh/backstop freshness,
> JSONL-first telemetry, BM25-first retrieval, 9-tool MCP surface). Chapters **01–04**
> describe the running implementation and will be updated once the changes are
> implemented.

## Complexity Inventory (as originally specified)

What the full design carried before the cuts — the counts below are the *before*
picture this folder argues against:

| Dimension | Count | Where Defined |
|-----------|-------|---------------|
| MCP tools | 25 (incl. 3 deprecated aliases) | [Chapter 03](../03-mcp-and-retrieval.md#25-mcp-tools) |
| Retrieval modes | 4 (hybrid, bm25, dense, lsa) | [Chapter 03](../03-mcp-and-retrieval.md#search-modes) |
| Consumer access channels | 3 (markdown crawl, `kb` CLI, MCP) | [Chapter 04](../04-consumer-workflows.md#tri-channel-kb-access) |
| Version planes | 4 (OKF, framework, content, CLI) | [Chapter 01](../01-architecture.md#version-planes) |
| Quality runners | 3 scripts (validate, audit, conformance) | [Chapter 01](../01-architecture.md#operational-governance--log-lint-audit-evaluation) |
| Check IDs | 52+ (AUD-01…25, C1–C12, D1–D10, FED-01…05) | Chapters 01, 05 |
| Workflows | 10 | [Chapter 02](../02-getting-started.md#workflows) |
| Committed derived files | 1,291 chunks | [Chapter 03](../03-mcp-and-retrieval.md#rag-chunking) |
| Evaluation layers | 5 (E1–E5) | [Chapter 06](../06-freshness-and-evaluation.md#the-evaluation-stack--five-layers-of-proof) |
| Freshness triggers | 3 (event, schedule, TTL) | [Chapter 06](../06-freshness-and-evaluation.md#the-freshness-model--detect-refresh-backstop) — now detect/refresh/backstop |
| Docker containers (Phase 1) | 2 (postgres, grafana) | [Chapter 06](../06-freshness-and-evaluation.md#phase-1-usage-telemetry--jsonl-first-zero-docker) — now JSONL-first |

---

## The Tiers

| Chapter | Tier | Rule | Items |
|---------|------|------|-------|
| **[01 — Cut Outright](./01-tier-1-cut-outright.md)** | 1 | Remove now; no purpose lost, no evidence needed beyond what we already have | S1.1–S1.5 |
| **[02 — Slim by Default](./02-tier-2-slim-by-default.md)** | 2 | Ship the slim version as default; restore the heavy version only when our own eval/telemetry demands it | S2.1–S2.4 |
| **[03 — Defer Until Trigger](./03-tier-3-defer-until-trigger.md)** | 3 | Already designed — do **not** build until a named, measurable trigger fires | S3.1–S3.5 |
| **[04 — Do Not Cut](./04-do-not-cut.md)** | — | Load-bearing elements; cutting any of these collapses the purpose or the proof | K1–K8 |

---

## Net Result (Tiers 1 + 2 applied)

| Dimension | Before | After |
|-----------|--------|-------|
| MCP tools | 25 | **9** |
| Retrieval modes | 4 | **1–2** (bm25 default, hybrid opt-in) |
| Access channels | 3 | **2** |
| Version planes | 4 | **2** |
| Quality runners | 3 | **1** (IDs preserved) |
| Committed derived files | 1,291 | **0** |
| Docker containers | 2 | **0** (JSONL first, graduate later) |
| Cross-team CI changes required | 45 repos | **0** |
| Workflows | 10 | **7** |

The pipeline itself — `seed.yaml → ingest → sources/ → generate → markdown bundle →
search → gates → MCP` — is untouched. So is every part of the
[leadership defense](../05-scaling-and-federation.md#defending-the-approach--theory--citations-for-leadership).

---

**Start**: [01 — Cut Outright](./01-tier-1-cut-outright.md) | **Framework docs**: [Main README](../README.md)

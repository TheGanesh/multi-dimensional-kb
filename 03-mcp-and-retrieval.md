# 03 — MCP Server, Search & CI Pipeline

> **Audience**: Developers integrating with the MCP server, or maintaining the CI pipeline.

---

## MCP Server Overview

`graph-mcp/server.py` serves the KB through **9 unified MCP tools** ([S1.2](./simplification/01-tier-1-cut-outright.md#s12--collapse-25-mcp-tools-to-9)). Retrieval is layered,
deterministic, and fully local. The shared `kblib` package powers both the MCP server
and the `kb` CLI.

![MCP Server Architecture](./diagrams/mcp-server-architecture.svg)

```bash
kb serve                           # SSE on http://localhost:8787
MCP_PORT=9000 kb serve             # custom port
python3 multi-dimensional-kb/graph-mcp/server.py --stdio   # stdio transport
```

At startup: loads all YAML into memory → derives RAG chunks from `markdown/` pages →
builds BM25 index → builds dense MiniLM ONNX embeddings (BM25-only fallback with a
startup notice if unavailable) → caches in `.kb_index/`. Startup: <2s warm, ~20s cold.

---

## 9 Unified MCP Tools

### Enumerate & Fetch (2)

| Tool | Use Case |
|------|----------|
| `kb_list(type, filter?)` | Enumerate: `services`, `flows`, `adrs`, `metadata`, `api-contracts` (filter=service), `apis` (filter=path pattern) |
| `kb_get(type, name)` | Fetch one: `service`, `flow`, `adr`, `metadata`, `nfr`, `api` (`service/operationId`), `architecture-layers`, `document` (`wiki`/`bruno`/`service-wiki` stores) |

### Search (2)

| Tool | Use Case |
|------|----------|
| `kb_smart_query(query)` | Intelligent router — classifies → routes to best tool |
| `kb_search(query, mode?)` | Hybrid BM25 + dense ranked search over derived chunks |

### Analysis (2)

| Tool | Use Case |
|------|----------|
| `kb_impact(name)` | Blast-radius analysis: upstream/downstream + impact page |
| `kb_compare(a, b)` | Side-by-side service comparison |

### OKF Navigation (2)

| Tool | Use Case |
|------|----------|
| `kb_navigate(path?)` | Walk hierarchical indexes (OKF progressive disclosure) |
| `kb_read_concept(id)` | Read one concept with parsed frontmatter + trust tier |

### Ops (1)

| Tool | Use Case |
|------|----------|
| `kb_health()` | Server health + index stats + active retrieval mode |

---

## Smart Query Routing

`kb_smart_query` classifies queries and routes to the optimal retrieval:

![Retrieval Architecture](./diagrams/retrieval-architecture.svg)

| Route | Pattern | Method |
|-------|---------|--------|
| **Analytical** | "impact of X", "blast radius" | Graph traversal + impact chunks |
| **Navigational** | "list APIs of X" | Deterministic catalog lookup (O(1)) |
| **Factual** | Contains service name / operationId | Deterministic detail lookup (O(1)) |
| **Exploratory** | "how", "why", open-ended | Hybrid BM25 + dense search |

Classifier accuracy: **>95%** on labeled test cases.

---

## Hybrid Search Pipeline

```
Query → tokenize → ┬─ BM25:  term relevance (α=0.4)              → Ranked List A
                    └─ Dense: MiniLM ONNX 384d cosine (β=0.6)
                              (skipped if unavailable — BM25-only)  → Ranked List B
                                                              ↓
                    Weighted RRF: α/(60+rankA) + β/(60+rankB) → Merged
                                                              ↓
                    Lexical Rerank: entity/section/disclosure  → Final Top-K
```

- **BM25** — `rank_bm25.BM25Okapi`, best for exact terms (service names, operationIds)
- **Dense** — all-MiniLM-L6-v2 ONNX (committed in-repo, no downloads, CPU-only).
  Captures paraphrases with no keyword overlap.
- **BM25-only fallback** — when ONNX deps are unavailable, retrieval runs on the lexical
  tier alone and prints a clear notice at startup

### Search Modes

| Mode | Retrievers | Best For |
|------|-----------|----------|
| `hybrid` *(default)* | BM25 + dense → W-RRF (BM25-only if dense unavailable) | General queries |
| `bm25` | BM25 only | Exact keywords |
| `dense` | MiniLM only | Conceptual/paraphrase |

### RAG Chunking

Chunks are a **build artifact** ([S1.3](./simplification/01-tier-1-cut-outright.md#s13--stop-committing-chunks-1291-derived-files)): derived in-memory from the committed
`markdown/` pages (embedded `<!-- chunk:... -->` markers) at index/startup time by
`kblib.chunker.derive_chunks()` — currently 1,291; nothing is committed. Each carries HTML
comment metadata (`entity-type`, `entity`, `section`, `tokens`). **8 KB ceiling** per
chunk — oversized pages split at heading boundaries. Debug on disk with
`kb generate --show-chunks` (writes to `.kb_index/chunks/`).

---

## CI Pipeline

![CI Pipeline](./diagrams/ci-pipeline.svg)

Defined in `.github/workflows/kb-validation.yml`. Fully deterministic — **$0 LLM cost**.

### Triggers

| Trigger | When | Jobs |
|---------|------|------|
| **pull_request** | PR touching `sources/`, `scripts/`, `graph-mcp/`, `markdown/`, etc. | validate, mcp-tests, cli-smoke, pr-comment |
| **push (main)** | Push to `main` | validate, mcp-tests, cli-smoke, publish |
| **schedule** | Monday 06:00 UTC | weekly-audit |

### Job 1: `validate` — KB Quality Gate

1. **Unified quality check** — `kb check` runs schema validation (6 JSON schemas),
   health audit (AUD-01…25, score ≥ 60, 0 errors), and OKF conformance (C1–C12) in one
   command → `check-report.md`
2. **Sync gate** — regenerates ALL markdown, diffs vs committed. **Fails on any diff** — no stale pages reach `main`
3. **Viz build** — ensures interactive graph generates
4. **AID coverage** — API contract sample report (informational)

### Job 2: `mcp-tests` — Retrieval Quality (parallel)

- **9-tool self-test** (`test_tools.py`)
- **54 golden queries** (`test_retrieval.py`) — fails on regression

| Metric | Target | Current |
|--------|--------|---------|
| Recall@5 | ≥ 0.90 | **0.982** |
| Recall@10 | ≥ 0.95 | **1.0** |
| MRR | ≥ 0.80 | **0.912** |

- **Classifier accuracy** (`test_smart_query.py`) — >85% required

### Job 3: `cli-smoke` — CLI Smoke Test (parallel)

Verifies all 17 `kb` subcommands (including `kb check`), runs `kb search` and `kb context` smoke tests.

### Job 4: `pr-comment` — PR Score (after validate)

Posts audit score on the PR: 🟢 ≥ 80, 🟡 ≥ 60, 🔴 < 60.

### Job 5: `publish` — Artifacts (main push)

Uploads `audit-report.md`, `audit-metrics.json`, `viz.html` (30-day retention).

### Job 6: `weekly-audit` — Scheduled Health Check

Full audit every Monday — creates a GitHub issue on failure.

---

## Quality Gate Summary

| Gate | Tool | Pass Criteria |
|------|------|--------------|
| Unified check | `kb check` (schema + audit + OKF) | 0 violations, score ≥ 60, 0 errors, C1–C12 pass |
| Schema | `validate_catalog.py` (alias: `kb validate`) | 0 violations |
| Audit | `audit_kb.py` (alias: `kb audit`) | Score ≥ 60, 0 errors |
| OKF | `okf_conformance_check.py` (alias: `kb conformance`) | C1–C12 all pass |
| Sync | `generate + git diff` | No diff |
| Retrieval | `test_retrieval.py` | No regression |
| CLI | smoke tests | All subcommands work |

---

## MCP Resources & Prompts

**Resources**: `kb://services`, `kb://stats`, `kb://subdomains`

**Prompt Templates**: `kb_explore`, `kb_investigate_service`, `kb_compare`

---

**Next**: [04 — Consumer Workflows](./04-consumer-workflows.md) | **Back**: [02 — Setup & Daily Operations](./02-getting-started.md)

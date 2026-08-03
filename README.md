# Commerce Multi-Dimensional Knowledge Base — Documentation

> **Version**: 2.0.0 (`kb-framework`) | **Domain**: Commerce (apm0045079) | **Stack**: Python 3.10+, PyYAML, FastMCP, numpy
>
> **Standards**: OKF v0.2 knowledge bundle + LLM-Wiki three-layer architecture

---

## What Is This?

The Commerce Knowledge Base transforms **45 scattered Java microservice repos**, Confluence
wiki pages, Bruno API collections, ADRs, NFRs, and curated metadata into a **unified,
cross-linked Markdown knowledge base** that both humans and AI agents can navigate.

![KB Pipeline](./diagrams/kb-generation-pipeline.svg)

| Dimension | Count |
|-----------|-------|
| Microservices | 45 across 11 subdomains |
| Inbound APIs | 830 endpoints |
| API Contracts | 686 with scenarios |
| Capability Flows | 127 (1 primary + 115 Bruno + 11 curated groupings) |
| ADRs | 5 architecture decisions |
| NFR Profiles | 45 |
| RAG Chunks | 1,291 derived at index time (BM25 + dense MiniLM-ONNX 384d, BM25-only fallback) |
| MCP Tools | 9 unified query tools |

---

## How It Works

```
seed.yaml                              ← YOU EDIT THIS (declare all inputs)
    ↓
10 Workflows (LLM agent)              ← scans code, crawls wiki, parses Bruno
    ↓
{KB_ROOT}/sources/                     ← Intermediate YAML (structured, validated)
    ↓
kb generate (deterministic Python)     ← no LLM — pure template rendering
    ↓
{KB_ROOT}/markdown/                    ← Final Markdown KB (read-only, fully cross-linked)
    ↓
graph-mcp/server.py                    ← 9 unified MCP tools, hybrid search, in-memory
```

**One rule**: edit `seed.yaml` → run workflows → everything else is derived.

---

## Quick Start

```bash
# 1. One-time setup (from repo root)
python3 -m venv .venv && source .venv/bin/activate
pip install -e "multi-dimensional-kb/[all]"
kb index                                     # warm RAG cache (~20s)

# 2. Verify
kb stats
kb search "shopping cart API" --top 5
kb serve                                     # starts MCP on http://localhost:8787

# 3. Bootstrap the full KB (first time only — runs in IDE chat)
/lifecycle-kb --bootstrap

# 4. Daily: ingest one changed service (IDE chat), then regenerate
/ingest-service shoppingcartms
/generate-kb --scope=shoppingcartms
```

---

## Chapters

| Chapter | What It Covers |
|---------|---------------|
| **[01 — Architecture & Data Model](./01-architecture.md)** | Three-layer architecture, OKF/LLM-Wiki patterns, data flow, seed.yaml, schemas, frontmatter contract |
| **[02 — Setup & Daily Operations](./02-getting-started.md)** | Prerequisites, install, bootstrap, `kb` CLI reference, workflows, recipes, FAQ |
| **[03 — MCP, Search & CI](./03-mcp-and-retrieval.md)** | 9 unified MCP tools, hybrid retrieval pipeline, CI pipeline, quality gates |
| **[04 — Consumer Workflows](./04-consumer-workflows.md)** | Story generation, execution plans, scaffold, E2E feature implementor, evaluation |
| **[05 — Scaling & Federation](./05-scaling-and-federation.md)** | Multi-domain federation (customer, order, …), ownership model, registry, gateway MCP, leadership defense with industry citations |
| **[06 — Freshness, CI/CD & Evaluation](./06-freshness-and-evaluation.md)** | Corrections to the original design, three-trigger freshness model, five-layer evaluation stack, monitoring stack |
| **[07 — Phase 2 Target Architecture](./07-phase-2-target-architecture.md)** | Central pgvector tier, retrieval v2, orchestration, online judge, migration triggers |
| **[Simplification](./simplification/README.md)** | Minimum viable framework — tiered cut list (cut outright / slim by default / defer until trigger / do not cut), each recommendation with proof |
| **[Head-First-Style Book](./head-first-design/README.md)** | The friendly companion: the whole framework in 9 visual, conversational chapters — for new joiners and leadership |

---

## Diagrams

All diagrams are in [`./diagrams/`](./diagrams/):

| Diagram | Shows |
|---------|-------|
| [`okf-three-layer.svg`](./diagrams/okf-three-layer.svg) | Three-layer architecture |
| [`kb-generation-pipeline.svg`](./diagrams/kb-generation-pipeline.svg) | End-to-end pipeline |
| [`seed-yaml-anatomy.svg`](./diagrams/seed-yaml-anatomy.svg) | seed.yaml sections |
| [`retrieval-architecture.svg`](./diagrams/retrieval-architecture.svg) | Smart query routing + hybrid search |
| [`workflow-folder-map.svg`](./diagrams/workflow-folder-map.svg) | Workflow-to-folder mapping |
| [`mcp-server-architecture.svg`](./diagrams/mcp-server-architecture.svg) | MCP server internals |
| [`ci-pipeline.svg`](./diagrams/ci-pipeline.svg) | CI pipeline (6 jobs, 3 triggers) |
| [`story-generation-flow.svg`](./diagrams/story-generation-flow.svg) | Story generation tri-channel flow |
| [`e2e-feature-lifecycle.svg`](./diagrams/e2e-feature-lifecycle.svg) | Full feature lifecycle |
| [`federation-topology.svg`](./diagrams/federation-topology.svg) | Multi-domain federation: domain repos, framework package, registry, gateway |
| [`freshness-triggers.svg`](./diagrams/freshness-triggers.svg) | Three-trigger freshness model + SLOs |
| [`evaluation-loop.svg`](./diagrams/evaluation-loop.svg) | Five-layer evaluation stack (E1–E5) + feedback loop |
| [`phase2-target-architecture.svg`](./diagrams/phase2-target-architecture.svg) | Phase 2: central index, orchestrator, online judge |

---

## Key Files

| File | Purpose |
|------|---------|
| `templates/seed.yaml` | **THE entry point** — declares all KB sources |
| `scripts/generate_multidim_kb.py` | Deterministic markdown generation (3100+ lines) |
| `scripts/validate_catalog.py` | Schema validation + cross-ref integrity |
| `scripts/audit_kb.py` | Full KB health audit (AUD-01…25 + D1–D10) |
| `scripts/okf_conformance_check.py` | OKF v0.2 conformance (C1–C12) |
| `graph-mcp/server.py` | MCP server (9 unified tools, hybrid search) |
| `kbcli/cli.py` | Unified `kb` CLI entry point |
| `.devin/workflows/multi-dimensional-kb/` | 10 workflow definitions |
| `.github/workflows/kb-validation.yml` | CI pipeline (6 jobs) |

---

**Owner**: Commerce Architecture Team | **Repository**: `apm0045079-commerce-AIFC-knowledge-base`

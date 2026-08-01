# Simplification 01 — Tier 1: Cut Outright

> **Audience**: Maintainers executing the cuts. Every item here can be removed **now** —
> the evidence already exists, no new measurement is required, and nothing the framework
> promises is lost. Each item follows the same template: current state → what it costs →
> the change → the proof → trade-off → verification.

---

## S1.1 — Delete the LSA Fallback Path

**Current state.** The retrieval pipeline maintains a *second* dense implementation:
TF-IDF → numpy Truncated SVD → 128-d vectors, exposed as `--mode lsa`, documented as the
tier used "when ONNX dependencies are unavailable"
([Chapter 03](../03-mcp-and-retrieval.md#hybrid-search-pipeline)). The minimal install
profile (`pip install -e "multi-dimensional-kb/"`) exists specifically to serve it.

**What it costs.** A full parallel retriever: its own vectorizer, its own SVD
factorization and cache, its own search mode, its own tests, its own documentation row —
and a permanent tax on every future retrieval change, which must now be designed and
tested against **three** dense/lexical combinations instead of two.

**The change.** Remove LSA entirely. The fallback ladder becomes:
hybrid (BM25 + ONNX dense) → **BM25-only**. One fallback, not two.

**The proof.**

1. *The scenario it guards is engineered away.* LSA exists for "ONNX deps unavailable."
   But the MiniLM model is **committed in-repo** (no download), and `onnxruntime` +
   `tokenizers` are ordinary PyPI wheels with prebuilt binaries for every platform we
   run — the same corporate pip that installs `numpy` and `PyYAML` installs them. The
   zoomcamp [ONNX lesson](https://github.com/DataTalksClub/llm-zoomcamp/blob/main/02-vector-search/lessons/09-onnx-embedder.md)
   documents this exact runtime as the *minimal* deployment profile (147 MB, 27
   packages), not an exotic one.
2. *When the scenario does occur, LSA is the wrong answer anyway.* A 128-d SVD over
   TF-IDF of 1,291 chunks is a much weaker semantic space than MiniLM. A user whose
   environment silently drops from MiniLM to LSA gets **silently degraded quality** —
   the worst failure mode, because nothing tells them. BM25-only is the honest floor:
   its behavior is well-understood, it excels at this corpus's exact-term queries
   (service names, operationIds), and the zoomcamp curriculum explicitly endorses
   keyword-only as a legitimate v1
   ([text-search-first argument](https://github.com/DataTalksClub/llm-zoomcamp/blob/main/02-vector-search/lessons/10-next-steps.md)).
3. *The golden suite never certified LSA.* The published numbers (Recall@5 = 0.98,
   MRR = 0.91) were measured on the hybrid pipeline. There is no evidence LSA meets any
   gate — so keeping it as a "safety net" is keeping an **unmeasured** code path in the
   serving stack, which contradicts the framework's own evaluation-first principle.

**Trade-off & mitigation.** An environment that truly cannot install `onnxruntime` loses
dense retrieval entirely. Mitigation: `kb health` and `kb stats` already report the
active mode — make BM25-only mode print a one-line notice, and run the E2 retrieval gate
in BM25-only configuration once so the floor is a *measured* floor.

**Verification.** Delete the code path; run the golden retrieval suite in `hybrid` and
`bm25` modes (both must pass their gates); confirm `kb search --mode lsa` fails with a
clear message; count the removed lines/tests as the reclaimed complexity.

---

## S1.2 — Collapse 25 MCP Tools to 9

**Current state.** [Chapter 03](../03-mcp-and-retrieval.md#25-mcp-tools) lists 25 tools.
Three are *already deprecated aliases* (`graph_get_wiki_content`,
`graph_get_service_wiki`, `graph_get_bruno_collection` → all folded into
`graph_get_document`). Of the rest, more than half are per-entity `list`/`get` pairs
that differ only in which YAML dictionary they read: services, API contracts, flows,
ADRs, metadata, NFRs.

**What it costs.** Tool definitions are not free — every tool's name, description, and
JSON schema is serialized into **every agent session's context**, on every call, for
every consumer. 25 schemas also degrade tool *selection*: the more overlapping options,
the more often an agent picks a suboptimal one or hesitates between near-duplicates.
And every tool is code + tests + docs + a row in the trajectory-eval rubric.

**The change.** Nine tools:

| New Tool | Replaces | Notes |
|----------|----------|-------|
| `kb_list(type, filter?)` | `graph_list_services`, `graph_list_api_contracts`, `graph_list_flows`, `graph_list_adrs`, `graph_list_metadata` | `type` is a closed enum; description enumerates valid values |
| `kb_get(type, name)` | `graph_get_service`, `graph_get_api_detail`, `graph_get_flow`, `graph_get_adr`, `graph_get_metadata`, `graph_get_nfr`, `graph_get_document` | Returns the same unified payloads |
| `kb_search(query, mode?, type?)` | `graph_semantic_search` | Unchanged behavior |
| `kb_smart_query(query)` | `graph_smart_query` | The router — unchanged |
| `kb_navigate(path)` | `graph_navigate` | OKF progressive disclosure |
| `kb_read_concept(id)` | `graph_read_concept` | Frontmatter + trust tier |
| `kb_impact(name)` | `graph_get_impact`, `graph_find_service_dependencies` | Dependencies are the depth-1 case of impact |
| `kb_compare(a, b)` | `graph_compare_services` | Kept — genuinely distinct output shape |
| `kb_health()` | `graph_health` | Unchanged |

(`graph_find_apis_by_path` and `graph_get_architecture_layers` become `filter` variants
of `kb_list`/`kb_get` — path-pattern filter, layers section.)

**The proof.**

1. *The framework already voted for this.* The three deprecated aliases and the unified
   `graph_get_document(store=...)` show consolidation-by-parameter already happened once
   and worked — S1.2 just finishes the job.
2. *Tool-design practice agrees.* The zoomcamp
   [function-calling lesson](https://github.com/DataTalksClub/llm-zoomcamp/blob/main/01-agentic-rag/lessons/13-function-calling.md)
   states the operative rule: *"the `description` is the most important field, because
   the model reads it to decide when to call the function."* Selection quality lives in
   descriptions, not in tool count — 9 well-described tools give the model less to
   confuse and less schema to carry. MCP ecosystem guidance runs the same direction:
   consolidate CRUD-shaped variants behind typed parameters.
3. *The router absorbs the difference.* `kb_smart_query` already classifies and routes
   >95% of queries correctly ([Chapter 03](../03-mcp-and-retrieval.md#smart-query-routing));
   most agent traffic never needed 25 entry points in the first place.
4. *E5 gets cheaper.* The [trajectory rubric](../06-freshness-and-evaluation.md#e5--trajectory-eval-mcp-tool-use)
   over 9 tools is dramatically easier to judge and to explain than over 25.

**Trade-off & mitigation.** Per-entity tools are marginally more discoverable than an
enum parameter. Mitigation: `kb_list`/`kb_get` descriptions enumerate every valid
`type`, and an invalid `type` returns the valid list in the error — the agent self-corrects
in one turn. Keep the old names as hidden aliases for one release, then delete.

**Verification.** Re-run the 25-tool self-test as a 9-tool self-test; re-run the golden
query suite through `kb_smart_query` (results must be identical — routing is internal);
measure serialized tool-schema tokens before/after (expect a 50–70% reduction per
session).

---

## S1.3 — Stop Committing `chunks/` (1,291 Derived Files)

**Current state.** `markdown/chunks/` holds 1,291 chunk files, committed to git, covered
by the CI sync gate, guarded by AUD-22 (oversize) and AUD-23 (metadata)
([Chapter 03](../03-mcp-and-retrieval.md#rag-chunking)).

**What it costs.** A one-line change to a service's `catalog-info.yaml` regenerates the
service page *plus every chunk derived from it* — PRs bloat with derived-file noise,
reviewers scroll past chunk diffs (training themselves to rubber-stamp, which erodes the
very review value that committing markdown was meant to provide), git history grows
permanently, and the sync gate diffs ~1,300 extra files on every run. Federation
multiplies all of it by the number of domains.

**The change.** Chunks become a **build artifact**: `kb index` derives them from the
committed markdown pages at index time (locally and in CI). Pages stay committed —
review, diffs, and the sync gate continue to operate on *pages*. The CI `publish` job
may still include chunks in the published bundle tarball for downstream consumers
(e.g., the Phase 2 pgvector loader).

**The proof.**

1. *Chunks are a pure function of pages.* The chunker is deterministic (heading-boundary
   splits + the 8 KB ceiling, [Chapter 01](../01-architecture.md#pattern-6-one-concept-per-chunk)).
   A deterministic function of committed input needs no separate copy in version
   control — the input plus the (versioned) function *is* the record. This is the
   docs-as-code norm: commit sources, build artifacts in CI.
2. *The design already believes this — one layer up.* [Chapter 07](../07-phase-2-target-architecture.md)
   declares "the bundle stays the product; **indexes are disposable caches**." Chunks
   are the first index artifact, not the last page artifact. S1.3 applies the Phase 2
   principle consistently.
3. *Nothing consumer-facing changes.* `kb search` / `kb context` already build
   `.kb_index/` locally; they simply gain the chunk-derivation step (~seconds). MCP
   server startup already rebuilds its index from files at boot.

**Trade-off & mitigation.** Chunk-level diffs disappear from PRs (they were noise, but
occasionally useful for debugging the chunker). Mitigation: `kb generate --dry-run`
gains a `--show-chunks` flag for chunker debugging; AUD-22/23 move from "check committed
files" to "check built artifacts" inside the same CI job — the checks survive, only
their input path changes.

**Verification.** Build chunks on two machines from the same commit and hash-compare
the outputs (must be byte-identical — this *proves* determinism instead of assuming
it); confirm repo size and average PR diff size drop; confirm the golden retrieval
suite passes against built-at-index-time chunks.

---

## S1.4 — Version Planes: 4 → 2

**Current state.** [Chapter 01](../01-architecture.md#version-planes) declares four
independently bumped versions: OKF frontmatter contract (0.2), Framework (9.0.0), KB
content (7.1.0), and the `kb` CLI (1.0.0) — with an explicit warning that they "must
not be conflated," which is itself evidence of the confusion risk.

**What it costs.** Four bump decisions per change, four changelogs to reconcile, and a
recurring governance question ("which plane does this touch?") that every contributor
must answer correctly. Federation would multiply the confusion: FED-04 must compare
versions across domains, and four planes give it four axes to compare.

**The change.** Two planes, matching the two artifacts that actually ship:

| Plane | Covers | Where |
|-------|--------|-------|
| **`kb-framework` package semver** | scripts, CLI, MCP server, JSON schemas, templates, *and the OKF contract the generator emits* | Internal PyPI |
| **Bundle version** (per domain) | Content — stamped as `generated.by: <domain>-kb-generator/<version>` | `registry.yaml` + frontmatter |

**The proof.**

1. *The four planes can no longer vary independently.* After the
   [package extraction](../05-scaling-and-federation.md#the-four-repositories-of-the-federation),
   the CLI, schemas, templates, and generator ship in one wheel — a schema change
   *requires* a framework release; the CLI cannot version apart from the package that
   contains it. Separate version numbers for things that release together describe a
   freedom that no longer exists.
2. *The OKF contract is a property, not a plane.* The generator either emits OKF v0.2
   bundles or it doesn't — that fact is determined by the framework version (the
   generator code), so it belongs *inside* the framework's changelog ("2.3.0: emits OKF
   0.3"), exactly how libraries declare the protocol versions they speak.
3. *Industry norm.* Platform packages carry one semver; content carries its own
   version. Nobody versions a compiler's CLI separately from the compiler.

**Trade-off & mitigation.** Coarser signaling — a CLI-only fix bumps the whole framework
patch version. That is how every packaged tool on the planet works; the changelog
section names what changed.

**Verification.** `registry.yaml` needs exactly two version fields per domain
(framework pin, bundle version); FED-04 compares one axis; grep the docs for the
four-plane table and replace it with the two-plane table.

---

## S1.5 — Three Quality Runners → One `kb check`

**Current state.** Three separate deterministic scripts walk the same trees:
`validate_catalog.py` (schemas + cross-refs), `audit_kb.py` (AUD-01…25, D1–D10),
`okf_conformance_check.py` (C1–C12) — three invocations, three report formats, three CI
wiring points ([Chapter 01](../01-architecture.md#operational-governance--log-lint-audit-evaluation)).

**What it costs.** Overlap and drift. C6 (cross-links resolve) overlaps audit's link
checking; C2/C3 (frontmatter present/required fields) overlaps AUD-23 (chunks missing
metadata); index-coverage exists as both C8 and D8. Overlapping rules implemented twice
*will* diverge — one gets fixed, the other doesn't — and then two gates disagree about
the same bundle, which is worse than one gate being wrong.

**The change.** One runner: `kb check` — single pass over `sources/` + `markdown/`,
emitting one report (`check-report.md`) and one metrics file, with **all existing check
IDs preserved** (AUD-*, C-*, D-*) so history, docs, and muscle memory stay valid.
`kb validate`, `kb audit`, `kb conformance` become thin aliases that filter the same
run's output during a deprecation window.

**The proof.**

1. *They are the same program.* All three are deterministic Python that parse the same
   YAML/markdown and assert predicates. Merging them is refactoring, not redesign — the
   check *logic* is untouched.
2. *Precedent.* This is exactly the linter-consolidation pattern the Python ecosystem
   just lived through: Ruff absorbed the rule sets of a dozen tools while **keeping
   their rule codes**, and adoption was driven by "one pass, one config, one output."
   Same shape, same benefit: one tree walk instead of three cuts CI wall-clock, and one
   implementation per rule ends the drift risk.
3. *CI gets simpler and stricter at once.* The `validate` job's four steps become one
   step whose report contains everything — fewer places for a gate to be accidentally
   skipped or mis-ordered.

**Trade-off & mitigation.** One larger module instead of three small scripts.
Mitigation: internal structure stays modular (`checks/schema.py`, `checks/audit.py`,
`checks/okf.py`) behind the single entry point — the consolidation is at the
*invocation and report* level, where the duplication hurts.

**Verification.** Seed a fixture bundle with one known violation per check family;
`kb check` must report all of them with their original IDs; compare CI wall-clock
before/after; delete the three old entry points after one deprecation release.

---

## Tier 1 Scorecard

| Cut | Removes | Risk |
|-----|---------|------|
| S1.1 LSA | 1 retriever, 1 mode, its cache + tests | None (unmeasured path today) |
| S1.2 Tools 25→9 | 16 tool surfaces, ~60% schema tokens/session | 1-turn self-correction on bad enum |
| S1.3 Chunks uncommitted | 1,291 files from git + sync-gate surface | Chunker debugging flag needed |
| S1.4 Planes 4→2 | 2 version axes + the confusion tax | Coarser changelog granularity |
| S1.5 One runner | 2 scripts, 3 report formats, drift risk | One-time refactor effort |

---

**Next**: [02 — Tier 2: Slim by Default](./02-tier-2-slim-by-default.md) | **Up**: [Simplification README](./README.md)

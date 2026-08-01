# 1 — Your knowledge is rotting (and your robot knows it)

*In which Sam has a terrible first day, Botty invents an API that doesn't exist, and we
discover that the problem isn't missing documentation — it's documentation you can't
trust.*

---

## Sam's first day

Sam joins the Commerce team. First task: "add a field to the shopping cart API." Simple,
right? Sam asks the obvious question — *"where do I find what shoppingcartms actually
does?"* — and gets five answers:

1. "There's a Confluence page… from 2023. Ish."
2. "Read the code. All 45 repos of it."
3. "Ask Priya. Oh wait, Priya left."
4. "There's a Bruno collection somewhere with example calls."
5. "The wiki says port 8081 but it's actually 8443 now, ignore that part."

![Knowledge rot](./images/knowledge-rot.svg)

Five sources. All partially right. All partially wrong. **Nobody knows which parts.**

That's not "we should document more." That's *knowledge rot*: the docs exist, but their
relationship with reality decays silently, and no alarm goes off.

## Then it got urgent

Here's the thing that turned a chronic annoyance into a crisis: **we started asking AI
agents to write our stories, plans, and code.**

Sam wastes an afternoon on a stale wiki page and gets annoyed. Botty reads the same page
and — with total confidence, in milliseconds, at scale — writes a story against an API
that was renamed eight months ago. Last Tuesday, Botty cheerfully invented
`getCartDiscountsV2`. It does not exist. It has never existed.

> **Your brain on stale docs:** humans cross-check, hesitate, ask a teammate. LLMs do
> none of that by default. An AI agent is a *massive amplifier* — of whatever you feed
> it. Feed it rot, get confident rot at machine speed.

## The big idea: treat knowledge like code

We already solved this exact problem once — for *software*. We don't email each other
zip files of source code and hope; we have a single source of truth, derived builds,
tests, and CI that blocks bad changes. So:

| Software engineering has… | So the knowledge base has… |
|---------------------------|----------------------------|
| One source of truth (git) | One declared input file (`seed.yaml`) + structured sources |
| Deterministic builds | A boring Python generator: same input, byte-identical output |
| Tests that block bad merges | Exams: does search still find the right page? (Chapter 7) |
| Code review + ownership | Every change is a PR; every fact has an owner |
| Dependency freshness bots | A nightly sniff-test for changed services (Chapter 6) |

And the output format? **Markdown files.** Not a database, not a wiki product — plain
files with a little YAML on top. Humans read them on any git host. Robots read them
without choking on wiki markup. Google literally published a spec for exactly this
pattern (it's called OKF — the serious docs have receipts).

## There are no Dumb Questions

**Q: Isn't Confluence enough? We already pay for it.**
A: Confluence is where knowledge goes to *be written once*. Nothing checks it against
the code, nothing expires it, and agents drown in its markup. We still *ingest* useful
wiki pages — Confluence becomes an *input*, not the source of truth.

**Q: Why is markdown suddenly a big deal? It's just text files.**
A: That's the point. Text files diff, review, version, and grep. Serving markdown to
agents instead of HTML cuts token cost ~10× — big companies (Cloudflare, Stripe,
Anthropic) already serve their docs to AI this way.

**Q: Couldn't we just tell developers to keep docs updated?**
A: We could also just tell everyone to write bug-free code. Process that depends on
human diligence loses to process that runs in CI. Every time.

## BULLET POINTS

- Knowledge rot is silent: docs don't *look* wrong, they just quietly stop being right.
- AI agents turn stale docs from an annoyance into confident, scaled-up wrongness.
- The fix is the software playbook applied to knowledge: one source of truth,
  deterministic builds, tests, CI, ownership.
- Markdown + YAML frontmatter is the output format because humans *and* robots read it
  natively — and it lives happily in git.

**Want the full spec?** [Main README](../README.md) ·
[Chapter 01 — Architecture](../01-architecture.md)

**Next**: [2 — The three-layer cake](./02-three-layers.md)

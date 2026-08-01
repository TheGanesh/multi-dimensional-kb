# 5 — Teaching robots to ask nicely (MCP)

*In which Botty stops rummaging through the kitchen, learns to order from a menu, and
we find out why nine dishes beat twenty-five.*

---

## The kitchen problem

Before MCP, when Botty needed to know something about `shoppingcartms`, it did what any
eager robot does: **walked into the kitchen and started opening drawers.** Read this
file, follow that link, grep for a promising-looking string, read four more files just
in case…

Result: Botty's brain (its *context window* — which is finite and expensive) fills up
with drawer contents. Whole pages, navigation menus, sections about the wrong service.
By the time Botty finds the answer, it's carrying so much clutter it starts confusing
the pricing service with the promo service.

The fix is the oldest one in hospitality: **don't let guests into the kitchen. Give
them a menu.**

![The MCP menu](./images/mcp-menu.svg)

## MCP: the menu protocol

**MCP (Model Context Protocol)** is a standard way for AI agents to call tools —
published as an open standard, governed by the Linux Foundation, and spoken by
basically every AI vendor (OpenAI, Google, Microsoft, the lot). That last part matters:
because it's a *standard* menu format, any waiter (Claude, Copilot, Gemini, your IDE's
agent) can read it. We didn't build a custom API that every tool would need custom
glue for — we printed a menu in the language every robot already reads. About a quarter
of the Fortune 500 already runs MCP servers; this is the industry's pick, not ours.

Our menu has **nine dishes**:

| The dish | What Botty gets |
|----------|-----------------|
| `kb_list(type)` | "What services / APIs / flows / decisions exist?" |
| `kb_get(type, name)` | The full detail on one thing — schemas, dependencies, error codes |
| `kb_search(query)` | Ranked results from the stacks (Chapter 4) |
| `kb_smart_query(query)` | "Just handle it" — the librarian routes it |
| `kb_navigate(path)` | Browse the shelves, index by index |
| `kb_read_concept(id)` | One page, with its business card parsed (and a staleness warning!) |
| `kb_impact(name)` | "What breaks if I change this?" |
| `kb_compare(a, b)` | Two services, side by side |
| `kb_health()` | "Is the kitchen even open?" |

Every result comes back **structured** — typed fields, not prose to re-parse — stamped
with its `kb://` address and its trust label. When a page is past its expiry date,
`kb_read_concept` *says so*, right in the response. The menu never bluffs about
freshness.

## Why 9 dishes and not 25?

The first version of this menu had 25 items. Sounds generous — it was actually worse,
for the same reason a nine-page diner menu is worse:

- **Choice paralysis, robot edition.** Give a model 25 overlapping tools and it picks a
  near-duplicate, or dithers. Give it 9 distinct ones and it orders correctly.
- **Menus take up table space.** Every tool's name + description + schema is loaded
  into Botty's brain *on every session*. Twenty-five schemas of clutter, before the
  first question is even asked.
- **The description is the dish.** The single most important line of an MCP tool is its
  *description* — that's what the model reads to decide when to order it. Nine
  well-written descriptions beat twenty-five mumbled ones.

(Fifteen list/get lookalikes got folded into `kb_list`/`kb_get` with a `type`
parameter. Order the dumplings, specify the filling.)

## There are no Dumb Questions

**Q: Why not just a REST API? We build those all day.**
A: A REST API needs custom client code in every consumer. MCP needs *zero* — the
agent's runtime already speaks it, the same way your browser already speaks HTTP. We
publish one server; every AI tool in the company can call it tomorrow.

**Q: What if Botty orders the wrong dish — or the same dish five times?**
A: It happens! Tool calls are logged (every order goes on the receipt), and there's a
whole grading rubric for tool-use behavior waiting in Chapter 7. Also: if Botty asks
`kb_get` for a type that doesn't exist, the error message lists the valid types —
Botty self-corrects on the next order.

**Q: Does Botty *have* to use the menu? What about the files directly?**
A: The files stay readable — humans browse them all the time. But agent workflows use
the menu (or the offline `kb` CLI): it's measured, structured, and doesn't fill
Botty's brain with drawer clutter. Rummaging is neither.

## BULLET POINTS

- MCP = a standard menu format for AI agents; every major vendor's robots can read it,
  so one server serves them all.
- Structured answers with addresses and trust labels beat "here's a wall of text,
  good luck."
- 9 tools > 25 tools: less schema clutter in the context window, better ordering
  decisions, one great description per dish.
- The menu is honest about freshness — expired pages come with a warning label.

**Want the full spec?** [Chapter 03 — MCP, Search & CI](../03-mcp-and-retrieval.md) ·
[S1.2 — the 25→9 consolidation](../simplification/01-tier-1-cut-outright.md#s12--collapse-25-mcp-tools-to-9)

**Next**: [6 — The milk carton principle](./06-freshness.md) | **Back**: [4 — Finding stuff](./04-search.md)

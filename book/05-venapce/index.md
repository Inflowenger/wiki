# Part V — Venapce

> **A product built on the product.**
>
> **Status: outlined.** Venapce is in active development; chapters will track it.

If [FloMorphic](../04-flomorphic/) proves a product can be built on the runtime, Venapce
proves something one layer up: **a product can be built on a product built on the runtime**,
without touching the runtime at all.

Venapce implements no backend contract, writes no compiler hook, and calls
`inflow.NewProcess` nowhere. **All of its business logic lives in FloMorphic workflows.**
Venapce itself is only the view.

That makes it a third integration tier, and the most interesting one for anybody evaluating
the platform as a foundation:

| Tier | What you write | Example |
| --- | --- | --- |
| **1 — SDK** | a backend implementing `IInflowService`, a compiler hook, extrinsic handlers | FloMorphic, inflow-inspector |
| **2 — Plugin** | a process speaking `inflowv1` | Jira, Postgres, `llm`, osctrl |
| **3 — Workflows** | **no runtime code at all** — logic as flows on an existing host | **Venapce** |

## What Venapce is

> A nervous system for security governance.
> *Vein — the agents across your compute. Synapse — the signal when something's wrong.*

Every security team is drowning in tools and starved of connective tissue. Endpoints,
network gear, cloud consoles, the ticketing system — each is a black box that already
decided what matters, and none of them talk. Venapce is the missing layer.

```
Senses — plugins & osquery agents
  Reach each system, return raw data frames. No opinions, no scoring, no stored secrets.
        ↓
Nervous system — FloMorphic workflows
  Correlate, enrich & evaluate per feature; an LLM node reasons in the open.
  ALL business logic lives here.
        ↓
The face — Venapce
  Dashboards, fleet view, issues, actions. Posture you can see; a panel you act from.
```

## Chapters

| Chapter | What it establishes |
| --- | --- |
| [What Venapce is](what-it-is.md) | The product, the data model, the assistant |
| [Built under FloMorphic](built-on-flomorphic.md) | Tier 3 in detail — what it did and did not write |
| [Collectors and the plugin roadmap](collectors.md) | The senses: osquery, Scrapli, GitHub, the clouds |

## Source material

`Venapce/getting-started/README.md` · `venapce-api/README.md` · `venapce-wapp/` ·
`plugin-catalog/docs/venapce.md` and `docs/security-collectors-plan.md` ·
the blog post *Venapce: A Nervous System for Security Governance*.

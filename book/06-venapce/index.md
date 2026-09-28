# Part VI — Venapce

> **A product built on the product.**
>
> **Status: written against the source as of the `operations` release.** Venapce is in
> active development; where this part and `venapce-api` disagree, the repo wins.

If [FloMorphic](../05-flomorphic/) proves a product can be built on the runtime, Venapce
proves something one layer up: **a product can be built on a product built on the runtime**,
without ever becoming a runtime host itself.

Venapce implements **no** `IInflowService`, writes **no** compiler hook, and calls
`inflow.NewProcess` **nowhere**. It has no `inflow-fusion` dependency at all. Its business
logic lives in FloMorphic workflows, and it drives them over FloMorphic's ordinary REST API.

```
Senses — plugins & osquery agents
  Reach each system, return raw data frames. No opinions, no scoring, no stored secrets.
        ↓
Nervous system — FloMorphic workflows
  Correlate, enrich & evaluate per feature; an LLM node reasons in the open.
  ALL business logic lives here.
        ↓
The face — Venapce
  Dashboards, fleet view, stage, findings, issues, activities. Posture you can see;
  a panel you act from.
```

---

## The tiers, and where Venapce actually sits

This is the table that makes the platform's shape concrete, and it needs one correction
that the product's own evolution supplied.

| Tier | What you write | Example |
| --- | --- | --- |
| **1 — SDK** | a backend implementing `IInflowService`, a compiler hook, extrinsic handlers | FloMorphic, inflow-inspector |
| **2 — Plugin** | a process speaking `inflowv1` | Jira, Postgres, `llm`, osctrl |
| **3 — Workflows** | **no runtime host code** — logic as flows on an existing host | **Venapce** |

Venapce was first described as a pure tier-3 product, and for its *logic* that is still
exactly right. But shipping it turned up something more interesting than a clean tier:

> **Venapce is tier 3 for its logic and tier 2 for its reach — and its tier-2 plugin runs
> *inside its own backend*.**

`venapce-api` depends on `github.com/Inflowenger/go-plugin-sdk` and registers an `inflowv1`
plugin node in-process. That plugin is how a FloMorphic flow writes a row into Venapce's
database or asks Venapce's osquery fleet a question. There is no separate binary, and
**no settings profile** — the connections the plugin needs are already the backend's own.

That turns out to be the most reusable idea in this part, and it is not specific to
security:

> **If your product already has the database, the credentials and the connections, the
> cheapest way to expose it to a workflow engine is to be a plugin, in-process.**

Full detail: [Built under FloMorphic](built-on-flomorphic.md).

---

## What Venapce is

> A nervous system for security governance.
> *Vein — the agents across your compute. Synapse — the signal when something's wrong.*

Every security team is drowning in tools and starved of connective tissue. Endpoints,
network gear, cloud consoles, the ticketing system — each is a black box that already
decided what matters, and none of them talk. Venapce is the missing layer.

Four things carry the product:

| | |
| --- | --- |
| **A three-level pipeline** | `stage` → `findings` → `issues`, with the pipeline **optional** — a flow writes to whichever level its rules decide |
| **Activities** | Every row's timeline. A `run` activity is *one flow executed on one row*, with its conclusion lifted back into typed columns |
| **Operations** | A feature is an installable **package of flows** with a manifest, params and readiness checks |
| **A BI surface** | A full chart and dashboard builder over everything above, Superset behind the backend |

---

## Chapters

| Chapter | What it establishes |
| --- | --- |
| [What Venapce is](what-it-is.md) | The pipeline, activities, the assistant, and what ships |
| [Built under FloMorphic](built-on-flomorphic.md) | What it wrote and did not write — including the in-process plugin and the run bridge |
| [Operations: features as installable flow packages](operations.md) | The package manager, and the idea worth stealing |
| [Collectors and the plugin roadmap](collectors.md) | The senses: osquery, and the catalog beyond it |

## Source material

`venapce-api/internal/store/schema.sql` (**the source of truth for the model**) ·
`venapce-api/internal/plugin/README.md` · `venapce-api/internal/operations/` ·
`venapce-api/internal/flomorphic/` · `venapce-wapp/src/views/` ·
`venapce-wapp/public/schemas/venapce-operation.schema.json` ·
`Venapce/getting-started/README.md` · `plugin-catalog/docs/venapce.md` ·
the blog post *Venapce: A Nervous System for Security Governance*.

> **A note on stale docs.** `Venapce/getting-started/README.md` still describes the
> two-table *Stage → Issues* model that predates `findings`, `activities` and `operations`.
> `internal/store/schema.sql` says of itself that it is the source of truth, and this part
> follows it.

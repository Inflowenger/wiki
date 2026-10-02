# 7 · What you built

> **Fifteen minutes.** The inventory, why this shape is the same at any organization size,
> the shortcuts this session took, and what would falsify the claims it made.

## The inventory

Everything that now exists, and where it came from:

| Thing | Count | Where it lives |
| --- | --- | --- |
| Flows | 5 | `ingest-crm` · `ingest-tickets` · `ingest-assets` · `ingest-nightly` · `ingest-webhook` · `answer-request` |
| Triggers | 3 | One cron (21:00), two webhooks — **one per flow, never two** |
| Document stores | 3 | `ingest_state` · `ontology_concept` · `decisions_store` |
| Vector stores | 1 | `org_knowledge`, embedding config captured at creation |
| Plugins | 3 | `postgres` and `mysql` from the catalog; `arangodb` you wrote |
| Settings profiles | 4+ | One per database, one per LLM provider — credentials off the graph |
| Lines of application code | ~0 | Excluding the plugin. The flows are graphs, not code |

That last row is the claim the whole book is about, and it is now a thing you can check
rather than a thing you were told.

## What you did not have to build

Set against what you did:

| You built | You did not build |
| --- | --- |
| Five graphs | The durable execution engine that walks them |
| One plugin | The protocol it speaks, or the three SDKs that implement it |
| Four store definitions | The vector index, the embedding pipeline, the read-only SQL validator |
| Two trigger configurations | The cron scheduler, the webhook ingress, its five auth methods |
| One Rego policy | The policy evaluation node |
| One audit row schema | The park/resume machinery that made a human a node |
| Zero UI | The canvas, the live run animation, the generic plugin form builder |

The asymmetry is the point. A FloMorphic install is a canvas, a backend, fifteen compiler
hook cases, four extrinsic handler families and five plugins — on top of a runtime that was
not changed to accommodate any of it.

## Why this maps to any size of organization

The session used a 400-customer service company. The same four stages hold from a two-person
consultancy to a bank, and it is worth being explicit about which dials move and which do
not.

### What never changes

- **The stage order.** Identify → ingest → decide. Skipping identification is how you get an
  embedding bill and no answers.
- **One trigger per flow**, so an orchestrator plus sub-flows is the shape at every size.
- **A watermark per source.** Whether the source has 400 rows or 400 million.
- **The decider / reasoner split.** A closed-set question is a decider's job at any scale.
- **The audit row.** Smaller organizations need it *more*, because they have fewer people who
  remember why.

### What changes

| Dial | Two people | Four hundred people | Twenty thousand |
| --- | --- | --- | --- |
| Sources | 1–2, probably one SaaS | 4–8, mixed vintages | Dozens, with a data team who own some |
| Ingest cadence | Nightly is plenty | Nightly + webhook for the hot source | Per-source cadence; some streaming |
| Who signs | Nobody — the flow answers | A duty manager | A role, with delegation and an escalation ladder |
| Topology | One container on one box | One box, or a small compose stack | Several Fractal engines, spaces per tenant or per department |
| The ontology | Six concepts on a whiteboard | Six concepts in a store, reviewed quarterly | A governed vocabulary someone owns |
| Policy | A `js` Rule | An `opa` Rule | Rego in version control, with tests |

Notice what the right-hand column is *not*: a different architecture. It is the same graphs
with more engines behind them and more rigour in front of them. Scaling out is adding
Fractal engines and isolating spaces, not restructuring flows.
→ [Part VII — Scaling out](../07-architecture/scaling.md) ·
[Spaces and isolation](../07-architecture/spaces-and-isolation.md)

### The smallest honest version of this session

If four and a half hours is too long, the irreducible core is:

1. Install.
2. One source, one plugin from the catalog, one settings profile.
3. One ingest flow with a watermark and a vector store, on a cron.
4. One decision flow: search → decider → reply, with no human and no policy.

About ninety minutes, and it still demonstrates every load-bearing idea except the plugin
authoring and the approval gate.

## What this session left out

Named, so none of it surprises you later:

| Shortcut | What production needs |
| --- | --- |
| `AUTH_ENABLED=false` | Auth on, and `/mcp` never on a public interface — the tools include full write access and runtime control |
| A static-token webhook | HMAC from any system that can sign |
| Secrets in settings profiles on a single box | A real secret manager, and spaces scoped per tenant |
| No retry on the outbound HTTP node | A Rule on its result plus a `Continue After` to come back |
| No tombstone handling | Source deletes are invisible to a watermark; you need a periodic reconcile |
| No re-index strategy | Changing embedding model means a new store and a full re-index |
| No tests on the Rego | Policy is code. It gets tests |
| One SQLite file | Fine for this. Back it up by copying `flomorphic/data/`, and know that is your whole system's state |
| No cost ceiling on embeddings | 380k tickets is a budget conversation, not a scope decision |

## The claims this session made, and what would falsify each

The book's convention is that a chapter making a claim names what would break it. Three were
made here.

**Claim 1 — "Nothing is reserved for FloMorphic."**
The ArangoDB plugin you wrote in [chapter 4](04-build-a-plugin.md) imports the same SDK, speaks
the same protocol, holds a connection, streams progress, reads and writes context, and can
route its own ports — exactly as the builtin LLM and MCP nodes do.
*Falsified by:* finding a capability the builtin nodes have on the execution path that your
plugin cannot reach. The one real asymmetry is the **UI**: a builtin node gets a bespoke Vue
drawer, yours renders through the generic form builder. That is a fidelity ceiling, and it is
the only one. → [Builtin nodes are plugins](../05-flomorphic/builtin-nodes.md)

**Claim 2 — "The product's UI and an external agent are peers on one API."**
Every flow in this session was built, planned, run, inspected and amended over MCP, and every
one is editable on the canvas by hand.
*Falsified by:* finding an entity the canvas can change that no `flo_*` tool can, or a flow
built over MCP that the canvas cannot open. The honest boundary: *AI build* and the MCP
designer share one brain (`flo_get_design_guide` returns the same guidance the canvas dialog
uses), but **nothing in either path knows your store ids, your settings profiles or your
server URLs.** That gap is permanent and is why every prompt in this session ends with "ids
you need and I have".

**Claim 3 — "Business logic changes without a redeploy."**
[Chapter 6's three amendments](06-decide.md#the-three-amendments-you-will-actually-make) — a
cost ceiling, a threshold, a new branch — changed behaviour with no release.
*Falsified by:* an amendment that requires a code change. There are real ones, and they are
worth knowing: **a new node kind** is a catalog entry plus a compiler case (a frontend and
backend change), and **a capability no primitive expresses** becomes a plugin — a new process
to write and run. The seam is open; it is not free.
→ [The coverage argument § the awkward cases](../02-fusion/coverage.md#the-awkward-cases-stated-plainly)

## Running this as a session

If you are delivering it to a room:

**Pre-bake before anyone arrives.** The install (chapter 1) and the catalog plugins (chapter
3) are the two places a room of twenty people desynchronises. Have the stack running and the
Postgres plugin registered, and demonstrate those steps rather than having everyone do them.

**The checkpoints are the pacing.** Each chapter ends with one. Do not move on with a room
where *Run* is inert or a plugin never logged its subscriptions — those two failures poison
every later chapter.

**The five failures that will happen.** In order of likelihood:

| It happens | The fix, in one line |
| --- | --- |
| *Run* is inert | No runtime. `INFLOW_INFRA_API` unset, or Infra is down → [chapter 1](01-install.md#the-thing-that-trips-everyone-the-runtime-is-optional) |
| A plugin "is running" but no node appears | It never logged its subscriptions. `./plugin.sh logs` |
| A Go build dies on `403 Forbidden` | Network, not code. `export GOPROXY=https://goproxy.cn,direct` |
| A node ran twice and nobody knows why | Two inbound edges. Put a `Wait for All` in front of it |
| A branch silently ended | A Rule fired no tag. Make the returned value exhaustive over the handlers |

**Timings.** 10 · 25 · 35 · 45 · 60 · 60 · 15 minutes, plus two breaks. Chapter 4 is the one
to cut if you are short — it is self-contained, and chapters 5 and 6 work with an HTTP node
standing in for the plugin.

## Where to go next

| If you want to… | Read |
| --- | --- |
| Understand *why* fifteen nodes reduce to six primitives | [Part II — The compiler seam](../02-fusion/the-compiler-seam.md) and [The coverage argument](../02-fusion/coverage.md) |
| Build a product like FloMorphic, not a flow on it | [Part II — Build your own workflow product](../02-fusion/build-a-workflow-product.md) and [Part V — How it was built](../05-flomorphic/how-it-was-built.md) |
| Let *your* users define their own processes | [Part IV — Building a process product](../04-frontend/build-a-process-product.md) |
| Attach a system that has been in production for a decade | [Part VII — Attaching a system you already run](../07-architecture/attaching-legacy.md) |
| See a second product built on the product | [Part VI — Venapce](../06-venapce/) |
| Ship a set of flows as an installable package | [Part VI — Operations: features as installable flow packages](../06-venapce/operations.md) |

That last row is the natural sequel to this session: Northwind's five flows plus their stores
are a **portable workflow document** away from being installable at the next customer.
`GET /flow/id/:id/export` emits one, `POST /flow/import` takes it back — with `dryRun` to plan
without saving, and `missingActions` naming any plugin the destination install does not have.
That is how a build becomes a product.

## Source material

This chapter synthesises the preceding six. Its claims trace to
`FloMorphic/builtin-plugins/README.md` (claim 1), `flomorphic-api/mcpserver/` and
`designer/` (claim 2), and `flomorphic-api/inflow/compiler.go` (claim 3) ·
[Part II — The coverage argument](../02-fusion/coverage.md) ·
[Part VII — Architecture in Practice](../07-architecture/) ·
[Part V — How FloMorphic was built § portable workflows](../05-flomorphic/how-it-was-built.md).

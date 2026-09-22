# The coverage argument

> Six generic nodes. Every requirement a workflow product has ever shipped. This chapter
> does the mapping exhaustively, because the claim is only worth as much as its worst case.

[The primitive node reference](node-primitives.md) states the axis argument: computation,
decision, reach-your-own-system, reach-the-world — four axes, each with a primitive.

An axis argument is a *sketch* of completeness. This chapter is the audit. It walks every
capability a real workflow product ships and names which generic node covers it, and how.
Where the answer is awkward, it says so.

---

## How to read the tables

| Column | Meaning |
| --- | --- |
| **Requirement** | A feature a user or a competitor's product would name |
| **Generic node** | Which of the six covers it |
| **How** | The reduction — what the compiler hook or the author actually does |

A reduction marked **authored** happens at compile time (a hook builds a rule or a subject).
A reduction marked **drawn** happens on the canvas (edges, tags, `Depends`). A reduction
marked **process** means it is a plugin, deliberately.

---

## 1. Computation and data shaping

| Requirement | Generic node | How |
| --- | --- | --- |
| Transform a value | **Code** (js) | authored — the expression is the logic rule |
| Set / assign fields | **Code** (js) | authored — `input.x = …; input` |
| Filter a collection | **Code** (js) | authored |
| Aggregate / reduce | **Code** (js) | authored |
| Rename or reshape a payload | **Code**, or **Plugin** (`cast`) | authored, or a mapping form for non-programmers |
| Parse JSON / CSV / text | **Code** | authored |
| Evaluate a policy over data | **Code** (opa) | authored — Rego, scope as `input`, conditions as `data` |
| Template a string from context | any node's `{{$.path}}` variables | resolved by the engine before dispatch |

**The seam that matters here:** a product does not have to expose JavaScript. A *Set* card
with three key/value rows compiles to a Code node whose rule was assembled from those rows.
FloMorphic's **Cast / Mapping** node is exactly this idea taken one step further — a form
that maps each target key to a static value or a JSONPath.

## 2. Decision and control flow

| Requirement | Generic node | How |
| --- | --- | --- |
| If / else | **Contract** | authored — rule returns `then` or `else` tags |
| Switch / multi-way branch | **Contract** | drawn — N tagged handlers on one node |
| Comparison operators (`>`, `<`, contains, matches, between, is empty) | **Contract** | authored — one hook case each; the engine learns nothing |
| Guardrail / policy gate | **Contract** (opa) | authored — Rego over the scope |
| Loop / iterate until | **Contract** + a backward edge | drawn — a loop is an edge plus a condition |
| Parallel fan-out | tag emission with multiple tags | drawn — `["a","b"]` fires both, concurrently |
| Join / wait for all | **Void** + `Depends` | drawn — the barrier lists its inbound nodes |
| Early exit / stop this branch | **Extrinsic** returning `_cmd: stop`, or no tag match | authored |
| Sub-workflow / reuse | **GoTo** | drawn |
| Recursion | **GoTo** into the same flow | drawn |
| Dead end / terminator | **Void** | drawn |
| Start marker | **Void** | drawn |

**The one to internalise:** there is no branch *node type*. There is one mechanism — emit
tags, match transitions — and three primitives that can drive it.
See [Tag routing](tag-routing.md).

## 3. Reaching your own system

| Requirement | Generic node | How |
| --- | --- | --- |
| Call an internal service | **Extrinsic** | authored — one NATS request/reply subject |
| Write to your database | **Extrinsic** | authored — your handler owns the SQL |
| Read from your database | **Extrinsic** | authored |
| Many operations, one door | **Extrinsic** with a wildcard subject | `svc.persist.*`, handler reads `recv_subject` |
| A decision your backend makes | **Extrinsic** + `FilterNextResponse` | authored — reply carries tags |
| Enqueue work in your existing queue | **Extrinsic** | authored |

**Why this is cheap:** an Extrinsic handler is a few lines in a Go process you already run.
No new deployment, no new container, nothing moved. It is the lowest-cost way to make a
decade-old system reachable from a graph.

## 4. Reaching the outside world

| Requirement | Generic node | How |
| --- | --- | --- |
| HTTP / REST call | **Plugin** (`http`) | process |
| Database driver (Postgres, MySQL, Mongo, ClickHouse) | **Plugin** | process — the plugin holds the pool |
| Vector store | **Plugin** (Qdrant), or **Extrinsic** if your backend owns it | process / authored |
| SaaS API (Jira, Gmail, Telegram, GitHub) | **Plugin** | process |
| Network device / SSH | **Plugin** (Scrapli) | process |
| Message queue consumer | **Plugin** | process — a background loop in a live process |
| Webhook receiver | **Plugin**, or your own API + `NewProcess` | process |
| An LLM call | **Plugin** (`llm`) | process |
| An MCP client | **Plugin** (`mcp`) | process |
| Hardware / serial / proprietary protocol | **Plugin** | process |

**Why these must be a process, not a compiled node:** they hold connections, run background
loops, need their own release cadence, and often need their own configuration UI. A
compiled primitive can do none of that. This is precisely the gap Plugin exists to fill —
reaching for it is the design working, not failing.

## 5. Time, people and long waits

This is the group people assume needs engine support. It does not.

| Requirement | Generic node | How |
| --- | --- | --- |
| Wait N minutes / until a timestamp | **Extrinsic** that stops the branch, + a scheduled resume | authored — FloMorphic's *Continue After* |
| Wait for human approval | **Extrinsic** that records a task and stops, + resume on answer | authored — FloMorphic's *Human in the Loop* |
| Wait for an external event | **Extrinsic** that stops, + resume from a webhook | authored |
| Scheduled / cron trigger | your backend calls `NewProcess` on a tick | outside the graph, deliberately |
| Timeout on the whole run | `Settings.ExecuteTimeOut` | configuration |
| Timeout on one request | `Settings.RequestTimeOut` | configuration |
| Runaway protection | `Settings.ProcessNodeLimit` | configuration |

**The mechanism, in one line:** a node returns `{"_cmd":"stop"}`, the process ends, your
backend schedules whatever it likes, and later a new process starts at that node's
successors with `Resume: true`. The engine is **not running** during the wait — which is why
a three-day wait costs nothing.
See [the backend contract](the-backend-contract.md#resuming-a-run).

## 6. Errors, retries and idempotency

| Requirement | Generic node | How |
| --- | --- | --- |
| Branch on failure | **Contract** over the previous node's output | authored |
| An error port on a node | a tagged transition the node's error path selects | drawn |
| Retry | a backward edge + a **Contract** counting attempts in context | drawn + authored |
| Compensation / rollback | an ordinary downstream branch | drawn |
| Idempotency across a resume | `_registry.jobId` / `doneAt` from the previous run | in the plugin |
| Dead-letter | a branch ending in an **Extrinsic** that records it | drawn |

**Stated honestly:** retry is a *pattern* here, not a checkbox. A product that wants
"retries: 3" on every node card builds it once in the compiler — emitting the counter,
the backward edge and the Contract — and its users never see the assembly. That is the
compiler seam doing its job, but it is assembly you write once rather than a runtime
feature you inherit.

## 7. Observability and operations

| Requirement | Covered by | How |
| --- | --- | --- |
| Which nodes ran | `node.enter` / `node.exit` | the event stream |
| Which edge was taken | `edge.select.taken` | the event stream |
| **Why** that edge | `edge.select.reason` — the selecting tags | the event stream |
| Per-node duration | `node.exit.durationMs` | the event stream |
| Run status and error | `proc.finish` | the event stream |
| Live progress inside a long step | `job.Progress(…, Frame{…})` | the plugin |
| Free-form diagnostics | `log` events with `src` | the event stream |
| Inspecting run data | the context document | your storage |

See [Observing a run](observing-a-run.md).

---

## The awkward cases, stated plainly

A coverage argument that only lists wins is not an argument. Four places where the
reduction is real but uncomfortable:

**1. Map over a collection with per-item parallelism.** Tag fan-out gives you parallel
*branches*, not a parallel *map over N items* where N is known only at run time. The usual
reductions are a loop (sequential, correct, slower) or a plugin that does the fan-out
internally and commits an array. A first-class dynamic map would be the most defensible
addition to the primitive set, and its absence is a genuine constraint rather than a
non-issue.

**2. Retry as assembly, not a property.** Covered above. Correct, but it is three drawn
elements where a competitor offers a number field.

**3. Sub-second scheduling.** The stop/resume mechanism is designed for waits measured in
minutes to days. A flow that needs to resume in 200ms is fighting the design; keep that loop
inside a plugin.

**4. Exactly-once execution.** Not promised. A resumed run may re-enter a plugin action,
which is why `_registry` carries the previous `jobId`. Idempotency is the plugin author's
responsibility, and the protocol hands them the tools rather than the guarantee.

None of these falsify the claim — each has a working reduction. They are the honest cost of
a closed primitive set, and knowing them up front is worth more than a table of wins.

---

## What closure actually buys

The reason to hold the line at six, rather than adding a primitive whenever something is
awkward:

- **Two products share one engine.** FloMorphic's thirteen nodes and a hypothetical CI
  product's `job`/`step`/`matrix` both lower to the same six. Neither needs a runtime fork.
- **A plugin written once runs everywhere.** Because the protocol is the contract, not the
  product.
- **The engine can be hardened rather than extended.** A fixed execution surface is a fixed
  audit surface.
- **Adding fifty palette cards is a frontend release.** Not a runtime release, not a
  migration, not a version negotiation between engine and product.

> The claim is not that six types are elegant. It is that the four axes are exhaustive for
> this problem, each axis has a primitive, and the awkward cases have reductions you can
> read.

---

## Next

- [Tag routing](tag-routing.md) — the mechanism most of section 2 depends on.
- [Build your own workflow product](build-a-workflow-product.md) — using these tables to
  design a palette.

**Source material:** `inflow-fusion/docs/nodes.md`, `docs/nodes/from-frontend.md`,
`docs/routing.md`, `docs/infra.md`, `FloMorphic/getting-started/docs/nodes.md`.

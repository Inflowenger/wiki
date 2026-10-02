# 5 · Stage 2 — Ingestion

> **Sixty minutes.** Northwind's four sources are reachable. Now make the organization's
> knowledge *present*: a nightly sweep at 21:00, a webhook for what cannot wait, a vector
> store that is searchable, an ontology that makes it filterable, and a watermark that makes
> re-running safe.

## The shape, and the constraint that produces it

The obvious design is one big flow that reads every source. It is wrong, and the product
tells you so immediately:

> **A workflow can have only one trigger.** Try to add a second and the API answers
> `409`: *"a workflow can have only one trigger — clone the workflow to add another."*

That is not a limitation to work around. It is the constraint that forces the right shape:

```
                  ┌───────────────────────────┐
 cron 21:00 ──────▶  ingest-nightly            │   orchestrator · schedule trigger
                  │  (goto × 3, then join)     │
                  └──┬────────┬────────┬───────┘
                     │        │        │   goto
        ┌────────────▼──┐ ┌───▼──────┐ ┌▼────────────┐
        │ ingest-crm    │ │ ingest-  │ │ ingest-     │   sub-flows · NO trigger
        │ (postgres)    │ │ tickets  │ │ assets      │   reusable, testable alone
        └───────────────┘ └──────────┘ └─────────────┘
                     ▲
 POST /hooks/… ──────┘  via ingest-webhook (webhook trigger, routes to one sub-flow)
```

Four flows instead of one, and every one of the four is better for it:

- **Each sub-flow is independently runnable and testable.** You can run `ingest-crm` alone,
  against a crafted context, without waiting for 21:00 or touching the other sources.
- **The orchestrator is readable.** It is three `Goto` nodes and a join — the nightly policy,
  with none of the per-source detail.
- **A new source is a new sub-flow plus one `Goto`.** No edit to anything that already works.
- **Two entry points reuse the same work.** The nightly sweep and the webhook both call the
  same sub-flows, so there is exactly one definition of "ingest the CRM".

`Goto` is the composition primitive: it jumps into another flow like a subroutine and comes
back.

## The three stores, created first

Create these before any flow, because a flow references them by id and a model cannot
invent one.

### 1 · `ingest_state` — the watermark

A document store. One row per source, holding where the last sweep got to.

```text
flo_create_document_store
  name:  "ingest_state"
  table: "ingest_state"
  columns:
    - { name: "source",         type: "TEXT", primary: true }
    - { name: "last_record_id", type: "TEXT" }
    - { name: "last_seen_at",   type: "TEXT" }
    - { name: "rows",           type: "INTEGER" }
```

A document store, not a file and not a context document, for one reason: **a flow can
query it.** The read-side is real SQL — a single `SELECT` (or `WITH … SELECT`), with
writes, DDL, `PRAGMA`, comments and multiple statements rejected, and an unbounded query
capped at 1000 rows.

Seed one row per source so the first sweep has a floor to read:

```text
flo_write_document on ingest_state:
  { "source": "crm",      "last_record_id": "0", "last_seen_at": "1970-01-01T00:00:00Z", "rows": 0 }
  { "source": "tickets",  "last_record_id": "0", "last_seen_at": "1970-01-01T00:00:00Z", "rows": 0 }
  { "source": "assets",   "last_record_id": "0", "last_seen_at": "1970-01-01T00:00:00Z", "rows": 0 }
```

### 2 · `ontology_concept` — the vocabulary from chapter 3, as rows

```text
flo_create_document_store
  name:  "ontology_concept"
  table: "ontology_concept"
  columns:
    - { name: "concept",    type: "TEXT", primary: true }
    - { name: "definition", type: "TEXT" }
    - { name: "relations",  type: "TEXT" }
```

Six rows: `Customer`, `Contract`, `Entitlement`, `Asset`, `Service`, `Ticket`. The
`definition` is the sentence from the [chapter 3 sketch](03-identify.md#the-ontology-sketch);
`relations` is `has, grants, owns, depends-on, subject-of` as written there.

The point of putting it in a store rather than in a prompt: the tagging node *reads* it at
run time, so changing the vocabulary does not mean editing five flows.

### 3 · `org_knowledge` — the vector store

```text
flo_create_vector_store
  name:           "org_knowledge"
  provider:       "openai"              # or a full base URL of an OpenAI-compatible endpoint
  embeddingModel: "text-embedding-3-small"
  dimensions:     1536
  metric:         "cosine"
```

> **The embedding config is captured once, here, and reused for every index and search.
> It cannot change later.** Creating the store provisions a `sqlite-vec` index of exactly
> that width.
>
> This is a deliberate constraint and the most expensive thing in this chapter to get wrong.
> Changing embedding model later means a new store and a full re-index of everything. Decide
> the model now, with the cost of 380k tickets in mind — which is why
> [chapter 3](03-identify.md#the-inventory) made you write down the volume column.

## The rule that will bite you: scope cardinality

Before the first patch, the one concept that makes or breaks ingestion flows.

`scope` is a full JSONPath, and **its cardinality decides how many times a node runs**:

| scope | Runs |
| --- | --- |
| `$` | Once, over the whole context |
| `$.records[*]` | **Once per element** — each pass scoped to just that element |
| `$.records[?(@.kind == 'contract')]` | Once per matching element |

Those passes are a **queue inside the one node**. They run one after another, in order, on
the same node. Element 2 does not start until element 1 finishes. Nothing on the canvas
forks, no edge is drawn per element, and the node's outgoing edges are followed **once**,
after every pass is done.

That is how you iterate a collection. There is no loop node.

Inside a text field, `{{$this}}` is the element the current pass is scoped to, and
`{{$this.field}}` reaches into it. `{{$.records[0].text}}` would read the *first* record on
every pass — almost always a bug.

### The one limit

A many-scope node **must not decide where the flow goes.** A node has one set of edges for
the whole node; there is no per-element edge to carry a second answer. So the runtime stops
at the first element that picks a branch, logs a warning saying how many were skipped, and
your "iterate 500 records" node quietly becomes "run the first one, then decide".

This applies to every node whose ports are derived from its result:

- **Plugin-backed nodes** — `llm`, `jev`, `mcp`, `cast`, `http` and an imported `plugin`
  action are all the same Plugin primitive underneath. The visible signal is `functions`
  (LLM), `questions` with routed options (Jev), or `outbound` (a plugin action).
- **`rule` nodes**, whose `handlers` are the branches.

Give any of those a single-valued scope, usually `$`. An LLM node with **no** bound
functions is fine on a many-scope — it has one plain output, and running it per element is
exactly the point. That is the node the next flow uses.

## Flow A — `ingest-crm`, the sub-flow that does the work

Nine nodes, no trigger, entered by `Goto`. The whole pattern is here; the other two
sub-flows are this one with a different query.

### Build it

```text
Use the flomorphic MCP server.

1. Call flo_get_design_guide first and follow it exactly.
2. Build a workflow called "ingest-crm". It has NO trigger — it is entered by Goto
   from the nightly orchestrator.

   Steps, in order:
   a. Read the watermark: a Doc Store read on store ingest_state,
      SELECT last_record_id FROM ingest_state WHERE source = 'crm'
      → key "watermark", scope "$".
   b. Pull new rows: the POSTGRES plugin action postgres.query, with the SQL
      SELECT id, customer_id, title, body, updated_at FROM contracts
      WHERE id > '{{$.watermark.rows[0].last_record_id}}'
      ORDER BY id LIMIT 500
      → key "pulled", scope "$".
   c. Normalize into an array: a JS node at scope "$", key "records", producing
      one object per row with { id, text, customerId, updatedAt } where text is
      the title and body joined.
   d. Tag each record with ontology concepts: an LLM node at scope "$.records[*]",
      key "concepts", NO bound functions, prompt referencing {{$this.text}} and
      asking for the concepts from our ontology as a JSON array.
   e. Index each record: a Vector Store WRITE on store org_knowledge, scope
      "$.records[*]", input "$this".
   f. Compute the new high-water mark: a JS node at scope "$", key "nextWatermark",
      producing { source: 'crm', last_record_id, last_seen_at, rows }.
   g. Save it: a Doc Store WRITE on store ingest_state, input "$.nextWatermark".

   Constraints you cannot know — use these ids verbatim:
   - ingest_state store id: <mem_…>
   - org_knowledge store id: <mem_…>
   - POSTGRES pluginId: <…>, settings profile: northwind-crm (<nset_…>)

3. Call flo_plan_patch, NOT apply, and show me the `problems` list.
```

### What comes back

The patch, abridged to the parts that teach something:

```json
{
  "nodes": [
    { "ref": "start", "kind": "startNode", "title": "Start", "scope": "$" },

    { "ref": "wm", "kind": "docstore", "title": "Read watermark",
      "key": "watermark", "scope": "$",
      "data": { "storeId": "mem_…", "action": "read",
                "query": "SELECT last_record_id FROM ingest_state WHERE source = 'crm'" },
      "note": "the floor for this sweep" },

    { "ref": "pull", "kind": "plugin", "title": "Pull new contracts",
      "key": "pulled", "scope": "$",
      "data": { "pluginId": "…", "action": "postgres.query", "settingsId": "nset_…",
                "body": { "sql": "SELECT id, customer_id, title, body, updated_at FROM contracts WHERE id > '{{$.watermark.rows[0].last_record_id}}' ORDER BY id LIMIT 500" } },
      "note": "the plugin resolves {{$.…}} against the live context" },

    { "ref": "norm", "kind": "js", "title": "Normalize", "key": "records", "scope": "$",
      "data": { "lang": "js",
                "logic_rule": "let rows = input.pulled.rows || []\nlet records = rows.map(r => ({ id: String(r.id), customerId: r.customer_id, updatedAt: r.updated_at, text: (r.title || '') + '\\n\\n' + (r.body || '') }))\nrecords" } },

    { "ref": "tag", "kind": "llm", "title": "Tag with ontology concepts",
      "key": "concepts", "scope": "$.records[*]",
      "data": { "subject_prefix": "llm", "request": "run",
                "body": { "messages": [], "clear_history": false }, "functions": [] },
      "note": "no bound functions, so a many-scope is correct here" },

    { "ref": "index", "kind": "vecstore", "title": "Index record",
      "scope": "$.records[*]",
      "data": { "storeId": "mem_…", "action": "write", "input": "$this" },
      "note": "one pass per record; $this is this pass's record" },

    { "ref": "high", "kind": "js", "title": "New high-water mark",
      "key": "nextWatermark", "scope": "$",
      "data": { "lang": "js",
                "logic_rule": "let records = input.records || []\nlet prev = input.watermark.rows[0].last_record_id\nlet maxId = records.reduce((m, r) => (r.id > m ? r.id : m), prev)\nlet nextWatermark = { source: 'crm', last_record_id: maxId, last_seen_at: new Date().toISOString(), rows: records.length }\nnextWatermark" } },

    { "ref": "save", "kind": "docstore", "title": "Save watermark", "scope": "$",
      "data": { "storeId": "mem_…", "action": "write", "input": "$.nextWatermark" } }
  ],

  "edges": [
    { "from": "start", "to": "wm" },
    { "from": "wm",    "to": "pull" },
    { "from": "pull",  "to": "norm" },
    { "from": "norm",  "to": "tag" },
    { "from": "tag",   "to": "index" },
    { "from": "index", "to": "high" },
    { "from": "high",  "to": "save" }
  ],

  "notes": [
    "The Vector Store node's partition and metadata bindings are set in the drawer — set partition to 'crm' and carry concept tags as metadata.",
    "The LLM node needs a provider settings profile bound in its drawer."
  ]
}
```

### Four things to read in that patch

**The whole flow is a straight line.** No branches. Nothing here decides anything, so
nothing needs a port. Resist the urge to add error branches to an ingestion flow — a node
that fails is reported and committed, and the run carries on, which is usually what you
want at 21:00. [Flow B](#flow-b--ingest-nightly-the-orchestrator) is where failure is
noticed, once, for all three sources.

**`tag` and `index` are both many-scope, and both are legal** — neither has derived ports.
They are separate nodes running separate queues, in sequence, because `index` needs what
`tag` wrote.

**`clear_history` is `false` and that is correct here.** An LLM node's conversation is held
on the node's *scope*, and each pass of a many-scope stands on a different element — so
history does not leak between records. Inside a **loop** it is the opposite and
`clear_history: true` is mandatory, because every pass is the same scope. That asymmetry is
the quiet killer in this product; it is worth saying out loud once.

**The JS nodes end on a named variable.** In a `js` node the value of the **last
expression** is the output. There is no `return`, no `ctx`, no function wrapper — the
scoped slice arrives as `input`. Build the result in a named variable and put that variable
alone on the last line. Never write `{{$.x}}` inside code; template tokens are substituted
into *text* fields only, and in code they are either literal characters or a syntax error.

To reach data *outside* the node's scope there is `_get("$.path")`, and `_log("…")` writes
a line into the run log. Prefer plain property access on `input` whenever the value is in
scope.

## Flow B — `ingest-nightly`, the orchestrator

```text
Build a workflow "ingest-nightly".

- Start fans out to THREE Goto nodes in parallel — one per sub-flow:
  ingest-crm, ingest-tickets, ingest-assets.
- All three join into a Wait for All.
- Then a JS node at scope "$" producing a summary { perSource, totalRows, failures }.
- Then a Rule node at scope "$" with handlers "ok" and "failed", returning 'failed'
  when any source reported zero rows AND an error, 'ok' otherwise.
- The "failed" port goes to a Human in the Loop node that reports which source
  failed and asks whether to retry tonight or wait for tomorrow.
- The "ok" port ends the flow.

Then arm it: flo_set_schedule_trigger on this flow, mode cron, cron "0 21 * * *",
timezone "Europe/London", contextMode new, title "Nightly ingest".

flo_plan_patch first.
```

Three things this flow exists to teach.

### The join is not optional

**Edges into a node do not merge.** The runtime starts one task per edge it follows, so a
node with three inbound edges **runs three times**, in parallel. Three summaries, three
rule evaluations, three human tasks. Nothing about it looks unusual on the canvas, which is
why it is the easiest way to get a workflow subtly wrong.

`Wait for All` (`promissall`) is the **only** thing that merges branches. It waits until
every inbound branch has finished, then continues **once**, with the context all of them
wrote.

Two rules when you use one:

- It waits for exactly the nodes wired into it — so wire every branch it must wait for
  **directly** into it.
- Every branch feeding it must run the **same number of times**. It waits for its inbound
  nodes to reach the same round, so a branch that is itself doubly-run waits for a round
  that never comes, and the run stalls until the process times out.

### Parallel because of data dependency, not because of order

The three sub-flows do not read each other's output, so they fan out. An edge means "B
needs what A produced" — chaining independent steps because the goal listed them in that
order serialises work for nothing. Northwind's three sources are genuinely independent, so
three edges from Start.

### The schedule trigger

| Field | Value | Why |
| --- | --- | --- |
| `mode` | `cron` | `interval` exists too — seconds between fires — but "21:00" is a wall-clock statement |
| `cron` | `0 21 * * *` | — |
| `timezone` | `Europe/London` | **Set it.** Without it you are at the mercy of the container's clock, and the sweep moves an hour twice a year |
| `contextMode` | `new` | A fresh context document per fire — isolated runs, at the cost of a row per night. `existing` reuses one doc and mutates it in place |

`flo_get_trigger` returns the trigger with its **next fire time** computed, which is the
fastest way to confirm the cron string means what you think.

## Flow C — `ingest-webhook`, for what cannot wait

Tickets change continuously. Waiting until 21:00 to know about a severity-1 incident is not
a design, it is a latency.

```text
Build a workflow "ingest-webhook".

- Start → a JS node at scope "$" that reads the delivered payload and produces
  { source, recordId } — validating that source is one of crm|tickets|assets.
- → a Rule node at scope "$" with handlers "crm", "tickets", "assets", "unknown",
  returning the source name (or 'unknown').
- Each of the first three ports goes to a Goto into the matching sub-flow.
- "unknown" goes to a JS node that logs the rejection with _log and ends.

Then arm it: flo_set_webhook_trigger on this flow, methods ["POST"],
auth { method: "static", secret: "<token>", headerKey: "X-Northwind-Token" },
contextMode new, title "Ad-hoc ingest".
```

The webhook's payload lands in the run's context, which is why the first node can just read
it. The public URL is `/hooks/<slug>`; the slug is minted if you do not give one and kept
stable across updates.

> **Auth is not optional on an ingress.** The methods are `none`, `static`, `basic`, `jwt`
> and `hmac` — and when the method is `none`, **an IP allow-list is required**, which is the
> API refusing to let you open an unauthenticated public endpoint by accident. `static` with
> a header token is the floor; `hmac` is what you want from a system that can sign.

### A Rule node that fires nothing ends the branch silently

The `unknown` handler is not defensive padding. **A Rule node that fires no tag prunes every
one of its outgoing edges** — that branch of the flow simply ends, with no error and no log
line.

So a Rule's `logic_rule` must return a handler **name** (or an array of names) on every
path:

```js
let src = (input.source || '').toLowerCase()
let decision = ['crm','tickets','assets'].includes(src) ? src : 'unknown'
decision
```

Write it as an exhaustive expression, not as `let decision; if (ok) { decision = 'crm' }`.
The second form leaves `decision` undefined in the else case and produces a dead port that
looks fine on the canvas and fails invisibly at run time.

## Proving it works

This is the part the session should not skip, because "the flow is green" is not the same
as "the data is in there".

### 1 · The watermark moved, and only once

```text
Before: flo_query_documents on ingest_state —
        SELECT source, last_record_id, rows FROM ingest_state

Run:    flo_start_process on ingest-crm with a fresh context, then
        flo_get_process until it leaves `running`

After:  the same SELECT
```

`last_record_id` should have advanced and `rows` should equal what the source actually had
new. Now **run it again immediately**:

```text
Run again. Expect: rows = 0, last_record_id unchanged.
```

That is the whole idempotence story. The watermark is doing its job: the second sweep's
`WHERE id > '<last>'` matches nothing.

> **The honest caveat.** The watermark prevents re-*reading*. It does not prevent
> re-*indexing* — the vector store does not deduplicate for you. If a record is pulled
> twice (a replayed webhook, a restored backup, a manual re-run after resetting the
> watermark) you get two vectors with the same content. Carry a stable `recordId` in each
> vector's metadata and reconcile on it, or accept the duplicate and let `topK` absorb it.
> Know which one you chose.

### 2 · The content is findable

```text
flo_search_vectors on org_knowledge:
  text: "emergency restore outside business hours"
  topK: 5
```

You get content, metadata, distance, and a normalized similarity score where higher is
nearer. If the matches are nonsense, the problem is almost never the search — it is what
`norm` put in `text`. Go and look at a record in the Contexts browser.

### 3 · The ontology actually attached

```text
flo_get_context on the run's context — show me records[0].concepts
```

If the concepts are free text rather than terms from the six, the tagging prompt is not
constraining the model. That is a prompt fix, and it is also the argument for
[chapter 6's decider node](06-decide.md#why-jev-and-not-an-llm-here): when the answer must
come from a closed set, asking a model to be disciplined is weaker than asking a node that
cannot do otherwise.

## Debugging, when it goes wrong

```text
Workflow ingest-crm, run <indexId>, did not index anything.

- flo_get_process for status, error and timings.
- flo_get_context: show me watermark, pulled, and records — in that order.
- Tell me which of the three is the first one that is empty or wrong.

Do not propose a fix until you have named that node.
```

"The first one that is empty" is the whole method, and the ordered list of the usual
answers:

| First empty | Cause |
| --- | --- |
| `watermark` | Wrong store id, or the seed row for `crm` was never written |
| `pulled` | No settings profile bound, the plugin is not running, or the SQL matched nothing (check the watermark value it interpolated) |
| `records` | The JS node ended on something other than a named variable, or `input.pulled.rows` is not the shape the plugin returned |
| `concepts` | No provider profile on the LLM node |
| nothing empty, but no vectors | The Vector Store node is on `action: "read"`, or its `input` is not `$this` |

## Amendments you will actually make

Three, with the prompts, because these are the real ones:

**The sweep is too slow — 500 rows at a time is not enough.**

```text
Amend ingest-crm: raise the LIMIT to 2000, and add a Continue After node that
parks the run for 2 hours and re-enters the pull node when `records.length`
equals the LIMIT — so a backlog drains over several passes instead of one.
Read the flow with flo_get_workflow first, keep every node id, plan before apply.
```

**A fourth source arrives.**

```text
Build "ingest-parts" as a sub-flow in the same shape as ingest-crm, but using
the HTTP node against the vendor parts API instead of a plugin. Then amend
ingest-nightly to add a fourth Goto into it, wired into the SAME Wait for All.
```

Note what is *not* in that second prompt: any change to the three existing sub-flows. That
is the shape paying for itself.

**The ontology changed.**

Nothing to amend. Add a row to `ontology_concept` and the tagging node reads it on the next
run. That is why it is a store and not a prompt.

## What this stage costs, stated plainly

- **Embeddings cost money and time.** Every record indexed is an API call. 380k tickets is
  not a first pass — scope it to the last 24 months, or to tickets with a resolution, and
  say so in the inventory.
- **The 1000-row cap** on an unbounded document-store query is real. Paginate.
- **Deletes are invisible to a watermark.** `WHERE id > last` never sees a row that was
  removed. If the source deletes, you need tombstones or a periodic full reconcile — and
  this session does neither.
- **Changing the embedding model means a new store.** The config is captured at creation.
- **At-least-once, not exactly-once.** A run that crashes after indexing but before saving
  the watermark will re-index on the next sweep. The durable engine guarantees the run
  resumes, not that your side effects were transactional.

## Checkpoint

Three stores, four flows, one armed cron and one armed webhook. A `flo_search_vectors` call
that returns something true about Northwind. Stage 2 is complete — the brain now has a
memory, and it refreshes itself.

## Source material

`flomorphic-api/designer/assets/preamble.md` (scope cardinality, the many-scope limit,
wiring, joining, writing code — this chapter is that guide applied) and `assets/catalog.json`
(the node kinds and their data fields) · `flomorphic-api/mcpserver/tools_memory.go`,
`tools_trigger.go`, `tools_process.go` · `flomorphic-api/api/trigger/` (the one-trigger rule,
the `/hooks/:slug` ingress, auth methods) · `flomorphic-api/inflow/port.go` (the
`svc.store.doc.*` and `svc.store.vec.*` contracts) ·
`FloMorphic/getting-started/docs/nodes.md` ·
[Part II — Waiting: joins and delays](../02-fusion/waiting.md),
[Tag routing](../02-fusion/tag-routing.md),
[Observing a run](../02-fusion/observing-a-run.md).

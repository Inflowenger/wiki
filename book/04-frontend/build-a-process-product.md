# Building a process product on any frontend

> **The feature:** your users define their own processes, inside your product, in your
> vocabulary.
>
> **The claim:** that feature is a frontend you write, two npm packages, and one backend
> that answers three questions. Everything else in this book is already built.

[Build your own workflow product](../02-fusion/build-a-workflow-product.md) makes this case
from the server side. This chapter makes it from the browser, because that is where the
feature is *seen*, and because the frontend is where teams reliably overestimate what they
have to write.

---

## The problem class

There is a category of requirement that appears in almost every serious enterprise or
data-driven system, and it is nearly always solved badly:

> *"Each account needs to define its own process."*

It arrives with different nouns every time:

| The domain | What the user wants to define |
| --- | --- |
| **Procurement / finance** | An approval chain: who signs off, at what threshold, in what order, with what escalation after 48 hours |
| **HR** | Onboarding and offboarding: which accounts get created, which trainings are assigned, which manager confirms |
| **Data platform** | An ingestion pipeline: pull, validate, transform, land, reconcile, alert on drift |
| **Security / GRC** | Evidence collection and review: collect from N systems, evaluate policy, route exceptions to an owner, re-check in 90 days |
| **Support / ITSM** | Ticket routing and escalation: classify, enrich from the CRM, assign, wait for a human, reopen on SLA breach |
| **Insurance / lending** | Adjudication: score, check fraud rules, request documents, decide, notify, archive the reasoning |
| **Manufacturing / logistics** | A work order: steps, gates, quality checks, exceptions |
| **Marketing / CRM** | A journey: on this event, wait, branch on behaviour, act, measure |

Every one of these is the same shape. **A graph with branches, waits, retries, human steps
and external calls, defined per tenant, changed without a deploy, and auditable
afterwards.**

### How it usually goes

**Option one: hardcode it.** Ship the process as code, and take a change request every time
a customer's approval threshold moves. The flow lives in service boundaries, scattered
across handlers and queue consumers, and nobody can answer "what actually happened on
request 4471?" without reading logs from six services.

**Option two: a configuration table.** A `workflow_steps` table, a status column, a state
machine. It works until the first branch, then the first parallel path, then the first
"wait for a human for three days", then the first loop. Each of those is a rewrite, and the
last one is a distributed-systems problem you did not plan to own.

**Option three: bolt on n8n or Zapier.** Now the process lives outside your product, in
someone else's UI, with your data crossing a boundary, your branding gone, and your
per-tenant isolation a matter of trust.

The fourth option is what this book is about: **the process definition becomes a
first-class feature of your product**, and the durable engine underneath it is something
you bind to rather than build.

---

## What you write, and what you get

This is the whole cost model for the frontend.

| Layer | Who writes it |
| --- | --- |
| Authoring surface — canvas, wizard, DSL editor | **You.** The thing you differentiate on |
| Your palette, your node names, your vocabulary | **You.** The product |
| Node configuration forms for **your own** nodes | **You** — ordinary forms |
| Node configuration forms for **integrations** | `@inflowenger/plugin-form-builder` — the plugin declares them |
| Launching a run | **You** — one `POST` to your backend |
| Run state: which node, which edge, how far, did it finish | `@inflowenger/flow-trace` |
| A runtime log view with a real taxonomy | `@inflowenger/flow-trace` — see [The log taxonomy](log-categories.md) |
| Durability, retries, loops, joins, multi-day waits | **The engine.** Not a feature you build |
| Multi-tenant isolation | **Infra.** Spaces, as NATS accounts |
| The integration catalogue | **The plugin ecosystem.** Your list starts full |

The two library rows are the ones people do not believe until they see them. Integration
forms and run observation are, between them, most of the frontend work in a workflow
product — and both arrive on npm.

---

## The wiring, end to end

Four things happen between a browser and a running process. Here is each one, with what it
costs.

```
  ┌────────────────────────── your frontend ──────────────────────────┐
  │                                                                   │
  │  1. author        2. configure       3. launch      4. watch      │
  │   your canvas   plugin-form-builder    POST        flow-trace     │
  └──────┬────────────────┬──────────────────┬──────────────┬─────────┘
         │ save verbatim  │ pluginFn         │              │ WebSocket
  ┌──────▼────────────────▼──────────────────▼──────────────┴─────────┐
  │                 your backend  —  inflow-fusion                    │
  │  RetrieveFlow · RetrieveContext · UpdateContext · svc.* handlers  │
  └──────┬────────────────┬──────────────────┬──────────────┬─────────┘
         │ store          │ NATS             │ Exec()       │ subscribe
  ┌──────▼──────┐  ┌──────▼──────┐   ┌───────▼──────┐  ┌────┴─────────┐
  │ your DB     │  │ plugin proc │   │   Fractal    │  │ event log    │
  └─────────────┘  └─────────────┘   └──────────────┘  └──────────────┘
                                        Infra: credentials · spaces · registry
```

### 1. Author — and persist verbatim

Your canvas saves **its own document**, unchanged. A Vue Flow graph saves a Vue Flow graph,
positions and all. A React Flow graph saves a React Flow graph. A YAML editor saves the
YAML text.

```ts
await http.post(`/workflow/${id}/graph`, { nodes, edges, viewport })
```

**The frontend does no translation.** This is the single most valuable convention in the
platform: the compiler runs on the backend, on demand, per run — so you can change how a
node lowers without a data migration, and the editor stays lossless. See
[the compiler seam](../02-fusion/the-compiler-seam.md).

### 2. Configure — for nodes you have never seen

Your own nodes get your own forms; nothing clever is needed there. Integration nodes get
theirs from the plugin:

```vue
<InflowForm
  :schema="form.schema" :uischema="form.uischema"
  v-model:data="node.data"
  :renderers="vanillaRenderers"
  :plugin-id="node.pluginId" :settings="node.settingsProfile"
/>
```

with one transport registered once, app-wide:

```ts
app.use(InflowUiPlugin, createInflowUi({
  call: ({ pluginId, fn, body }) => api.pluginFn(pluginId, fn, body),
  theme: { /* four class names */ },
  onNotify,
}))
```

`api.pluginFn` is an ordinary `POST` to your backend, which forwards it to the plugin over
`inflowv1`. That is the entire integration surface for **every plugin in the catalogue,
including ones published after you ship**. See
[Dynamic forms](plugin-form-builder.md).

### 3. Launch — one POST

The browser does not hold credentials, does not know the engine's address, and never opens
a NATS connection. It calls your API:

```ts
processesApi.start({ flowId, startNodeId, contextId })
// → POST /process
```

Your backend — which already imported the SDK at boot — does the rest:

```go
// at boot: this is the connection to Infra, and the only one there is
inflow.InitBackend(ctx, &MyService{db: db},
    inflow.WithInfraApi(os.Getenv("INFLOW_INFRA_API")))

// per run
proc, err := inflow.NewProcess(startNodeId,
    inflow.WithFlowId(flowId),
    inflow.WithContextDocument(contextId),
).Exec(ctx)
```

`InitBackend` is where "connect to Infra" happens: it authenticates with the API secret,
obtains scoped NATS credentials, registers your service's subjects, and discovers the live
engine pool. `Exec` picks an engine from that pool and hands it a `ProcessRequest`.

**There is no second integration.** No SDK per engine instance, no connection string in
your frontend, no queue to provision. Two calls, one at boot and one per run. See
[the backend contract](../02-fusion/the-backend-contract.md).

> **A second Fractal instance needs no code change.** It registers with Infra, the pool
> reloads, and new runs round-robin across both. Your `Exec` call is identical.

### 4. Watch — relay the stream, feed the tracker

Your backend subscribes to the registration's event-log subject and relays every event
**verbatim** to the browser over whatever socket you already have. FloMorphic uses
Socket.IO over Fiber (`flomorphic-api/api/wslog`); an SSE endpoint or a raw WebSocket works
identically — the tracker does not care what delivered the bytes.

On the frontend, one tracker owns the stream:

```ts
const tracker = createFlowTracker()          // no pid filter: this socket carries all runs

socket.on('message', (raw) => tracker.ingest(raw))

tracker.on('node:enter', ({ pid, node }) => runs[pid].mark(node.key, 'running'))
tracker.on('move',       ({ pid, edgeId }) => runs[pid].light(edgeId))
tracker.on('finish',     ({ pid, status }) => runs[pid].settle(status))
tracker.on('log',        (e) => drawer.push(e))
```

That is the whole observability layer. Demultiplexing by `pid`, ordering by `seq`,
validating the envelope and detecting gaps are the library's problem. See
[Watching a run](flow-trace.md).

---

## What each frontend gets

The two packages are not equally portable, and it is worth being exact about that rather
than cheerful.

| Your stack | `flow-trace` | `plugin-form-builder` |
| --- | --- | --- |
| **Vue 3** | ✅ | ✅ everything in this part |
| **React / Svelte / Angular / Solid** | ✅ — it is plain TypeScript with an event emitter and plain-object state | ❌ Vue-only. You render the plugin's JSON Schema + UI schema with your own JSON Forms binding (`@jsonforms/react` exists) and implement `pluginFn` yourself — the [contract](plugin-form-builder.md#pluginfn-the-platforms-one-action) is a page long and fully specified |
| **A mobile app** | ✅ | ❌ — same as above |
| **A CLI, a script, a test** | ✅ | n/a |
| **No UI at all** | ✅ — `replay(lines).get(pid)` turns a stored run into final state | n/a |

`flow-trace` was built dependency-free on purpose, and this table is that purpose. The
package that costs a non-Vue app something is the form renderer — and what it costs is
*re-implementing a documented contract against a library that already has bindings for your
framework*, not inventing anything.

> **The authoring canvas is not in this table**, because it was never in the packages. Vue
> Flow and React Flow are deliberately mirrored data models — `nodes`/`edges` with
> `id`/`type`/`position`/`data` and `id`/`source`/`target`/`sourceHandle`/`targetHandle` —
> and the shipped compiler decodes either. See
> [Compiling a canvas](../02-fusion/compiling-a-canvas.md).

---

## Per-tenant processes

The requirement that started this chapter was *"each account defines its own process"*, and
it has an architectural answer rather than a `WHERE` clause.

**In your data:** a flow is a row. One per tenant, or one per tenant per process type.
Because you own the storage and the compiler hook, "tenant A's approval flow" and "tenant
B's approval flow" are two documents that lower through the same code.

**In the platform:** Infra models tenants as **spaces** — NATS accounts with credentials
scoped so that one tenant's plugins cannot observe another's traffic. That is isolation at
the message bus, not at the application layer, and it is the difference between a
multi-tenant product and a shared database with a filter. See
[Spaces and isolation](../07-architecture/spaces-and-isolation.md).

**In the UI:** the event stream carries **every process on that engine**, so the browser
must filter. `createFlowTracker({ pid })` accepts one pid or an array; a tenant-facing view
filters to that tenant's runs, and a backend that relays the socket should not be sending
another tenant's events in the first place.

---

## FloMorphic as the worked example

The strongest argument for any of this is that a real product is built exactly this way,
with its source open. FloMorphic's frontend is a Vue 3 + Vite + TypeScript + Vue Flow +
Tailwind + Pinia app that imports **these two packages at their published versions** and
writes its own business logic.

Read it as a guide. The files that matter, and what each one shows:

| File | What to take from it |
| --- | --- |
| `src/main.ts` | `createInflowUi` with the four theme classes, the `call` transport (one line), and an `onNotify` policy that claims only interrupting messages |
| `src/stores/flowLogs.ts` | One socket, one tracker, three derived views: log lines, per-pid lifecycle, per-pid canvas state. Also the display-buffer cap |
| `src/lib/runState.ts` | Folding tracker events into *the canvas's* node/edge model — the adapter between library state and product state |
| `src/stores/flowGraphs.ts` | A Pinia store that simply **is** a `GraphIndex`, handed straight to `resolveRefs` with no adapter |
| `src/components/flow/FlowRefText.vue` | `parseRefs` + `resolveRefs` rendering a log line with ids swapped for titles |
| `src/lib/pluginForm.ts` | The defensive boundary: a third party's form document parsed without ever throwing |
| `src/components/plugin/PluginForm.vue` | `<InflowForm>` with both slots overridden, and a comment explaining why the wrapper is load-bearing |
| `src/api/processes.ts` | Launching and stopping a run — an ordinary REST client, no platform concepts |

Note what is **not** in that list: nothing about NATS, nothing about credentials, nothing
about the engine's protocol. The whole platform reaches the browser as two imports and a
socket.

> **`runState.ts` is the file worth studying twice.** It exists *next to* the tracker's own
> state rather than replacing it, because a canvas needs a different shape than a stream
> consumer does. That is the correct relationship to have with these libraries: take the
> ordered, validated, deduplicated stream, and fold it into whatever your product actually
> draws.

---

## Compared to bolting on n8n

The honest comparison, since that is the alternative most teams weigh.

| | **A workflow tool beside your product** | **Processes inside your product** |
| --- | --- | --- |
| Where the user defines a process | Another app, another login, another UI | Your app, your vocabulary, your design system |
| Whose data | Crosses a boundary | Yours. The engine never connects to your database |
| Branding and pricing | Theirs | Yours |
| Per-tenant isolation | Your policy, enforced by convention | NATS accounts, enforced by credentials |
| Integration catalogue | Theirs — large | The `inflowv1` catalogue, plus anything you write in Go, Node or Python |
| Custom node | Their plugin model, their language, their release cycle | A process you deploy on your own cadence |
| Run history and audit | In their system | In your database, with the executed path and the reason for each branch |
| The graph's relationship to your domain | Calls your API from outside | *Is* your domain logic — `svc.*` handlers are your code in your process |

The row that decides it for most products is the last one. A workflow tool beside your
product can only call your API. A workflow layer **inside** it can invoke your domain
logic directly as a step, which means the graph is an expression of your business rather
than an orchestrator poking at it through a keyhole.

---

## Checklist

Backend (from [Part II](../02-fusion/build-a-workflow-product.md)):

- [ ] Platform installed; API Secret Key recorded
- [ ] `IInflowService` implemented over your storage; `ContextDoc.Header` stored **verbatim**
- [ ] Compiler hook written, using `nodes.*` builders
- [ ] Domain actions registered via `ImplHandlerOnSubject`
- [ ] Event-log subject **resolved from the registration**, not the fallback constant
- [ ] Events relayed to the browser **verbatim** — do not reshape them in the relay

Frontend:

- [ ] Authoring surface chosen; **the author's document persisted unchanged**
- [ ] `@inflowenger/flow-trace` owns the socket; one tracker, not one per component
- [ ] Node state keyed on **`flow:node`** via `nodeKey()` — never a bare id
- [ ] Ordering left to the tracker; **never sort by `ts`**
- [ ] Runs filtered by `pid` — per tenant, and per view
- [ ] Display buffer capped; a looping flow emits without bound
- [ ] `GraphIndex` implemented over the flow store you already have, for id → title
- [ ] Log view driven by `category` + `level` + `src`, not by string matching
- [ ] Failure read from `proc.finish.status`, **not** from the presence of an error log
- [ ] `createInflowUi` registered once: `call`, `theme`, `onNotify`
- [ ] A plugin's form document parsed defensively — a malformed form degrades to "no form"
- [ ] `<InflowForm>` used rather than bare `<JsonForms>` wherever an action may re-render the schema

---

## Next

- **[Part V — FloMorphic](../05-flomorphic/)** — the product this chapter keeps pointing
  at, described on its own terms.
- **[How FloMorphic was built](../05-flomorphic/how-it-was-built.md)** — the same story
  from the backend side.

**Source material:** `flomorphic-wapp/src` (`main.ts`, `stores/flowLogs.ts`,
`stores/flowGraphs.ts`, `lib/runState.ts`, `lib/pluginForm.ts`,
`components/plugin/PluginForm.vue`, `api/processes.ts`); `flomorphic-api/api/wslog`;
`inflow-fusion/README.md`; `inflow-js/README.md`.

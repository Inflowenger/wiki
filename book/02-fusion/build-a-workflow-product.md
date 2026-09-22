# Build your own workflow product

> Everything in Part II, assembled. What you write, what you get, and in what order.

FloMorphic is one product on this runtime. Venapce is a product on *that*. This chapter is
the map for building a third — an n8n-shaped automation tool, a CI runner, an ERP whose
processes are graphs, a domain-specific orchestrator.

---

## The four actors

Your backend sits inside a cast of four. Know the cast before writing anything.

1. **Infra — the control plane.** The NATS message substrate plus account, space and
   resource management. Hands out scoped credentials, isolates traffic into spaces, tracks
   which engine instances are alive. You talk to it over plain REST with a shared-secret
   bearer JWT.
2. **Your backend — the SDK.** Owns the product data. **Does not execute flows.** Answers
   three questions over NATS, and optionally exposes domain actions.
3. **Fractal — the engine.** Give it a `ProcessRequest` and it walks the compiled graph,
   asking your backend for anything it lacks.
4. **Plugins.** Isolated external integrations a `Plugin` node hands off to over NATS.

```
 your authoring surface
        │  save (verbatim)
        ▼
 your backend (inflow-fusion) ──REST──▶ Infra (spaces + engine registry)
        │  ▲                                     │
   answers │ 3 questions over NATS               │ round-robin pool
        ▼  │                                     ▼
        Fractal  ◀── POST /engine ── inflow.NewProcess().Exec()
        │
   Plugin node ──NATS──▶ isolated plugin process
```

---

## The build order

### 1. Stand up the platform

```bash
curl -fsSL https://raw.githubusercontent.com/Inflowenger/getting-started/main/install.sh | bash
```

Two containers — Infra and Fractal — on a shared Docker network. Infra prints an **API
Secret Key**; keep it. Ports: `8022` (Infra HTTP), `4222` (NATS), `8222` (NATS monitor).

Full detail: [Part I — Topology](../01-platform/topology.md).

### 2. Decide your authoring surface

The question is not *"canvas or not"* but *"what does the user author, and what do I
persist?"*

| Surface | Compiler | Persist |
| --- | --- | --- |
| Vue Flow / React Flow canvas | the shipped `compilers/vueFlow` | the raw `{nodes, edges}` graph |
| A YAML or JSON DSL | [write one](compiling-a-yaml-dsl.md) | the source text |
| A form wizard | usually trivial | the form document |
| An API only | your caller's payload | that payload |

**Persist the author's document verbatim, and compile separately.** This is the single
most valuable convention in this chapter — it is what keeps the editor lossless and lets
you change a node's lowering without a data migration.

### 3. Define your palette, and its mapping

Write the table before the code. One row per node type your users see:

| Your node | Lowers to | Config fields |
| --- | --- | --- |
| *HTTP Request* | Plugin (`http`) | url, method, headers, body |
| *If* | Contract | condition, then-tags, else-tags |
| *Set* | Code (js) | assignments |
| *Save to DB* | Extrinsic (`svc.persist.*`) | table, payload |
| *Wait for approval* | Extrinsic (`svc.hitl.add`) | questions, assignee |
| *Sub-workflow* | GoTo | target flow, return node |

If a row has no plausible primitive, it is a **Plugin**. That is what Plugin is for, and
reaching for it is not a failure.

FloMorphic publishes exactly this table, in the product UI, for all thirteen of its nodes —
see [Part IV](../04-flomorphic/the-palette.md).

### 4. Implement the backend contract

```go
type MyService struct{ db *sql.DB }

func (s *MyService) RetrieveFlow(msg *nats.Msg)    { /* look up compiled flow, reply */ }
func (s *MyService) RetrieveContext(msg *nats.Msg) { /* look up ContextDoc, reply */ }
func (s *MyService) UpdateContext(msg *nats.Msg)   { /* persist ContextDoc */ }

func main() {
    inflow.InitBackend(context.Background(), &MyService{db: db},
        inflow.WithInfraApi(os.Getenv("INFLOW_INFRA_API")))
    select {}
}
```

Store `ContextDoc.Header` **verbatim**. It is engine-managed memory; treat it as opaque.
Full detail: [The backend contract](the-backend-contract.md).

### 5. Write the compiler hook

One function, one switch. See [the compiler seam](the-compiler-seam.md#what-the-hook-actually-does).

### 6. Register your domain actions

Everything your product can already do becomes available to the graph:

```go
svcHandler.ImplHandlerOnSubject("orders",
    svcHandler.SvcTopic("svc.orders.{ACTION}"),
    func(header nats.Header, data []byte) ([]byte, error) { ... })
```

One wildcard registration can back many palette cards.

### 7. Run it

```go
inflow.NewProcess(startNodeId,
    inflow.WithFlowId(flowId),
    inflow.WithContextDocument(contextId),
).Exec(ctx)
```

### 8. Show the run

Subscribe to the process event stream and feed it to `@inflowenger/flow-trace` to animate
your canvas. See [Observing a run](observing-a-run.md).

---

## What you inherit on day one

The point of this exercise is what arrives without being asked for.

**A durable engine.** Crash-resumable, cyclic, join-aware, with a node-visit cap and
per-request and whole-process timeouts. Not a feature you built — a property of the thing
you bound to.

**Horizontal scale without a code change.** Start a second Fractal; it registers with
Infra; `ReloadResources` picks it up; new runs round-robin across both. Pin a specific
instance when you need to.

**Multi-tenancy as an architecture, not a `WHERE` clause.** Infra models tenants as NATS
accounts — *spaces* — with credentials scoped so one tenant's plugins cannot observe
another's traffic. See [Part VI](../06-architecture/spaces-and-isolation.md).

**A full integration catalog.** This is the one people underestimate. Plugins target the
`inflowv1` protocol, not any product. Every plugin in the
[catalog](../03-plugins/catalog.md) — Jira, Postgres, MySQL, MongoDB, ClickHouse, Qdrant,
Gmail, Telegram, GitHub, Google Workspace, Scrapli — is available to your product the day
you start. Your integration list starts full.

**Three SDK languages for extensions.** Go (reference), Node/TypeScript, Python —
wire-identical. Contributors pick a language, not a protocol.

**Frontend packages.** `@inflowenger/flow-trace` turns the event stream into movement on
your canvas; `@inflowenger/plugin-form-builder` renders a plugin's declared form. Both on
npm. See [Part III](../03-plugins/forms-and-ui.md).

---

## What stays yours

Deliberately:

- **The product.** Palette, branding, pricing, docs, the vocabulary users think in.
- **The data.** Your database, your schema, your backups, your residency. The engine never
  connects to it.
- **The domain logic.** Extrinsic handlers are your code in your process.
- **The authoring experience.** The hardest thing to differentiate on, and the thing you
  most want to own.

---

## Three products, one runtime

| | **FloMorphic** | **Venapce** | **A CI product** |
| --- | --- | --- | --- |
| Authoring | Vue Flow canvas | FloMorphic workflows | `.ci.yaml` in a repo |
| Vocabulary | LLM, MCP, Rule, Stores, HITL | posture features | job, step, matrix |
| Storage | SQLite + `sqlite-vec` | Postgres | yours |
| Compiler | `compilers/vueFlow` + hook | *(inherits FloMorphic's)* | a YAML compiler + hook |
| Runtime change | none | none | none |

Venapce is the interesting column: it did not implement the backend contract at all. It
built its logic *as FloMorphic workflows* and kept only the view. That is a third
integration tier — above the SDK, above plugins — and [Part V](../05-venapce/) is about it.

---

## A checklist

- [ ] Platform installed; API Secret Key recorded
- [ ] Authoring surface chosen; **source document persisted verbatim**
- [ ] Palette → primitive mapping table written down
- [ ] `IInflowService` implemented over your storage
- [ ] `ContextDoc.Header` stored and returned **unmodified**
- [ ] Compiler hook written, using `nodes.*` builders
- [ ] `Next` left entirely to the compiler
- [ ] Domain actions registered via `ImplHandlerOnSubject`
- [ ] **No direct `nats.Connect` anywhere in your code**
- [ ] A pause/resume path designed if flows wait on people
- [ ] Event stream wired to the UI

---

## Next

- **[Observing a run](observing-a-run.md)** — the last piece: seeing what happened.

**Source material:** the blog post *Build Your Own Workflow Product*,
`inflow-fusion/README.md`, `FloMorphic/getting-started/docs/concepts.md`.

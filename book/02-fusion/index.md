# Part II — The Fusion Layer

> **The part you don't have to write.**

Everyone who has looked at n8n, Zapier, Dify, Temporal or GitHub Actions has had the same
thought at least once: *I could build that.* Drop a canvas on the page, let people wire
boxes together, run the boxes in order. How hard can it be?

The canvas is not the hard part. [Vue Flow](https://vueflow.dev) or
[React Flow](https://reactflow.dev) gives you drag, drop, edges and handles in an
afternoon. A YAML parser is an afternoon too.

The hard part is underneath: an execution engine that takes a graph and *actually runs
it* — durably, across crashes, through loops, pausing for days while it waits on a human
or an event, then resuming exactly where it left off, with every step observable. That
engine is months of work, and it is the part that decides whether you have built a demo
or a product.

Inflowenger ships that engine as a reusable runtime. **`inflow-fusion` is the seam you
bind to it.**

---

## What this part answers

This is the centre of the book, because it is where the platform's central claim is
either made concrete or exposed as marketing. The claim:

> **Any authoring surface, and any format that describes steps and transitions, can be
> compiled to a flow this runtime executes — without changing the runtime.**

Not *a canvas can*. Any of them. A Vue Flow graph. A React Flow graph. A YAML file shaped
like a GitHub Actions workflow. A JSON data model minted by a user in a form. A
domain-specific language you invent this afternoon. The runtime never learns your format
exists.

Six chapters build that argument, and four more make it usable:

| Chapter | What it establishes |
| --- | --- |
| [The compiler seam](the-compiler-seam.md) | The single architectural idea the whole claim rests on |
| [The primitive node reference](node-primitives.md) | The six things the engine can execute, and why there is no seventh |
| [Tag routing](tag-routing.md) | How *every* branch, gate, policy and agent decision is one mechanism |
| [The coverage argument](coverage.md) | Every requirement a workflow product ships, mapped to a primitive — including the awkward cases |
| [The backend contract](the-backend-contract.md) | The three questions your backend answers, and nothing more |
| [The wire](the-wire.md) | A real backend walked end to end — process rows, the traversal snapshot, the error ledger, and a run that finishes hours later |
| [Compiling a canvas](compiling-a-canvas.md) | The shipped compiler, worked end to end |
| [Compiling a YAML DSL](compiling-a-yaml-dsl.md) | The *same* seam, on a format with no graph library at all |
| [Build your own workflow product](build-a-workflow-product.md) | The assembly instructions: what you write, what you get |
| [Observing a run](observing-a-run.md) | The event stream, and turning it back into movement on your canvas |

---

## The split: already written vs. yours

The cleanest way to see the deal is to divide the system in two.

**Already written — the reusable runtime:**

- **The engine (Fractal).** Given a compiled node map and a start node, it walks the
  graph, executes each node by type, follows edges, loops on backward edges, joins
  parallel branches, and persists state as it goes. *Durable and resumable is a property
  of the engine*, not a feature you bolt on. Register one instance or twenty; new runs
  round-robin across them.
- **Infra, the control plane.** A NATS substrate plus the account manager on top of it.
  It carves the system into isolated **spaces**, issues scoped credentials, and keeps a
  live registry of every engine instance. It is why the same product runs on a laptop and
  in a cluster without changing shape.
- **The plugin isolation model — and the catalog that comes with it.** External
  integrations run as their own processes, credential-scoped so one cannot observe
  another's traffic. Because plugins target the *protocol* rather than any one product,
  **every plugin already written for the ecosystem works on your product on day one.**
  Your integration catalog starts full, not empty.
- **The primitives and the compiler contract.** A small, fixed set of node types, and a
  defined seam for turning your format into the engine's node map.

**Yours to write — the product:**

- **The authoring surface.** A canvas, a YAML schema, a wizard, an API — whatever your
  users should be handed.
- **Your data.** Flow definitions, run contexts, your domain entities, stored however you
  like. The engine never touches your database.
- **The compiler hook.** One function mapping each of your node types to a primitive.
- **Your domain actions.** Business logic a node should be able to call, exposed as
  *extrinsic* services.

The engine is the moat you did not have to dig. Everything else is product surface you
would want to own anyway.

---

## Where `inflow-fusion` sits

`inflow-fusion` is **not a standalone application**. It is a Go library you import into
your own backend so that backend can participate in the platform.

```
 ┌──────────────────────┐   REST: creds, accounts, engine registry   ┌────────────────────────┐
 │  Infra               │◄──────────────────────────────────────────►│  YOUR BACKEND          │
 │  control plane       │                                            │  imports inflow-fusion │
 │  NATS + accounts     │   NATS: get flow · get/set context         │  implements            │
 │                      │◄──────────────────────────────────────────►│  IInflowService        │
 └──────────┬───────────┘                                            └───────────┬────────────┘
            │ registered engines                            POST /engine         │
            ▼                                               (inflow.NewProcess)  │
 ┌──────────────────────┐                                            ◄───────────┘
 │  Fractal (engine)    │  walks the compiled node map
 │  executes the graph  │  asks your backend for what it lacks
 └──────────┬───────────┘
            │ NATS, on an isolated scoped plugin account
            ▼
 ┌────────────────────────────────────┐
 │  Plugin processes (inflowv1)       │
 └────────────────────────────────────┘
```

Your backend never executes a flow. It answers questions, and when a node calls out to it,
it runs a bit of business logic. The engine does the traversal.

---

## The load-bearing decision

One design choice carries this entire part:

> **The engine never touches your database.**

It knows how to ask *"give me the flow with this id"* and *"give me / take this context"*
over well-known NATS subjects, and nothing else. It has no schema, no driver, no
migration, no opinion about your storage.

That is precisely what makes one engine reusable across completely different backends.
Each backend answers those three questions however it likes. A SQLite file, a Postgres
cluster, an in-memory map in a test — the engine cannot tell the difference and does not
try.

Everything else in Part II is downstream of that decision.

---

## Next

- **[The compiler seam](the-compiler-seam.md)** — start here; it is the idea the rest
  depends on.

**Source material:** [`inflow-fusion`](https://github.com/Inflowenger/inflow-fusion) —
`README.md`, `docs/architecture.md`, `docs/nodes.md`, `docs/compilers/`, `docs/infra.md`.

# Built under FloMorphic

> The chapter that makes Part VI worth its place in the book: a real product whose business
> logic is *entirely* workflows on somebody else's host — and the one place that claim
> needed amending once it shipped.

---

## What Venapce did not write

The list is the argument.

| | |
| --- | --- |
| `IInflowService` implementation | **none** |
| Compiler hook | **none** |
| `inflow.NewProcess` calls | **none** |
| Node types / a palette | **none** |
| `inflow-fusion` dependency | **not in `go.mod`** |

Venapce is not a runtime host. It never learns what a primitive is, never compiles a graph,
never dispatches a process to Fractal. FloMorphic does all of that, and Venapce asks it to.

## What it did write

A Go + Fiber backend, a Vue panel, a Postgres schema, a BI proxy — and the actual product:
**a set of FloMorphic workflows, packaged as [operations](operations.md)**.

Plus three integration pieces that are the real content of this chapter.

---

## 1. The bridge: an ordinary REST client

`internal/flomorphic` is an HTTP client against FloMorphic's public API. Nothing about it
is privileged:

| It calls | For |
| --- | --- |
| `GET /flow`, `GET /flow/id/:id` | the flow picker, and recording a run's flow title |
| `POST /flow/import` | landing an operation's workflow export as a flow |
| the context endpoints | writing the run's context document, and reading the outcome back |
| the process endpoints | launching a run, and following it |

The connection — API URL, shared JWT secret, infra host — is settable from the panel and
stored in Venapce's database, overriding the environment.

**This is the point of tier 3.** No special coupling, no SDK, no shared library: a product
integrating with FloMorphic looks exactly like a product integrating with any REST API,
and everything interesting happens in the flows on the other side.

---

## 2. The run bridge: a row becomes a context document

The single most important mechanism in the product, and it is about twenty lines of glue:

1. An operator picks a row and a flow → `POST /api/activities/run`.
2. Venapce **opens an activity** (`kind: run`, `status: running`) and records `flow_id`,
   `flow_title`.
3. It writes a **context document** shaped `{ subject, row, activity, history, input,
   outcome }` and launches the flow over that context, keeping `process_id`, `pid` and
   `context_id` on the activity row.
4. The flow reads `{{$.row.…}}` and leaves its conclusion in `$.outcome`.
5. When the process ends, Venapce reads the context back and **lifts `outcome` into the
   activity's typed columns** — `title`, `description`, `remediation`, `proof`, `facts`,
   `tags`, `data`.

`POST /api/activities/:id/sync` re-reconciles an activity against its run, and
`POST /api/activities/:id/stop` ends one — because a flow can pause for days and the panel
must not assume a run it launched is still the run that is live.

> **Note what this makes possible.** A flow author never learns Venapce's API. They read
> `$.row` and write `$.outcome`. The contract between the product and its own business
> logic is **a JSON document shape**, which is why a domain expert can take the pen.

Earlier activities travel in `history`, so flows compose over one row — see
[What Venapce is](what-it-is.md#why-history-is-the-clever-part).

---

## 3. The correction: Venapce is its own plugin

Venapce was first described as writing *no runtime code at all*. Shipping it produced one
exception, and it is more interesting than the clean claim was.

`venapce-api` depends on **`github.com/Inflowenger/go-plugin-sdk`** and registers an
`inflowv1` plugin node **in-process**:

> *"It is not a standalone binary: it connects to infra over NATS using the plugin env
> venapce already stores, and its handlers use venapce's **own** database pool and osctrl
> client. There is therefore **no settings profile** — the connections it needs are already
> venapce's."*

### Why that is the right shape

Consider the alternative. To let a flow write a finding, Venapce could have exposed a REST
endpoint and used FloMorphic's generic HTTP node. That would mean: an auth scheme for it, a
settings profile per environment holding a credential, a hand-written form for each call,
and a node whose failure modes are "some HTTP request went wrong".

Instead the plugin is *inside the process that already has everything*:

- **No settings profile**, because the pool and the osctrl client are already the
  backend's.
- **No credential to store**, rotate or leak — the plugin env is the one Venapce already
  holds for FloMorphic.
- **Real typed actions with real forms**, because that is what the SDK gives you.

Generalised, and this is the reusable idea in this part:

> **If your product already owns the database, the credentials and the connections, the
> cheapest way to expose it to a workflow engine is to be a plugin, in-process.**

It costs one dependency and a lifecycle hook, and it is available to any tier-3 product in
any domain.

### The actions

| Action | |
| --- | --- |
| `db.stages.upsert` / `db.stages.update` | the pipeline inbox |
| `db.findings.upsert` / `db.findings.update` | a conclusion drawn from data |
| `db.issues.upsert` / `db.issues.update` | insert, or update in place when an existing `id` is supplied |
| `db.activities.upsert` / `db.activities.update` | record or complete an activity on a subject row |
| `osquery.query` / `osquery.queryByTags` | dispatch osquery SQL to a node, or to everything carrying a tag — run → poll → collect |
| `osquery.meta.nodes` / `.tags` / `.environments` | the pickers behind the query form's dependent fields |

The meta lookups are [`x-inflow-ui` dependent fields](../04-frontend/plugin-form-builder.md#pluginfn-the-platforms-one-action)
doing exactly what that mechanism is for: the Node and Environment pickers show what *this*
osctrl can actually see, because the form asked while it was open.

### The detail worth stealing: writable columns are reflected, not listed

The `db.*` actions do **not** hardcode their writable columns. They are derived by
reflecting over the **sqlc-generated models** (`model.Issue`, `model.Finding`,
`model.Stage`):

- Change `schema.sql`, run `make generate`, and a new column of a supported type
  **automatically becomes a writable field**; `RETURNING *` carries it back.
- Column names come **only** from the generated model, never from user input, so the SQL
  never interpolates an untrusted identifier. Every value is a bound parameter.
- String fields accept `{{$.path}}` flow tokens.

That is a genuinely good answer to the usual tension between "a generic write action" and
"SQL injection", and it means the plugin does not drift from the schema it writes to.

### Lifecycle

`plugin.Manager` starts at boot from the stored env and restarts when the operator saves a
new one (`PUT /api/settings/flomorphic`) or hits
`POST /api/settings/flomorphic/plugin/restart`. **`Start` is idempotent**: it fingerprints
the env and no-ops when unchanged. Status — running / pluginId / error — is surfaced on
`GET /api/settings/flomorphic`, so a misconfigured bridge is visible in Settings rather
than only in logs.

---

## So which tier is it?

Both, and the split is clean:

| | |
| --- | --- |
| **Logic** | tier 3 — FloMorphic workflows, driven over REST |
| **Reach** | tier 2 — an `inflowv1` plugin, in-process |
| **Host** | tier 1 — *not Venapce*. FloMorphic is the host |

Tier 3 never meant "no plugins". It means the **logic** is flows while the **reach** is
plugins — and Venapce's own database turned out to be one of the things worth reaching.

---

## What this generalises to

**If a FloMorphic-shaped host already exists in your domain, you may not need to build on
the runtime at all.** Build on the product, and inherit its canvas, its palette, its plugin
catalogue and its observability for the price of a REST client.

Then, if your product has data or connections the flows need, add an in-process plugin and
inherit typed nodes and dependent-field forms for the price of one dependency.

### The honest limits

- **You inherit the host's palette and its release cadence.** A capability FloMorphic does
  not expose is a plugin (tier 2) or a fork.
- **You inherit its authoring model.** Your users author in FloMorphic's canvas, in
  FloMorphic's vocabulary — which is why [operations](operations.md) exist: to hand a user
  a working feature without asking them to draw it.
- **Two systems to run.** The installer hides it, and `Settings → Connect FloMorphic` makes
  it recoverable, but it is still two containers and a shared secret.

Choosing tier 3 is choosing speed over control, and it is the right trade far more often
than teams assume.

---

## Next

- **[Operations](operations.md)** — how a set of flows becomes a shippable feature.

**Source material:** `venapce-api/go.mod`, `internal/plugin/` (`README.md`, `plugin.go`,
`manager.go`, `db/`, `osquery/`, `flow/`), `internal/flomorphic/`
(`client.go`, `runs.go`, `install.go`), `internal/httpapi/activities.go`,
`internal/httpapi/flomorphic.go`.

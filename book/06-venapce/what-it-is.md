# What Venapce is

> A panel over a pipeline, where every judgment in the pipeline was made by a workflow
> somebody drew.

---

## The data model: a three-level pipeline, and it is optional

The model is four tables, and the most important sentence about them is the one the schema
writes first:

> *"The pipeline between them is fully **OPTIONAL**: a FloMorphic flow — the expert user's
> rules — decides where a row lands."*

| Table | What it holds |
| --- | --- |
| **`stage`** | The pipeline **inbox**. Raw, un-triaged rows, noise and false positives included. A flow routes each one via `disposition` — `pending` \| `promoted` \| `held` \| `dropped` |
| **`findings`** | Something a process **concluded** from data: an observation with a `severity`, a `confidence` and a `target`, still to be validated |
| **`issues`** | Venapce's **main axis** — the rows that need validating, fixing, a mission, or otherwise acting on |
| **`activities`** | The **history** of a row in any of the three, including every flow run against it |

Raw data *usually* arrives on `stage`; a later process may turn a staged row into a
`finding` when it shows some aspect worth tracking; a finding that needs validation, a fix
or a mission becomes an `issue`. **But a flow may write straight to findings or issues —
nothing forces the order.**

That is the design, not a convenience. The seam between *what was collected* and *what was
judged to matter* is exactly where a workflow belongs — so the platform's job is to make
every level writable and let the flow decide, rather than to enforce a funnel.

### One vocabulary, so every row is self-describing

The three pipeline tables share a column set, which is what lets one panel render all of
them and one plugin write all of them:

| Column | |
| --- | --- |
| `source` | where the underlying **data** came from (connector / node / feed) |
| `origin` | which **process** produced this row (flow, query, api, manual …) |
| `ref` | structured **provenance** — how the row was made: flow id, run, rule, query, upstream ids. Any shape |
| `data` | the **payload / evidence** itself. Any shape |
| `meta` | **enrichment** attached by later processes. Any shape |
| `tags` | labels; `text[]` with a GIN index |
| `stage_id` / `finding_id` / `issue_id` | typed links between levels — **`0` means not linked** |

`data`, `meta` and `ref` are free-form `JSONB` *precisely because* each producer has its
own model, and the panel renders them as an explorable JSON tree rather than pretending to
know their shape.

Two columns earn their place on one table each:

- **`findings.fingerprint`** — a producer-chosen dedup key (rule + target, typically), so a
  flow can recognise a repeat rather than pile up duplicates.
- **`issues.tags`** — *the* sub-view mechanism. Rather than a table per issue type, every
  row carries tags and **a saved view is just a tag filter**. That is why there is no
  `vulnerabilities` table, no `incidents` table, and no schema change when a new class of
  issue appears.

---

## Activities: a flow run is a row's history

`activities` is where Venapce stops being a database with a dashboard.

Every row of `stage` / `findings` / `issues` is changed by hand and — above all — by
FloMorphic flows, and **each change is recorded against its subject** (`subject_kind` +
`subject_id`). `edit`, `promote` and `create` activities record manual and pipeline changes,
with `ref` holding the field diff.

The interesting kind is **`run`**: *one flow executed on one row.*

### The context document a run gets

When an operator sends a row through a flow (`POST /api/activities/run`), **the row becomes
the run's context document**:

```json
{
  "subject":  { "kind": "stage", "id": 21 },
  "row":      { "…the row: title, tags, data, meta, ref…" },
  "activity": { "id": 4, "flowId": "flow_…", "flowTitle": "…" },
  "history":  [ { "id": 3, "kind": "run", "title": "…", "facts": [] } ],
  "input":    "optional extra input given at launch",
  "outcome":  {}
}
```

The flow reads the row through `{{$.row.…}}` tokens and leaves its conclusion in
`$.outcome` — typically a `js` node with `key: outcome`. When the process ends, Venapce
reads the context back and **lifts `outcome` into the activity's typed columns**:

| key | becomes |
| --- | --- |
| `title`, `description` | what the flow concluded, in words |
| `remediation` | what to do about it |
| `proof` | the evidence the conclusion rests on |
| `facts` | `[{"k":"severity","v":"low"}, …]` — typed key/values the panel identifies and renders |
| `tags` | labels for the activity |
| `data` | the output document to keep (default: the final context minus `row` / `history`) |

A flow can also write these directly with the `db.activities.update` action
(`id = {{$.activity.id}}`), and **a key the outcome does not set never overwrites what the
flow already wrote** — so a long flow can report as it goes and still finish cleanly.

### Why `history` is the clever part

Earlier finished activities travel **in the context**, newest first. So a later flow can
read what earlier flows concluded about this same row, and **outcomes compound**:

> **collect → assess → decide → act**

Four flows, authored separately, possibly by different people, composing over one row
without any of them knowing about the others. That is the product-level cash-out of the
platform's central claim, and it is a database column rather than a framework.

---

## The assistant

Every feature is a flow, and some of those flows carry an AI node. Click a row and that
flow works it the way a person would — **bounded by the workflow designed for it**.

The reason this is a different proposition from an AI assistant bolted onto a dashboard is
structural, not a matter of prompt quality:

- **The steps are drawn.** The model acts inside a graph somebody authored.
- **The ports are finite.** An LLM node picks among outbound edges that already exist; it
  cannot invent an action. (See [tag routing](../02-fusion/tag-routing.md#the-exception-port--fail-and-still-route).)
- **The run is recorded.** The pid, the context id and the executed path are on the
  activity row, and the event stream says which edge fired and why.

Judgment applied to one row, inside a contract you drew, with a receipt.

---

## The panel

The Vue app is organised around the model above:

| Section | Views |
| --- | --- |
| **Pipeline** | Stage · Findings · Issues — each a paginated list plus a detail page with a JSON tree over `data` / `meta` / `ref` |
| **Activities** | The run log, and a detail page per activity |
| **Operations** | Installed feature packages, and a detail page per package → [Operations](operations.md) |
| **Fleet** | Nodes, and agent enrollment |
| **BI** | Dashboards, the chart builder, datasets |
| **Settings** | Superset, osctrl, and **Connect FloMorphic** |

Over the pipeline sits a **full BI chart and dashboard builder** — every chart drawn on
data that came from FloMorphic. Posture is not a fixed set of screens; you build the views
you need.

### The native dashboard model

A `dashboard.layout` is the grid the drag/resize editor reads and writes — an array of
cells referencing saved charts by id:

```json
[{ "chartId": 1, "x": 0, "y": 0, "w": 6, "h": 8 }]
```

Each `chart` carries its `query_context` (what to ask Superset) and `builder_state` (so the
front can reopen it for editing).

### The Superset decision

The browser does not talk to Superset. The service account lives in `venapce-api`, which
logs in, refreshes the token, injects CSRF and proxies the data endpoints. The old
"Connect to Superset" login screen is gone — **Superset is a connection configured in
Settings**, and its password is encrypted at rest with AES-256-GCM under `APP_SECRET_KEY`.

---

## What ships

**One container image, and behind it one Postgres.**

| Component | Role |
| --- | --- |
| **Go backend API** | Fiber v2 + pgx/v5 + sqlc. The service the panel talks to, the bridge to FloMorphic, and the in-process `inflowv1` plugin |
| **Vue web app** | The panel |
| **Superset** | gunicorn, synchronous — **no Redis, no Celery** |
| **nginx** | Front door, tying panel, API and dashboards behind one origin |
| **PostgreSQL** | One instance, two databases: `venapce` and Superset's `superset` metadata |

```
:80   nginx ──► /         venapce-wapp (Vue SPA)
              └► /api     venapce-api (Go)  ─┐
:8088 (→ host :8090)  Superset (gunicorn)    │  one PostgreSQL, two databases
                        └─────────────────────┴─►  venapce  +  superset
```

Packaged as a product image on a **permanent base image** — the base carries Postgres,
Superset, nginx and supervisord and is rebuilt only when Superset or the plumbing changes;
a product release layers the Go binary and the compiled panel on top, so it never
recompiles Superset. All state lives in one named volume.

FloMorphic is reached over the shared `inflow_net` network at `flomorphic:8025`,
authenticated with FloMorphic's shared secret.

---

## Installing it, and pointing it at FloMorphic later

One command. It checks for a FloMorphic instance, offers to install one, wires in the
shared secret, and starts Venapce. Panel on `:8080`, Superset's own UI on `:8090`.

Afterwards: **Settings → Connect FloMorphic** takes the API URL, the shared JWT secret and
the infra host, tests them, and stores them in Venapce's database — **overriding the
environment** and surviving a container recreate. *Reset to environment* undoes it.

Two operational traps, both worth repeating because both look like bugs:

- **The env is read once at startup**, and `docker compose restart` reuses the container's
  existing environment. Use `up -d` (a recreate), or your edit will look ignored.
- **Values resolve from *inside* the container**, so `localhost` is the container itself. A
  FloMorphic container on `inflow_net` is `http://flomorphic:8025`; one on your host is at
  the Docker gateway.

---

## Deliberate gaps

Recorded as a known state rather than hidden: **there is no app-level login yet.** The API
is open behind CORS. App auth and per-space scoping slot in as Fiber middleware later.

---

## Next

- **[Built under FloMorphic](built-on-flomorphic.md)** — what Venapce wrote, what it did
  not, and the in-process plugin.

**Source material:** `venapce-api/internal/store/schema.sql`,
`internal/plugin/README.md`, `internal/httpapi/`, `venapce-wapp/src/views/`,
`venapce-wapp/src/router/`, `Venapce/getting-started/README.md`,
`aio-superset/SUPERSET-INTEGRATION-STUDY.md`.

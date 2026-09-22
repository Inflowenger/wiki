# What Venapce is

> **Status: outlined.** In active development — past feasibility study, being built now.

## The data model: Stage → Issues

Two tables carry the whole model, with FloMorphic doing the data engineering between them.

| Table | What it holds |
| --- | --- |
| **Stage** | The raw intake — everything the agents collect, including the noise and false positives. |
| **Issues** | The enriched, real signal FloMorphic promotes: a raised problem, a fix, a security incident. |

Over both sits a **full BI chart and dashboard builder** — every chart drawn on data that
came from FloMorphic. Posture is not a fixed set of screens; you build the views you need.

## Sections planned

**1. Why the two-table model is the whole design.** The separation between *what was
collected* and *what was judged to matter* is exactly the seam where a workflow belongs.
Collection has no opinions; promotion is a flow.

**2. The assistant.** Every feature is a flow, and some carry an AI node. Click an issue and
that flow works it the way a person would — **bounded by the workflow designed for it**.
Exactly the steps in the graph, no false-positive actions. Judgment applied to one issue,
inside a contract you drew. This is the clearest product-level statement of the platform's
central promise.

**3. What ships.** A single container image plus one Postgres: a Go + Fiber backend, a Vue
web app, and Superset behind the backend for the BI layer.

**4. The Superset decision.** Why the browser stopped talking to Superset directly — the
service account lives in the backend, which logs in, refreshes the token, injects CSRF and
proxies the data endpoints. Superset became a *connection configured in Settings* rather
than a login screen.

**5. The native dashboard model.** `dashboard.layout` as a grid of cells referencing saved
charts; each chart carrying its `query_context` and `builder_state`.

**6. Installing it.** One command that checks for a FloMorphic instance, offers to install
one, wires in the shared secret, and starts Venapce. The panel on `:8080`, Superset's own UI
on `:8090`.

**7. Pointing it at FloMorphic later.** Settings → Connect FloMorphic. Stored in Venapce's
database, overriding the environment and surviving a container recreate. Includes the two
operational traps: values are read once at startup (so `up -d`, not `restart`), and they
resolve *from inside* the container (so `localhost` is the container itself).

**8. Deliberate gaps.** No app-level login yet; the API is open behind CORS, with app auth
and per-space scoping slotted in as middleware later. Recorded as a known state, not hidden.

## Source material

`Venapce/getting-started/README.md`, `venapce-api/README.md`,
`aio-superset/SUPERSET-INTEGRATION-STUDY.md`.

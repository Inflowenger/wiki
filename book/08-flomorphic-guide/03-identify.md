# 3 · Stage 1 — Identification

> **Thirty-five minutes.** Before a single flow: what does Northwind know, where does it
> live, and what can reach it? This is the stage that gets skipped, and skipping it is why
> most "AI on our data" projects stall at the demo.

## What a brain is made of

A brain is not one flow. It is a small number of flows with distinct **missions**, over one
body of context that is kept current. Name the missions before you draw anything, because
the mission is what decides whether a flow needs a trigger, a human, or a memory store.

| Mission | Owns | Runs | Chapter |
| --- | --- | --- | --- |
| **M1 · Know the sources** | The inventory, the plugins, the credentials | Once, then whenever a source changes | This one |
| **M2 · Ingest and ground** | The nightly sweep, the ad-hoc webhook, the vector store, the ontology | Nightly at 21:00, and on demand | [5](05-ingest.md) |
| **M3 · Decide** | One request → one defensible action | Per request | [6](06-decide.md) |
| **M4 · Keep humans in control** | Approvals, exceptions, the audit trail | Woven through M3, not a flow of its own | [6](06-decide.md) |

M4 is deliberately not a flow. An approval queue that is its own system is an approval queue
nobody reads; in this architecture the human is an `Extrinsic` node that parks the run, and
the queue is a consequence.

## The inventory

This is a table you write by hand, in a wiki, before you touch the product. Northwind's:

| # | Source | Holds | Reach it with | Volume | Changes |
| --- | --- | --- | --- | --- | --- |
| 1 | **Postgres** `crm` | Customers, contracts, SLA tiers, entitlements, cost ceilings | `postgres` plugin — catalog | 400 customers, ~1.2k contract rows | Daily, low |
| 2 | **MySQL** `ticketing` | A decade of tickets, resolutions, engineer notes | `mysql` plugin — catalog | ~380k tickets | Continuously |
| 3 | **ArangoDB** `assets` | Which asset depends on which service, who owns it, site topology | **nothing in the catalog** | ~60k vertices | Weekly |
| 4 | Vendor parts API | Lead times and stock for replacement parts | HTTP node — no plugin needed | — | Live |

Four columns of that table are the interesting ones.

**"Holds"** is not a schema dump. It is the sentence a person would say. If you cannot write
it, you do not yet know what the source is for, and an ingestion flow over it will produce
embeddings nobody can use.

**"Changes"** decides the ingestion strategy by itself. Daily-and-low is a nightly sweep.
Continuous is a nightly sweep *plus* a webhook for the cases that cannot wait. Weekly is a
nightly sweep that will usually find nothing, which is fine and costs nothing.

**"Volume"** decides whether you can embed everything (you cannot — 380k tickets is not a
first pass) and therefore forces the scoping conversation now rather than after the
embedding bill.

**"Reach it with"** is the column this chapter is about.

## Installing a plugin from the catalog

Two of Northwind's four sources are covered by published plugins. A plugin targets the
**`inflowv1` protocol**, not a product — so every plugin in the
[catalog](https://github.com/Inflowenger/plugin-catalog) works on any host that implements
the protocol, and a new product on this runtime starts with an integration list rather than
an empty one.

### What is in the catalog today

| Plugin | SDK | Actions | Runs on |
| --- | --- | --- | --- |
| ClickHouse · MySQL · Qdrant | Node | 4 · 4 · 7 | any host |
| Jira · MongoDB · Postgres · Nuclei | Go | 14 · 4 · 3 · 6 | any host |
| Scrapli · Playwright | Python / Node | 2 · 5 | any host |
| Gmail · Google Workspace · Telegram · GitHub | Node/Go/Python | — | **FloMorphic only** ★ |

★ These reach FloMorphic's central **Connect / OpenConnector** proxy over the
`flomorphic.svc.oc.*` NATS subjects, which only FloMorphic provides. The dependency is
recorded as `hostDependency` in the catalog's `index.json` — treat that file as
authoritative, because the table above will drift.

### Register it: Extensions → New extension

A plugin is a **process you run**; FloMorphic only needs it to reach the platform under an
id this install knows. So adding one is two steps: register the row, which mints the
identity, then take away whatever it takes to run it.

The **Extensions** portal offers two ways in, and the only difference is how much you
already have:

| | You have | It gives you |
| --- | --- | --- |
| **From a repo** | A GitHub URL | A one-liner that clones it, writes the env *including the credential*, builds and starts it in a directory you name |
| **Bring your own** | The plugin already | Just the `.env.inflow` file — which is the whole of what it needs to connect |

For Postgres, take the first. Give it a name, the repository
`https://github.com/FloMorphic/postgres-plugin`, and leave the runtime on **Detect**
(`go.mod` → Go, `package.json` → Node, `Dockerfile` → Docker). FloMorphic provisions the
plugin in a **space** — a NATS account managed by Infra — which is what creates its
`PLUGIN_ID` and credential. You never touch Infra.

### Run it

The extension page then shows an **Install** command, generated per extension. Copy it from
the page, not from here, because it carries that plugin's identity:

```bash
curl -fsSL "https://<your-flomorphic>/…/install.sh" | bash
```

> **That command embeds a live credential.** Treat it like a password — not into issues, not
> into chat, not into a script you commit.

Run it on the machine that will host the plugin. It clones the repo, writes `.env.inflow` at
the root, detects the language, runs that language's standard command, and injects
`plugin.sh` next to it:

| `./plugin.sh …` | Does |
| --- | --- |
| `start` / `stop` / `restart` | The language's standard command, with a PID file and log redirection |
| `status` | Running or not, PID, uptime, **and whether the last startup logged its subscriptions** |
| `logs` (`-f`) | `.inflow/plugin.log` |
| `update` | Pull the latest tagged release, reinstall, rebuild, restart |

Repeat for MySQL (`https://github.com/FloMorphic/mysql-plugin`).

### Confirm it actually registered

Two independent checks, and you want both:

1. **The startup log lists its subjects.** `./plugin.sh logs` should show
   `inflow.v1.<PLUGIN_ID>.*` subscriptions. **No log, no registration** — this is the single
   diagnostic that separates "running" from "silently misconfigured".
2. **The Extensions card says so.** Each card probes the plugin's `@intro` live through the
   backend's `inflowv1` proxy. It is asked, never stored, so a card that says live is live
   right now.

The common failures:

| Symptom | Cause |
| --- | --- |
| Starts, exits immediately | `.env.inflow` missing or not at the repo root |
| Starts, no subscription log | Wrong `INFRA_URL`, or `INFRA_CRED` from a different space |
| Registered, node not on the palette | Provisioned in a space this workspace does not load |

## Credentials off the graph: Node Settings

The Postgres plugin stores no database credentials and reads no database environment
variables. It **declares a settings form** — a full connection string, or discrete
host/port/database/user/password/SSL fields — and the platform ships the bound profile with
every call as `body.settings`.

So create one profile per database under **Node Settings**:

| Profile | Bound to | Holds |
| --- | --- | --- |
| `northwind-crm` | `POSTGRES` | The CRM connection |
| `northwind-ticketing` | `MYSQL` | The legacy ticketing connection |

Three consequences worth stating, because they are why this is not bureaucracy:

- **One running plugin serves many databases.** The profile names the database, not the
  process.
- **Rotating a password needs no restart.** Change the profile.
- **An exported flow carries no secret.** The node references a profile *by id*, and import
  resolves that id against the destination install's own profiles.

The Postgres plugin also ships a meta RPC, `postgres.meta.ping.check`, which backs the
settings form's test button and its submit validation. Use it. A profile that has never been
tested is a profile that will fail at 21:04 in chapter 5.

### Doing this part by chat

The bookkeeping is a reasonable thing to hand to the assistant:

```text
Use the flomorphic MCP server.

- flo_list_extensions: show me what is registered, with each plugin's identity.
- flo_list_node_settings: show me every profile and which node kind it is bound to.
- For anything in the first list with no profile in the second, tell me which
  connection values you need from me, then create it with flo_upsert_node_setting.

Do not invent connection values. Ask.
```

The last line is not decoration. A model given a half-specified connection will cheerfully
produce `localhost:5432/postgres`.

## The ontology sketch

The last artefact of stage 1 is the one with no UI: **what concepts does this organization
reason in?** Write it now, use it in chapter 5.

For Northwind, six concepts and the relations between them:

```
Customer ──has──▶ Contract ──grants──▶ Entitlement
   │                  │
   │                  └──sets──▶ SLA tier, cost ceiling, hours of cover
   │
   └──owns──▶ Asset ──depends on──▶ Service
                 │
                 └──subject of──▶ Ticket ──resolved by──▶ Resolution
```

This is not a formal ontology and does not need to be. It is a contract about vocabulary,
and it does three jobs in chapter 5:

1. It is the **tagging vocabulary** — every ingested record gets labelled with the concepts
   it mentions, which is what makes a vector search filterable rather than a lucky dip.
2. It is the **metadata schema** on every indexed vector, so a search can be narrowed to
   `{concept: "Entitlement", customer: "arcadia-mills"}`.
3. It is the **review checklist** — if a flow produces an answer that references something
   not on this diagram, either the diagram is wrong or the answer is.

Store it as rows in a document store (chapter 5 creates one), not as a diagram in a wiki,
because a flow can read a document store.

## The gap: ArangoDB

Source 3 has no plugin. Search the catalog: `arangodb` is not there, and neither is `neo4j`.
This is the normal case and the catalog is honest about it — **a plugin joins the catalog
when it is built and published, not when it is planned.**

So this is a real decision, and it has three honest answers:

| Option | Good when | Cost |
| --- | --- | --- |
| **HTTP node** | One or two fixed calls, a simple REST surface, no connection to hold | No form, no typed params, no reuse; the URL and auth live in a settings profile but the query shape lives on each node. Re-wire it per flow |
| **Build a plugin** | You will call it from several flows, it has a real connection, the author of a flow should see a form rather than a URL | A day's work, a process to run, a repo to maintain |
| **Push it upstream** | The data could live somewhere you already reach | A migration — which Northwind has already refused |

ArangoDB fails the HTTP-node test on two counts: the asset graph is queried from at least
three flows, and a graph traversal has enough parameters (start vertex, edge collection,
depth, direction) that a flow author should get a form, not a hand-assembled JSON body.

> **The deeper reason to build it.** An HTTP node is a call. A plugin is a *node on the
> palette*, which means it also appears in the **AI build prompt's "Plugins available"
> section** — with its action names and its parameter schema. The moment ArangoDB is a
> plugin, every assistant designing a flow in this install knows it exists and can reach for
> it instead of inventing an integration. That is the difference between a connection and a
> capability.

So: build it. That is [chapter 4](04-build-a-plugin.md).

## Stage 1 is done when

- [ ] The inventory table exists, with the "Changes" column filled in honestly.
- [ ] Postgres and MySQL plugins are registered, running, and their startup logs show
      subscriptions.
- [ ] A settings profile exists per database, and each has passed its ping check.
- [ ] The ontology sketch is written down somewhere a flow could read.
- [ ] Every gap is named, with a decision attached — HTTP node, build it, or out of scope.

That last checkbox is the deliverable. "We'll figure out the graph database later" is how
stage 2 produces a brain with a hole in it.

## Source material

`plugin-catalog/README.md`, `plugins/index.json`, `plugins/postgres.md`, `plugins/mysql.md`,
`docs/run-a-plugin.md` (§ The rule, § Start from FloMorphic) ·
`flomorphic-wapp/src/views/ExtensionsView.vue` (the two ways in) ·
`FloMorphic/getting-started/docs/nodes.md` § Around the canvas ·
[Part III — The plugin catalog](../03-plugins/catalog.md) ·
[Part VII — Spaces and isolation](../07-architecture/spaces-and-isolation.md) ·
[Part VII — Attaching a system you already run](../07-architecture/attaching-legacy.md).

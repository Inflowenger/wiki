# Part VIII — A guided build: the brain of the organization

> **A session, not a chapter.** Parts I–VII explain what the platform is. This part is
> one continuous build, from `curl | bash` to a system that answers a customer request
> against its own contracts — and it is written so it can be *delivered*: read alone, or
> run in front of a room with people typing along.

Everything before this is architecture. The risk with architecture is that it reads well
and still leaves you with an empty canvas. So this part takes a single scenario and builds
it end to end, in the order you would actually hit the problems.

## The scenario

**Northwind Field Services** — a mid-size B2B company that maintains industrial equipment
under contract. Four hundred customers, each on an SLA tier with its own entitlements and
response windows. Field engineers, assets in the field, a decade of ticket history.

Their data is where a decade puts it: a modern Postgres for CRM and contracts, a legacy
MySQL that still owns ticketing, and a graph database holding which asset depends on which
service and who owns it. Nothing is going to be migrated. Nothing is going to be rewritten.

What they want is a **brain**: something that knows what the organization knows, and can
answer *"Arcadia Mills wants an emergency restore at 02:00 on Saturday — are they entitled
to it, what will it cost, and who has to sign?"* with an answer you could defend in a
contract dispute.

That is the thing this session builds.

## What "the brain of the organization" actually means here

It is not one large flow, and it is not a chatbot with a database attached. It is three
things, in this order — and the order is the session:

| Stage | The question it answers | What exists at the end |
| --- | --- | --- |
| **1 · Identification** | *What does this organization know, and where does it live?* | A source inventory, the plugins that reach each source, credentials off the graph, and an ontology sketch |
| **2 · Ingestion** | *How does what it knows get into the brain, and stay current?* | A nightly flow at 21:00 and a webhook for *right now*, a vector store, an ontology, and a watermark that makes re-runs safe |
| **3 · Decision** | *Given a real request, what is the right action — and can we defend it?* | One flow that retrieves, decides, checks policy, asks a human when it must, answers, and leaves an audit row |

Each stage is a chapter. Each flow in each stage comes with the **prompt** that builds it,
because the second thesis of this session is that you do not have to draw any of it by
hand.

## The one habit to take from this session

> Every flow here is built, debugged, run and amended by **talking to an assistant that has
> FloMorphic's MCP server installed** — and every one of those flows is also editable on the
> canvas, by a person, with no assistant involved. Neither path is the real one.

That is not a convenience story. The product's own UI and an external agent are peers on one
API, which is a constraint the product had to hold rather than a feature it added — and you
will feel it in [Orientation](02-orientation.md), where you connect the assistant before you
draw a single node, and then never stop using it.

So each stage carries four prompts, always the same four moves: **build it · debug it · run
it as a test · amend it.** By chapter three you will be writing them yourself.

## Chapters

| Chapter | What you do | Rough session time |
| --- | --- | --- |
| [1 · Install](01-install.md) | One command, three questions, a canvas on `:8088` | 10 min |
| [2 · Orientation and the assistant](02-orientation.md) | The menus, the canvas, *AI build*, and connecting an assistant over MCP | 25 min |
| [3 · Stage 1 — Identification](03-identify.md) | Inventory the sources, install the plugins, hit the gap | 35 min |
| [4 · Filling the gap: build a plugin](04-build-a-plugin.md) | Write the ArangoDB plugin the catalog does not have, and watch the palette grow | 45 min |
| [5 · Stage 2 — Ingestion](05-ingest.md) | The 21:00 flow, the webhook, the vector store, the ontology, the watermark | 60 min |
| [6 · Stage 3 — Decision](06-decide.md) | A real request, decided against a real contract, with a human in the loop | 60 min |
| [7 · What you built](07-what-you-built.md) | The inventory, how it scales down and up, and what would falsify it | 15 min |

Total, as a workshop: about four and a half hours with breaks. As an article, it is one
afternoon.

## Prerequisites

- **Docker** and the Compose v2 plugin. That is the whole install prerequisite.
- **An assistant with MCP support** — Claude Code, Claude Desktop, Cursor, or anything that
  speaks Streamable HTTP. You will connect it in chapter 2.
- **Go 1.27+**, only for chapter 4 (the plugin). Skip that chapter and the rest still works.
- **An embedding provider** for chapter 5 — an OpenAI-compatible endpoint and a key. The
  vector store captures the provider once, at creation.
- No Kubernetes, no cluster, no Postgres of your own. Where the session needs a source
  database, it says so and gives you the smallest thing that stands in for one.

> **What this session does not do.** It does not secure anything for production. Auth is off,
> `/mcp` is open on localhost, and the webhook in chapter 5 uses a static token because that
> is the shortest thing that is not *nothing*. [Chapter 7](07-what-you-built.md#what-this-session-left-out)
> lists every shortcut taken, so none of them surprise you later.

## Source material

`FloMorphic/getting-started` — `README.md`, `docs/concepts.md`, `docs/nodes.md`,
`docs/mcp.md`, `docs/architecture.md` · `flomorphic-wapp/src/router`,
`src/components/layout/AppSidebar.vue`, `src/components/flow/` ·
`flomorphic-api/designer/`, `mcpserver/`, `inflow/` · `plugin-catalog/docs/` and
`plugins/index.json` · [Part V — FloMorphic](../05-flomorphic/) for the architecture this
session takes for granted.

# 2 · Orientation, and the assistant

> **Twenty-five minutes.** Learn the menus, open the canvas, meet *AI build*, and connect an
> assistant to FloMorphic's MCP server. From here to the end of the session you have two
> hands: the canvas and the chat.

## The chrome: header, sidebar, page

Three fixed pieces, and one of them answers a question you will ask all session.

```
┌──────────────────────────────────────────────────────────────────────────┐
│ ☰  ◆ FloMorphic by Inflowenger        ● Connected   ☾  v0.4.2           │  header
├────────────┬─────────────────────────────────────────────────────────────┤
│  BUILD     │                                                             │
│  Workflows │                                                             │
│  Extensions│                      the page                               │
│  Node Set… │                                                             │
│            │                                                             │
│  DATA      │                                                             │
│  Memory    │                                                             │
│  Contexts  │                                                             │
│  Prompts   │                                                             │
│            │                                                             │
│  OPERATE   │                                                             │
│  Human Task│                                                             │
│  Processes │                                                             │
├────────────┤                                                             │
│  Connect   │                                                             │
│  Settings  │                                                             │
└────────────┴─────────────────────────────────────────────────────────────┘
```

### The header, and the badge that matters

Left to right: a **sidebar toggle** (the sidebar collapses to a 60px icon rail, which is
worth doing on the canvas), the logo — a link back to Workflows — then on the right a
**status badge**, a **theme toggle** (light / dark / system) and the **app version**.

That badge reads **Connected** (green) or **Local** (grey), and it is the first thing to look
at when something does not work. It answers *"is this canvas talking to `flomorphic-api` at
all?"* — `Connected` means `VITE_API_BASE_URL` is set and reachable; `Local` means the app is
running **standalone with browser-local persistence**.

> **Two different axes, and conflating them wastes an afternoon.**
>
> | | Question | Where to look |
> | --- | --- | --- |
> | **Axis 1 — is there a backend?** | Can anything be saved server-side? | The header badge · **Settings → Backend** |
> | **Axis 2 — is there a runtime?** | Can a flow actually *execute*? | **Settings → Engine resources** |
>
> `Connected` + no engine still cannot run a flow. A canvas on `Local` can design, save
> (in the browser) and use *AI build*, and can never run anything. Both states are
> deliberate, and [chapter 1](01-install.md#the-thing-that-trips-everyone-the-runtime-is-optional)
> is why.

### The sidebar: three groups, two footer items

The grouping is the product's own mental model, and it reads as a sentence: you **build** a
thing, over **data**, then you **operate** it.

#### Build

| Item | What it holds | When you come back to it |
| --- | --- | --- |
| **Workflows** | Every flow. The list, and the editor behind it | Constantly. This is the product |
| **Extensions** | Third-party `inflowv1` plugins registered in this install | [Chapter 3](03-identify.md), to add Postgres and MySQL; [chapter 4](04-build-a-plugin.md), to add your own |
| **Node Settings** | Named configuration profiles bound to a node kind — an LLM provider, a database connection | Chapter 3, and every time a credential changes |

**Node Settings is the one people skip and regret.** A profile is where connection config and
provider tokens live, referenced by id from a node on the canvas — so credentials never live
on the graph, and a flow you export carries no secret. One running Postgres plugin can serve
five databases because the profile, not the plugin, names the database.

#### Data

| Item | What it holds | When you come back to it |
| --- | --- | --- |
| **Memory** | Vector and Document store *definitions*, and a browser for the rows inside them | [Chapter 5](05-ingest.md), heavily |
| **Contexts** | The run documents themselves — the JSON a run reads and writes — listed, opened, edited as a tree or raw | Every time you debug. This is the state |
| **Prompts** | A reusable prompt-template library with `{{var}}` placeholders and declared variables | Chapter 6 |

#### Operate

| Item | What it holds | When you come back to it |
| --- | --- | --- |
| **Human Task** | The queue the human-in-the-loop node feeds — open, answer, close | Chapter 6, when a decision needs a signature |
| **Processes** | Every run: status, duration, error, and a jump to the context it carried | Chapters 5 and 6, every single run |

Note the singular — the menu item is **Human Task**, not "Human Tasks".

#### The footer: Connect

Not an OAuth screen for one provider — a **gateway**. FloMorphic reaches a large catalogue of
SaaS providers through **oomol OpenConnector**, which holds the OAuth handshakes, the token
refresh and the API keys, *so your workflows and plugins never touch a provider credential*.
It requires a backend.

That is what the `-oc` plugins in the catalog — Gmail, Google Workspace, Telegram, GitHub —
depend on, and why they are marked **FloMorphic only**: they proxy over the
`flomorphic.svc.oc.*` NATS subjects, which only FloMorphic provides. This session does not
use Connect; know what it is so you recognise the dependency when you read a catalog entry.

#### The footer: Settings

Six sections, and two of them are load-bearing for this session:

| Section | What it is |
| --- | --- |
| **Appearance** | Light / dark / system |
| **Backend** | Connected or Standalone, and the API base URL it resolved. Axis 1 |
| **Engine resources** | **The engine instances a run is dispatched to**, read live from Infra. Dispatch round-robins across the pool; *pin* one to send every run to just that resource. There is a **Reload** to re-read it. Axis 2 — an empty pool is why a run sits at `scheduled` |
| **Node registry** | A link to its own page (below) |
| **MultiPlugin Credential** | A shared credential for running several plugins under one identity |
| **Local data** | The browser-local store, for a `Local` install |

**Engine resources is the check nobody finds on their own.** It is the only place in the UI
that tells you a Fractal engine registered itself with Infra — and "no engine" and "engine
down" look identical from a flow that will not run.

#### Settings → Node Registry

Its own page: *"define the nodes that make up the canvas palette."* **Builtins ship with
FloMorphic and are seeded on first run; extensions are `inflowv1` plugins imported by
users.** So the palette is data — a registry row — not a hardcoded list, which is the
product-side half of the claim that
[adding a node kind is a catalog entry plus a compiler case](../05-flomorphic/the-palette.md).

You will not edit it in this session. You will watch chapter 4's plugin appear in it.

## The canvas

**Workflows → New workflow** opens the editor. The header and sidebar **stay** — the editor
is a page like any other, and it adds its own toolbar row beneath the header.

### The toolbar, in its real groups

Left: a back arrow, the **workflow name** as an editable field, and an *Unsaved changes* hint.
Right, five groups separated by dividers:

| Group | Buttons | Note |
| --- | --- | --- |
| **AI** | `AI build` · `AI connect` | Deliberately first, as one segmented pair. Build puts nodes on *this* canvas; connect explains pointing your own client at the MCP server. Two answers to one question |
| **View** | `Tidy` · `Fit` · `Map` | Tidy re-lays the graph out left → right and **replaces hand-placed positions**. Fit frames everything. Map toggles the minimap |
| **Document** | `Import` · `Export` · `Snapshot` | Export downloads the design as JSON, **unsaved edits included**. Snapshot saves a PNG of the whole graph — the thing to use for a slide |
| **Runtime** | `Processes` · `Logs` · `Run` | Logs appears **only when connected to a backend**, and carries a red badge with the error count. Run knows whether you have unsaved changes |
| **Commit** | `Save` | The primary button |

Two more things appear in the editor without being buttons:

- An **in-flight banner** — a non-blocking heads-up when the flow you are editing still has
  runs or human tasks in flight, such as an unfinished human-in-the-loop session. In
  [chapter 6](06-decide.md) you will see it, because that flow parks.
- A **run HUD** on the canvas: which run the node and edge overlays belong to, and the
  control for switching between the runs live on this flow.

Select a node and an **inspector panel** opens on the right — the node's drawer. That is
where `title`, `key`, `scope` and everything kind-specific live, including the Start node's
triggers.

### The palette has two halves

**The builtins**, grouped by intent, each card annotated with the runtime primitive it lowers
to. That annotation is visible in the product, not buried in a compiler, and it is the most
useful thing on screen while you are learning:

| Group | Nodes |
| --- | --- |
| **Flow** | Start · Wait for All · Continue After · Goto |
| **AI & Logic** | LLM · Jev · MCP · Rule · JS · OPA |
| **Stores** | Doc Store · Vector Store · Cast / Mapping |
| **Integrations** | HTTP |
| **Human** | Human in the Loop |

Fifteen nodes, and between them they exercise **all six** runtime primitives. Nothing was
added to the engine to make any of them work. → [The node palette](../05-flomorphic/the-palette.md)

**Then the imported plugin actions**, below the builtins, grouped under the plugin that
contributed them as **collapsible accordions** — collapsed by default, because a plugin with
thirty actions would otherwise bury everything. A plugin whose actions declare more than one
`tags.class` splits into labelled, colour-coded sections under its accordion; a
single-service plugin renders as one flat list.

That is the product-side payoff of the SDK's `Tags` field, and it is why one binary can host
several logical products and still read clearly on a palette —
[chapter 4](04-build-a-plugin.md) is where you decide whether yours needs it.

### Three fields on every node

Whatever the kind, every node carries the same three, and they are mirrored onto the compiled
primitive:

| Field | Meaning | The mistake |
| --- | --- | --- |
| **title** | The human label | — |
| **key** | Where this node's output is written into the context | Leaving it empty on a node that emits a plain value. It has nowhere to go and lands under `unknow` |
| **scope** | The JSONPath slice this node reads and writes | Giving a wildcard scope to a node that routes. [Chapter 5](05-ingest.md#the-rule-that-will-bite-you-scope-cardinality) is where this bites |

`scope` is the field that does the most work and surprises people the most. It is a full
JSONPath, and **its cardinality decides how many times the node runs**. `$` runs it once over
the whole context. `$.records[*]` runs it once per element — a queue inside the one node, not
a branch per element. That is how you iterate a collection; there is no loop node.

### Where triggers live

Not in a menu. The **Start node's drawer** holds the webhook and the schedule for this flow —
one trigger per flow, which [chapter 5](05-ingest.md#the-shape-and-the-constraint-that-produces-it)
turns into an architecture. You will open it there.

### Watching a run

Press **Run**, pick a context document (or let it mint a fresh one), and the canvas animates
while the log drawer streams. That movement is `@inflowenger/flow-trace`, an ordinary npm
package doing exactly what it would do in your own product — nothing about it is reserved
for FloMorphic. → [Watching a run](../04-frontend/flow-trace.md)

## *AI build* — the first of the two AI paths

The first button in the toolbar's AI pair. Press it.

It does *not* call a model. This is the point, and it is worth being precise about, because
it is the opposite of what the button looks like:

1. You describe the workflow in plain language.
2. It **generates a prompt** — containing this install's real node catalog, the plugin
   actions you have registered, and a summary of the graph already on the canvas.
3. You copy that prompt and run it in whatever assistant you already use — a chat window, a
   desktop app, a subscription you already pay for.
4. You paste the JSON back. It is **validated and previewed** before a single node is added.

> **No key, no backend.** Nothing here calls a provider. The model never sees your install,
> only the prompt you carry to it. So this works on a subscription plan, and it works with
> browser-local persistence and no API configured at all.

The preview is the feature. An unknown node kind, or an edge leaving a port that does not
exist, is **dropped and named**. The things a model cannot possibly know — which settings
profile, which store id, which server URL — are listed as warnings for you to fill in.
Nothing is added until you apply it.

## The second path: connect an assistant over MCP

The FloMorphic API **is** an MCP server, mounted at `/mcp`, on by default. Every entity the
canvas edits is a tool over the same call path the web app uses — so an external agent and
the product's own UI are peers on one API.

### Claude Code

```bash
claude mcp add --transport http flomorphic http://localhost:8026/mcp
```

Then `/mcp` inside Claude Code lists the `flomorphic` tools. No bridge — Claude Code speaks
Streamable HTTP natively.

### Claude Desktop

**Settings → Connectors → Add custom connector**, name it `FloMorphic`, URL
`http://localhost:8026/mcp`. On classic builds that only show a `command`-based server list,
bridge it:

```json
{
  "mcpServers": {
    "flomorphic": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "http://localhost:8026/mcp"]
    }
  }
}
```

### Cursor and anything else

```json
{
  "mcpServers": {
    "flomorphic": { "url": "http://localhost:8026/mcp" }
  }
}
```

With `AUTH_ENABLED=true`, every one of these takes a bearer token — the connector's token
field, `mcp-remote --header`, or `claude mcp add --header`.

## Do not conflate the two directions

This catches people within an hour of starting, so name it now:

| | FloMorphic **as** an MCP server | FloMorphic's **MCP node** |
| --- | --- | --- |
| Who is the client | your assistant | the flow |
| Who is the server | this install | some external MCP server you point it at |
| What it is for | designing, running and inspecting flows from a chat | a flow calling a tool, or driving a model bound to a server's tools |
| Where you configure it | `claude mcp add …`, once | a node on the canvas, per flow |

You will use the first throughout this session. The second appears in
[chapter 6](06-decide.md) only if you want it to.

## The `flo_*` tools, grouped

All tools are prefixed `flo_` and mirror the REST surface — read and write for every entity,
plus the runtime actions a trusted client is given.

| Area | Tools |
| --- | --- |
| **Designer** | `flo_get_design_guide` · `flo_plan_patch` · `flo_apply_patch`, plus the `flo_design_workflow` prompt |
| **Workflows** | `flo_list_workflows` · `flo_get_workflow` · `flo_export_workflow` · `flo_import_workflow` |
| **Runs** | `flo_list_processes` · `flo_get_process` · `flo_start_process` · `flo_stop_process` · `flo_delete_process` |
| **Contexts** | `flo_list_contexts` · `flo_get_context` · `flo_upsert_context` · `flo_delete_context` |
| **Memory — vector** | `flo_list_memory_stores` · `flo_get_memory_store` · `flo_create_vector_store` · `flo_index_vector` · `flo_search_vectors` |
| **Memory — document** | `flo_create_document_store` · `flo_list_documents` · `flo_write_document` · `flo_update_document` · `flo_delete_document` · `flo_query_documents` |
| **Triggers** | `flo_list_triggers` · `flo_get_trigger` · `flo_set_webhook_trigger` · `flo_set_schedule_trigger` · `flo_delete_trigger` |
| **Human tasks** | `flo_list_human_tasks` · `flo_get_human_task` · `flo_answer_human_task` · `flo_message_human_task` · `flo_close_human_task` · `flo_delete_human_task` |
| **Prompts** | `flo_list_prompts` · `flo_get_prompt` · `flo_upsert_prompt` · `flo_delete_prompt` |
| **Node settings** | `flo_list_node_settings` · `flo_get_node_setting` · `flo_upsert_node_setting` · `flo_delete_node_setting` |
| **Extensions** | `flo_list_extensions` · `flo_get_extension` |

The designer tools are the same brain as *AI build*. `flo_get_design_guide` returns the
*exact* instructions the canvas dialog builds its prompt from — node catalog, scope and
branching rules, `{{$this}}` per-row templates, wiring and joining, and the plugin actions
available in *your* install.

## The prompt pattern

Here is the shape you will reuse for every flow in this session. Four moves. Learn them
once.

### Move 1 — Build

The single highest-leverage instruction is **make the assistant read the guide first.**
Models that skip it invent node kinds, wire ports that do not exist, and give routing nodes
wildcard scopes.

```text
Use the flomorphic MCP server.

1. Call flo_get_design_guide first and follow it exactly.
2. Build this workflow:

   <plain-language description of the flow>

   Constraints I know and you cannot:
   - <store ids, settings profile ids, URLs, anything install-local>

3. Call flo_plan_patch (NOT apply) and show me the returned `problems` list.
4. Explain any problem before fixing it, then apply with flo_apply_patch
   once I say go.
```

`flo_plan_patch` converts the patch into a Vue Flow graph and **compiles it without
saving**. You get the lowered node graph, or the compile error, plus a `problems` list that
flags exactly the mistakes the guide warns about: a routing node on a many-valued scope, a
no-op *Wait for All*, branches converging without a join. Read it before you apply.

### Move 2 — Debug

```text
Workflow <id> run <indexId> did not do what I expected.

- flo_get_process for the run — status, error, timings.
- flo_get_context for the context it carried — show me what each node actually wrote.
- flo_get_workflow and compare: which node's output is missing or wrong?

Tell me the single most likely cause before proposing a fix.
```

The second line is the one that matters. **The context document is the truth.** A flow that
"did nothing" almost always wrote something, and what it wrote names the broken node.

### Move 3 — Run it as a test

```text
Create a test context with flo_upsert_context containing exactly:

<the JSON a real trigger would deliver>

Then flo_start_process on workflow <id> with that contextId, poll flo_get_process
until it leaves `running`, and show me the final context.
```

A run needs a context document — that is what `contextId` is. Crafting one by hand is how
you test a webhook flow without firing a webhook, and a scheduled flow without waiting for
21:00.

### Move 4 — Amend

```text
Change workflow <id>: <the one thing that changed>.

Read it with flo_get_workflow first. Keep every node id that is not affected.
Plan with flo_plan_patch, show me the diff in words, then apply.
```

Amendment is the move you will make most. A policy threshold changes, a branch is missing, a
node is on the wrong scope. "Read it first, keep the ids, plan before apply" is the whole
discipline.

## A first design → run → inspect, right now

Before the scenario starts, do one throwaway loop so the machinery is proven:

1. **`flo_get_design_guide`** — load the authoring contract.
2. **`flo_apply_patch`** — a two-node flow: Start → an LLM node that summarizes
   `{{$.text}}` into `key: "summary"`, scope `$`. Read the returned `problems`.
3. **`flo_upsert_context`** — `{"text": "…a paragraph…"}`.
4. **`flo_start_process`** — the flow id and that context id.
5. **`flo_get_process`**, then **`flo_get_context`** — the summary is in the document.

If step 2 complains that the LLM node has no settings profile, that is correct and expected:
the model cannot know which provider you use. Create one under **Node Settings** and bind it
in the drawer, or pass its id in the prompt. This is the one piece of setup that is always
yours.

## What this chapter did not cover

The canvas has more surface than this: prompt templates with declared variables, the Cast
node's mapping UI, the raw-JSON context editor, log categories and filters. None of it is
needed before chapter 3, and all of it is better learned against a flow you care about.
→ [Part IV — The Frontend Layer](../04-frontend/) for what the browser half actually is.

## Checkpoint

An assistant that lists `flo_*` tools, one throwaway flow that ran, and a context document
with its output in it. Now the scenario.

## Source material

`flomorphic-wapp/src/router/index.ts` and `src/components/layout/AppSidebar.vue` (the menu
structure) · `src/components/flow/AiNodeImporter.vue` and `AiConnectDialog.vue` (the two AI
paths) · `FloMorphic/getting-started/docs/mcp.md` (clients, the `flo_*` catalog, the designer
prompt) and `docs/nodes.md` (the palette and the three universal fields) ·
`flomorphic-api/designer/assets/preamble.md` (the design guide this chapter paraphrases) ·
[Part V — Driving it over MCP](../05-flomorphic/mcp.md).

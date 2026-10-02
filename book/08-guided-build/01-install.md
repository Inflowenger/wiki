# 1 · Install

> **Ten minutes, one command, three questions.** At the end of this chapter you have a
> canvas in a browser and an API that is also an MCP server.

## The one line

```bash
curl -fsSL https://raw.githubusercontent.com/FloMorphic/getting-started/main/install.sh | bash
```

Docker and the Compose v2 plugin are the only prerequisites. The script pulls a published,
baked image — it never compiles anything at install time — and asks three things.

| It asks | Why it asks | What to answer here |
| --- | --- | --- |
| **Install directory** | Where the compose stacks and the SQLite database land | Anything. `~/flomorphic` is fine |
| **Platform: the one already running, or a new one?** | FloMorphic is a product *on* the Inflowenger runtime, and a new platform is installed by [the Inflowenger installer](https://github.com/Inflowenger/getting-started) itself — one source of truth, not a copy | **A new one**, unless you already have Infra + Fractal running |
| **Advanced options (optional)** | Port overrides, if `8088` or `8026` are taken | Skip it |

Everything it asks can be set with an environment variable instead; `ASSUME_YES=1` runs it
unattended. The header of `install.sh` is the reference.

The builtin plugin nodes are deliberately *not* one of the questions. They are what the
canvas's stock LLM, MCP, Cast, HTTP and Jev nodes run as, so the image always bakes them in.
That is a packaging choice, not a privilege — see
[Builtin nodes are plugins](../05-flomorphic/builtin-nodes.md).

## What lands on disk

```
<install dir>/
├── platform/          Infra + Fractal        (only when it installs one for you)
├── inspector/         inflow-inspector       (only with INSTALL_INSPECTOR=1)
└── flomorphic/
    ├── docker-compose.yml
    ├── .env           image ref, ports, and the shared API Secret Key
    └── data/          the SQLite database — workflows, contexts, prompts, vectors
```

Note `data/`. Everything you build in this session — every flow, every context document,
every vector — is in one SQLite file in that directory. Back it up by copying it; reset the
session by deleting it.

## Where everything is listening

| Service | URL | What it is |
| --- | --- | --- |
| **Canvas** | <http://localhost:8088> | The web app. Start here |
| **API** | <http://localhost:8026> | REST + the MCP server at `/mcp` |
| **Infra** | <http://localhost:8022> | The control plane's REST base |
| NATS | `localhost:4222` (monitoring `8222`) | The bus everything talks over |

> The installed canvas and API sit on `8088`/`8026` rather than the from-source `5173`/`8025`
> so that a FloMorphic install and an `inflow-inspector` install can run side by side.

## Verify, in that order

```bash
cd <install dir>/flomorphic
docker compose logs -f          # follow the boot; Ctrl-C when it settles
```

Four checks, cheapest first, and each one rules out a different thing:

1. **The canvas loads.** <http://localhost:8088> shows the Workflows list, empty.
2. **The canvas has a backend.** The badge in the top-right of the header reads
   **Connected** (green), not **Local** (grey). `Local` means the app is running standalone
   with browser-local persistence — it will design and save in the browser and never run
   anything.
3. **The API answers.** `curl -s http://localhost:8026/mcp` returns something other than a
   connection refusal. (It is a Streamable-HTTP endpoint, so a bare GET is not a meaningful
   request — you are testing that the port is live and `/mcp` is mounted.)
4. **An engine registered.** **Settings → Engine resources** lists at least one resource,
   read live from Infra. This is the check people never find on their own, and the one that
   explains most "nothing happens" — see below.

Day-to-day:

```bash
docker compose down             # stop
docker compose up -d            # start again
```

## The thing that trips everyone: the runtime is optional

FloMorphic's API runs perfectly well with **no platform behind it**. Leave
`INFLOW_INFRA_API` unset and it serves CRUD only: you can design workflows, save them,
manage prompts and stores, and use *AI build* — everything except actually executing
anything. Set it, and *Run* goes live.

This is a real design position, not a degraded mode. Authoring works on a laptop with no
infrastructure, which is why the canvas can also run entirely browser-local. But it means
**"I saved a flow and nothing happened"** has two completely different causes, and the first
thing to establish is which one you are in.

| Symptom | Cause | Where to confirm it |
| --- | --- | --- |
| Header badge says **Local** | No backend at all — `VITE_API_BASE_URL` unset or unreachable | **Settings → Backend** |
| *Run* does nothing, no process row appears | No runtime — `INFLOW_INFRA_API` unset, or Infra is down | **Settings → Engine resources** says to connect a backend, or the pool is empty |
| A process row appears and sits at `scheduled` | Runtime reachable, but **no Fractal engine registered** | **Settings → Engine resources** — an empty pool. Hit *Reload* to re-read it from Infra |
| A process row runs and a node errors | You have a real runtime and a real bug. Good — that is chapter 5's problem | The run's log drawer, and the context document |

Those are **two independent axes** — *is there a backend* and *is there an engine* — and
telling them apart first is the single biggest time-saver in this session.
[Chapter 2](02-orientation.md#the-header-and-the-badge-that-matters) has the table.

Two variables bind everything to the platform: `INFLOW_INFRA_API` (Infra's REST base — the
NATS endpoint is derived from its host) and `INFLOW_INFRA_JWT_SECRET` (the platform's **API
Secret Key**). The installer sets both. You will need the secret again in
[chapter 4](04-build-a-plugin.md) only if you provision a plugin by hand, which this session
avoids.

## Two environment flags worth knowing now

Both live in `<install dir>/flomorphic/.env`, and both matter for the next chapter.

| Variable | Default | What it does |
| --- | --- | --- |
| `MCP_ENABLED` | on | Mounts the MCP server at `/mcp`. Set `false` to leave it unmounted |
| `AUTH_ENABLED` | off | Puts every route — `/mcp` included — behind a bearer token |

Leave both at their defaults for the session. Auth off is the right answer for a
localhost-only install and the wrong answer for anything else; the MCP tools include full
write access and runtime control, so an exposed `/mcp` is an exposed API.
→ [Driving it over MCP § Authentication](../05-flomorphic/mcp.md)

## If you are going to change FloMorphic, not just use it

Not needed for this session, but worth knowing it exists: you can take the container apart
and run the product layer from your own checkouts against the installed platform — API on
`:8025`, canvas on Vite's `:5173`, plugin nodes with `go run .`. The platform stays
installed. `FloMorphic/getting-started` → `docs/development.md` is the full guide.

One failure mode from that path shows up often enough to pre-empt: **a Go build that dies on
`403 Forbidden`** is a network problem, not a broken checkout. `proxy.golang.org` redirects
module zips to `storage.googleapis.com`, which some networks block. `export
GOPROXY=https://goproxy.cn,direct` and it goes away. The installed stack never hits this —
everything is baked.

## Checkpoint

Before chapter 2, you should have: a canvas at `:8088` with an empty Workflows list, a header
badge reading **Connected**, an API at `:8026`, and **at least one resource under Settings →
Engine resources**. If that last one is empty, fix it now — chapters 5 and 6 both end in a
run.

## Source material

`FloMorphic/getting-started/README.md` (§ Install, § Running from source) and
`install.sh` · `docs/architecture.md` § One container, on purpose · `docs/mcp.md` ·
`docs/development.md` · [Part V — What FloMorphic is](../05-flomorphic/what-it-is.md) ·
[Part I — Topology and installation shapes](../01-platform/topology.md).

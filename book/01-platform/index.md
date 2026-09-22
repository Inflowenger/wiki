# Part I — The Platform

> **What Inflowenger actually is.**
>
> **Status: outlined.** Chapter theses and section plans are committed; prose is pending.

Inflowenger is a **runtime for context processing** — a substrate for building software
whose logic is a workflow graph. It is positioned as a general-purpose engine for large
classes of systems (ERP, CRM, automation platforms, AI harnesses) where an operator or
super-admin must define and change logic **without redeploying the base system**.

Comparable in spirit to n8n, but as a *computational model* rather than an app.

## The shape of it

Two headless services, and nothing else is required:

- **Infra** — bootstraps and coordinates everything. Runs an embedded NATS server; mints
  accounts, credentials and the onboarding portal. *Everything starts here.*
  Ports `8022` (HTTP API), `4222` (NATS), `8222` (NATS monitor).
- **Fractal** — the runtime that executes workflow graphs. It registers itself with Infra.

Optional on top: the **dev panel** (`inflow-inspector-api` + `inflow-inspector`), a visual
window into context, workflows and Fractals — itself built on Inflowenger via
`inflow-fusion`, which makes it the reference consumer as well as a tool.

```
                          ┌───────────────────────────────────────────┐
    Browser  ─────────►   │  Dev panel frontend (Vue)      :8080      │
                          └──────────────────┬────────────────────────┘
                                             │ HTTP + WebSocket (logs)
                          ┌──────────────────▼────────────────────────┐
                          │  inflow-inspector-api (Go/Fiber)  :8025   │
                          └──────────────────┬────────────────────────┘
                                             │ NATS + HTTP (inflow_net)
      ┌──────────────────────────────────────▼────────────────────────────┐
      │                       Platform  (network: inflow_net)             │
      │   ┌─────────────────────────┐  register  ┌────────────────────┐   │
      │   │  Infra   :8022 / :4222  │◄──────────►│  Fractal (runtime) │   │
      │   │  NATS + coordinator     │            │  executes flows    │   │
      │   └─────────────────────────┘            └────────────────────┘   │
      └───────────────────────────────────────────────────────────────────┘
```

## Chapters

| Chapter | What it establishes |
| --- | --- |
| [The computational model](computational-model.md) | Context · Workflows · Fractals · Adapters, and the OS analogy |
| [The six primitives](the-six-primitives.md) | The closed set, from the platform's side |
| [Infra — the control plane](infra.md) | What it owns, what it guarantees, what its contract is |
| [Fractal — the execution engine](fractal.md) | What executing a graph actually involves |
| [Topology and installation shapes](topology.md) | Laptop → cluster, and the install paths |

> **On the closed components.** Infra and Fractal are not open source. These chapters
> document *what they do and what they promise*, not how they are built. That is sufficient
> to build on them, and it is the same information a source read would eventually yield.

## Source material

`Inflowenger/getting-started` README · `inflow-fusion/docs/architecture.md` ·
`inflow-fusion/docs/infra.md` · `inflow-plugin-sdk/docs/inflow-ecosystem.md` ·
`inflow-vue/inflow-nuxt/app/pages/index.vue` and `installation.vue`.

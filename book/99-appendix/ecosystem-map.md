# Ecosystem map

> **Status: outlined** (the tables are accurate today; the prose framing is pending).

## By layer

| Layer | Component | What it is | Open? |
| --- | --- | --- | --- |
| **Runtime** | **Infra** | Coordination + embedded NATS + credential minting + engine registry. Everything starts here. | closed |
| **Runtime** | **Fractal** | The execution engine. Registers with Infra, walks compiled node maps. | closed |
| **SDK** | `inflow-fusion` | The Go SDK: `InitBackend`, `IInflowService`, typed node builders, `svcHandler`, scoped credentials, the Vue Flow compiler. | open |
| **SDK** | `go-plugin-sdk` | Reference `inflowv1` implementation. Go 1.27+. | open |
| **SDK** | `node-plugin-sdk` | Node/TypeScript, on npm. Tracks the Go SDK. | open |
| **SDK** | `py-plugin-sdk` | Python, on PyPI. Beta. | open |
| **Frontend** | `inflow-js` → `@inflowenger/flow-trace` | Turns the process event stream into flow movement and completion. No dependencies, no framework. | open |
| **Frontend** | `inflow-js` → `@inflowenger/plugin-form-builder` | Renders `x-inflow-ui` on top of JSON Forms. Vue 3. | open |
| **Reference** | `inflow-inspector` + `inspector-api` | The low-level developer panel: edit raw primitives, inspect contexts and processes. Built on `inflow-fusion` — the worked example of consuming the SDK. | open |
| **Product** | `flomorphic-api` | Go 1.27 + Fiber v3, SQLite + `sqlite-vec` via sqlc, the Vue Flow → primitive compiler, `svc.*` handlers. | open |
| **Product** | `flomorphic-wapp` | The canvas: Vue 3 + Vite + TS + Vue Flow + Tailwind v4 + Pinia. | open |
| **Product** | `builtin-plugins` | `llm`, `mcp`, `cast`, `http`, `jev` — FloMorphic's stock nodes, as ordinary plugins. | open |
| **Product** | `venapce-api` + `venapce-wapp` | Go + Fiber + Postgres backend and Vue panel; Superset behind it. | — |
| **Catalog** | `plugin-catalog` | The plugin index and the plugin-developer knowledge base. | open |
| **Ops** | `Inflowenger/getting-started` | Installer for the platform (Infra + Fractal) and the inspector. | open |
| **Ops** | `FloMorphic/getting-started` | Installer and developer tooling for the FloMorphic stack. | open |
| **Ops** | `Venapce/getting-started` | Installer for Venapce (checks for / installs FloMorphic first). | — |
| **Site** | `inflow-vue/inflow-nuxt` | The marketing site, docs shell and blog corpus. | — |

## The dependency direction

```
 Venapce            ── workflows on ──▶  FloMorphic
   │                                        │
   │ plugins (osctrl, scrapli, github)      │ builtin-plugins (llm, mcp, cast, http, jev)
   ▼                                        ▼
 ─────────────────  inflowv1 protocol  ──────────────────
                         │
                    plugin SDKs (go · node · python)
                         │
 FloMorphic-api ── inflow-fusion ──▶ Infra ◀── Fractal
 inspector-api  ─────────┘
```

Read it as: **nothing points downward into the runtime except through a documented
contract**, and nothing in the runtime knows any product exists.

## The three integration tiers

| Tier | You write | Examples |
| --- | --- | --- |
| **1 — SDK** | a backend implementing `IInflowService`, a compiler hook, extrinsic handlers | FloMorphic, inflow-inspector |
| **2 — Plugin** | a process speaking `inflowv1` | Jira, Postgres, Qdrant, `llm`, osctrl |
| **3 — Workflows** | no runtime code at all — logic as flows on an existing host | Venapce |

## Sections planned

- How to choose a tier.
- What each layer guarantees to the layer above.
- Which repository answers which question (a routing table for the reader).

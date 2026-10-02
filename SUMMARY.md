# Summary

[The Inflowenger Book](README.md)

---

## Part 0 — Preface

- [About this book](book/00-preface/index.md)
- [The thesis: software as a graph over living context](book/00-preface/the-thesis.md)
- [Glossary](book/00-preface/glossary.md)

## Part I — The Platform

- [What Inflowenger is](book/01-platform/index.md)
- [The computational model](book/01-platform/computational-model.md)
- [The six primitives](book/01-platform/the-six-primitives.md)
- [Infra — the control plane](book/01-platform/infra.md)
- [Fractal — the execution engine](book/01-platform/fractal.md)
- [Topology and installation shapes](book/01-platform/topology.md)

## Part II — The Fusion Layer

- [Overview: the part you don't have to write](book/02-fusion/index.md)
- [The compiler seam](book/02-fusion/the-compiler-seam.md)
- [The primitive node reference](book/02-fusion/node-primitives.md)
- [Tag routing: one mechanism, three deciders](book/02-fusion/tag-routing.md)
- [Waiting: joins and delays](book/02-fusion/waiting.md)
- [The coverage argument](book/02-fusion/coverage.md)
- [The backend contract](book/02-fusion/the-backend-contract.md)
- [The wire: a backend, end to end](book/02-fusion/the-wire.md)
- [Compiling a canvas: Vue Flow / React Flow](book/02-fusion/compiling-a-canvas.md)
- [Compiling a YAML DSL](book/02-fusion/compiling-a-yaml-dsl.md)
- [Build your own workflow product](book/02-fusion/build-a-workflow-product.md)
- [Observing a run](book/02-fusion/observing-a-run.md)

## Part III — The Plugin Layer

- [Overview: the node that never compiles away](book/03-plugins/index.md)
- [The `inflowv1` protocol](book/03-plugins/inflowv1-protocol.md)
- [Plugin SDK specification](book/03-plugins/plugin-sdk-spec.md)
- [Jobs and commands](book/03-plugins/jobs-and-commands.md)
- [Forms: a node with its own UI](book/03-plugins/forms-and-ui.md)
- [The SDK matrix: Go, Node, Python](book/03-plugins/sdk-matrix.md)
- [The plugin catalog](book/03-plugins/catalog.md)

## Part IV — The Frontend Layer

- [Overview: two npm packages, and nothing else](book/04-frontend/index.md)
- [Watching a run: `flow-trace`](book/04-frontend/flow-trace.md)
- [The log taxonomy](book/04-frontend/log-categories.md)
- [Dynamic forms: `plugin-form-builder`](book/04-frontend/plugin-form-builder.md)
- [What a form can say: `x-inflow-notif`](book/04-frontend/notifications.md)
- [Building a process product on any frontend](book/04-frontend/build-a-process-product.md)

## Part V — FloMorphic

- [Overview: the runtime's first product](book/05-flomorphic/index.md)
- [What FloMorphic is](book/05-flomorphic/what-it-is.md)
- [The node palette, and what each node lowers to](book/05-flomorphic/the-palette.md)
- [Builtin nodes are plugins](book/05-flomorphic/builtin-nodes.md)
- [The AI harness](book/05-flomorphic/ai-harness.md)
- [How FloMorphic was built on the runtime](book/05-flomorphic/how-it-was-built.md)
- [Driving it over MCP](book/05-flomorphic/mcp.md)

## Part VI — Venapce

- [Overview: a product built on the product](book/06-venapce/index.md)
- [What Venapce is](book/06-venapce/what-it-is.md)
- [Built under FloMorphic](book/06-venapce/built-on-flomorphic.md)
- [Operations: features as installable flow packages](book/06-venapce/operations.md)
- [Collectors and the plugin roadmap](book/06-venapce/collectors.md)

## Part VII — Architecture in Practice

- [Overview](book/07-architecture/index.md)
- [Spaces and isolation](book/07-architecture/spaces-and-isolation.md)
- [Scaling out](book/07-architecture/scaling.md)
- [Customization: the seams that are open](book/07-architecture/customization.md)
- [Attaching a system you already run](book/07-architecture/attaching-legacy.md)

## Part VIII — Building on FloMorphic

- [Overview: a guided session](book/08-flomorphic-guide/index.md)
- [1 · Install](book/08-flomorphic-guide/01-install.md)
- [2 · Orientation, and the assistant](book/08-flomorphic-guide/02-orientation.md)
- [3 · Stage 1 — Identification](book/08-flomorphic-guide/03-identify.md)
- [4 · Filling the gap: build a plugin](book/08-flomorphic-guide/04-build-a-plugin.md)
- [5 · Stage 2 — Ingestion](book/08-flomorphic-guide/05-ingest.md)
- [6 · Stage 3 — Decision](book/08-flomorphic-guide/06-decide.md)
- [7 · What you built](book/08-flomorphic-guide/07-what-you-built.md)

## Appendix

- [Ecosystem map](book/99-appendix/ecosystem-map.md)
- [Repositories](book/99-appendix/repositories.md)
- [Further reading](book/99-appendix/further-reading.md)

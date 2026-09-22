# Part III — The Plugin Layer

> **The one primitive that never compiles away.**
>
> **Status: outlined.** Chapter theses and section plans are committed; prose is pending.

Five of the six primitives are compiled artifacts — after compilation nothing remains but a
rule body, a subject or a jump target. **Plugin is the exception.** It is a live external
process that the runtime calls into, and that difference is what makes it the ecosystem's
real extension point.

A plugin can do six things no compiled node can:

- **Its own UI.** Every action carries a form (JSON Schema + UI Schema) the host renders,
  so users configure the node visually. Fields can call back into the plugin *while the form
  is open*, so a picker shows what *this* account can actually see.
- **Context access.** Read and write the running flow's shared context by JSON path,
  mid-execution.
- **Flow control.** Stream progress, finish a job, route outbound ports, or end the branch.
- **Long life.** A persistent process holds connections, runs background loops, and surfaces
  queues, webhooks, hardware or third-party APIs as nodes on a canvas.
- **Independent deployment.** Your process, your cadence, your language, your infrastructure.
- **Isolation.** Narrowly-scoped NATS credentials — it can only publish and subscribe on
  subjects it owns.

> **The strong form of the claim:** with *only* the plugin node type, anyone can build a
> full workflow automation system on top of Inflowenger.

## A plugin holds no user credentials

Worth stating early because it surprises people. A plugin **declares** what a connection
needs; the platform stores the filled-in form as a named **settings profile** and folds the
values into every call as `body.settings`.

Consequences: one running plugin serves many accounts; rotating a token needs no redeploy;
credentials never live on the graph.

## Chapters

| Chapter | What it establishes |
| --- | --- |
| [The `inflowv1` protocol](inflowv1-protocol.md) | The wire contract — subjects, handshake, payloads |
| [Plugin SDK specification](plugin-sdk-spec.md) | What an SDK must implement, in any language |
| [Jobs and commands](jobs-and-commands.md) | The execution register: progress, context, routing |
| [Forms: a node with its own UI](forms-and-ui.md) | JSON Forms, `x-inflow-ui`, dependent fields |
| [The SDK matrix](sdk-matrix.md) | Go, Node/TypeScript, Python — wire-identical |
| [The plugin catalog](catalog.md) | What exists, and how to get listed |

## Source material

`inflow-plugin-sdk` (Go, reference) · `node-plugin-sdk` · `py-plugin-sdk` ·
`plugin-catalog/docs/` · `inflow-js/packages/plugin-form-builder`.

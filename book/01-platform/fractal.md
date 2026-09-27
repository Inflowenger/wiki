# Fractal — the execution engine

> **Status: outlined.**
>
> **Fractal is closed source.** This chapter documents its behaviour and guarantees, not
> its implementation.

## What Fractal is

The processor. Given a `ProcessRequest` — start nodes, flow id, context id, timeouts — it
walks the compiled node map, executing each node according to its type, and asking your
backend for whatever it does not already have.

It is headless and stateless with respect to your domain. It never touches your database.
Several instances can run at once; they do not coordinate with each other, only with Infra.

## Sections planned

**1. What "walking the graph" involves.** Queue a start node's successors (or the named
nodes on a resume), execute, resolve outgoing transitions, repeat. Where `Depends` forces a
wait. Where fan-out becomes genuine parallelism.

**2. Execution per node type.** Run JS/OPA in-engine; evaluate a rule and filter edges;
publish to an extrinsic subject and await reply; hand off to a plugin and manage the job
lifecycle; jump into another flow and return; do nothing.

**3. Durability, and what it actually promises.** Node results live in the context
document, which your backend persists. The traversal snapshot (`_sched`) in the context
header records completed generations and join watermarks. This is the mechanism — stated
honestly, including what it does *not* promise (it is not exactly-once; a plugin action may
be re-entered, which is why `_registry` carries the previous `jobId`).

**4. The resume gate.** Why a resume checks a structural flow signature and falls back to a
blank continue on drift — an edited flow cannot resume into a stale plan. The snapshot's
full shape, and how a backend stores it per-pid and hands it back on the resume request, is
in [The wire](../02-fusion/the-wire.md#5-the-traversal-snapshot--_sched).

**5. Safety rails.** `RequestTimeOut` (per NATS request, default 5s), `ExecuteTimeOut`
(whole process, default 3600s), `ProcessNodeLimit` (nodes visited, default 500). What each
one protects against, and what a hit looks like in the event stream.

**6. The event stream as the engine's only public output.** Cross-reference
[Observing a run](../02-fusion/observing-a-run.md).

**7. Registration and tags.** `FRACTAL_TAGS`, `FRACTAL_NAME`, the `rs` header on every
event, and how a backend pins to a named instance.

**8. Why "Fractal."** A node can itself be an embedded flow, so the same structure holds at
every scale.

## Source material

`inflow-fusion/docs/architecture.md`, `inflow-fusion/docs/infra.md`,
`inflow-fusion/docs/logs.md`, `Inflowenger/getting-started` README.

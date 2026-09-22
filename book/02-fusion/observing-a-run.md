# Observing a run

> A graph you cannot watch execute is not a glass box. This is the stream that makes it
> one — and the package that turns it back into movement on your canvas.

The engine emits a **process event stream**: one message per meaningful thing that happens
while a flow executes. It is the only way an outside observer learns that a process
started, which nodes ran, **which edges were actually taken**, and whether the process
finished.

It is a contract between the engine (which publishes) and everything else — your backend,
an inspector, a parser library, your own UI (which consume).

> **Schema version 1.** Everything below describes `v: 1`, which is the only format there
> is. A message that does not carry `v: 1` is not an event under this contract; consumers
> reject it rather than interpret it.

---

## Why it is shaped this way

Two consumer needs drive the entire design:

1. **Flow movement** — render a process walking the graph: which node is running, which
   edge it left by, where it went. This is why `edge.select` exists, and why every
   node-scoped event carries a flow id.
2. **Completion** — know that a process ended, and whether it ended well. This is why
   `proc.finish` carries a status and why nothing may follow it.

Everything else is diagnostics.

One decision keeps the stream small: **the consumer already has the graph.** It loaded the
flow definition in order to render it. So events never re-transmit the graph — they
reference nodes and edges by identity, and the consumer joins against the definition it
already holds.

---

## Transport

Every event arrives on **one subject: the registration's event log.**

That subject is **per-registration, not fixed.** A portal may define its own in
`subscribe_prefix`, and the engine publishes there; `inflow.event.log` is only the fallback
for a portal that defines none.

> A consumer that hardcodes the fallback does not *error* against a portal that set a
> subject — it silently receives nothing. **Resolve the subject from the registration.**
> (`subscribe_prefix` is misnamed: it is the whole subject, not a prefix.)

Every message carries a NATS header `rs` — the name the publishing Fractal instance
registered itself with on Infra. That is how a consumer tells apart instances sharing a
subject.

**The stream is shared across every process on that engine.** It is not a per-process
channel. Consumers must demultiplex by `pid`.

---

## The envelope

```json
{
  "v": 1,
  "pid": "33e2f7df-3d47-44c9-a670-38a5d334f238",
  "seq": 42,
  "ts": 1784112492271,
  "kind": "node.enter",
  "level": "info",
  "src": "rt",
  "flow": "flow:22",
  "node": "10",
  "detail": { "type": "ruleNodeType", "title": "Contract" }
}
```

| Field | Type | Required | Meaning |
| --- | --- | --- | --- |
| `v` | int | always | Schema version. `1`. |
| `pid` | string | always | Process id. The correlation key for the whole stream. |
| `seq` | int | always | Monotonic counter **per `pid`**, from `0`, no gaps. The ordering key. |
| `ts` | int64 | always | Wall clock, Unix ms. **Display only — never order by this.** |
| `kind` | string | always | What happened. |
| `level` | string | always | Severity only: `debug` / `info` / `warn` / `error`. |
| `src` | string | always | Who emitted it. |
| `flow` | string | node-scoped | Flow id owning `node`. |
| `node` | string | node-scoped | Node id — **unique only within `flow`**. |
| `detail` | object | always | Typed per `kind`. Real JSON, never a formatted string. |

### Three rules consumers get wrong

**1. A node is `(flow, node)`, never `node` alone.** Node ids are unique only within their
flow. A single process routinely spans several flows — a GoTo jumps into another flow and
back — so within one `pid`, id `9` can be a code node in `flow:22` *and* a contract node in
`flow:33`, both live at once when flows run in parallel. **Key all state on
`${flow}:${node}`.**

**2. Order by `seq`, never by `ts`.** `ts` has millisecond resolution and the engine emits
many events per millisecond, so timestamps collide constantly. `seq` is the only ordering
authority — and because it is gapless per `pid`, it also lets you detect dropped messages
and know a replay is complete.

**3. `level` is severity, nothing else.** It never encodes what kind of event this is. A
consumer filtering to `level >= warn` must still receive a coherent (if sparse) stream — it
must not lose lifecycle events because they were classed `info`. `error` means *this
operation failed*; normal completion is never `error`.

### `src` — who emitted it

| `src` | Meaning |
| --- | --- |
| `rt` | the runtime itself — lifecycle, routing |
| `js` | JS compiler / user JS code |
| `rego` | OPA compiler / user policy |
| `contract` | rule evaluation |
| `plugin:<title>` | a Plugin node, e.g. `plugin:RpcFn` |
| `extrinsic:<title>` | an Extrinsic node |

`src` is an *actor*, not a node title. Node identity lives in `flow`/`node`.

---

## Event kinds

| `kind` | Scope | `detail` |
| --- | --- | --- |
| `proc.start` | process | `{ flow, node, ctxId? }` — the entry point |
| `proc.finish` | process | `{ status, durationMs, error? }` |
| `node.enter` | node | `{ type, title, attempt }` |
| `node.exit` | node | `{ status, durationMs, error? }` |
| `edge.select` | node | `{ taken: EdgeRef[], pruned: EdgeRef[], reason? }` |
| `flow.jump` | node | `{ to: NodeRef, ret: NodeRef? }` |
| `log` | either | `{ msg, ...fields }` — free-form |

```
NodeRef  = { flow: string, node: string }
EdgeRef  = { flow: string, node: string, edgeId: string, tags: string[] }
```

Exactly one `proc.start` and one `proc.finish` per `pid`. `proc.start` is always `seq: 0`,
and **`proc.finish` is always the last event for that `pid`** — nothing may follow it.

### `edge.select` — the one that matters

This is the event that makes the platform's central promise auditable.

`edgeId` matches the edge id in the flow definition, so a consumer lights up the exact edge
it is already rendering. `taken` and `pruned` together say what the runtime *considered*,
not just what it did. And `reason` carries the **select reason** — the tags that did the
selecting.

So a run does not merely record *which* edge fired. It records **why**, in terms of the
tags a rule, a service reply or a plugin emitted. Cross-reference
[Tag routing](tag-routing.md): the decision was dynamic, the possibilities were drawn in
advance, and the event stream is the receipt for both.

---

## Consuming it: `@inflowenger/flow-trace`

You do not have to write the state machine that turns this stream into UI.
**`@inflowenger/flow-trace`** is one of two packages in
[`inflow-js`](https://github.com/Inflowenger/inflow-js), and it does exactly this job:

> Turns the Inflow process event stream into flow movement and completion: where the
> process is, which edges it took, whether it finished.

```sh
pnpm add @inflowenger/flow-trace
```

It has **no dependencies and no framework.** It runs in a browser, in Node, or in a test.
If your product is React, or has no UI at all, it is still yours to use — the sibling
package (`plugin-form-builder`) is the Vue-only one.

It handles the parts of the contract a clean run never exercises: out-of-order delivery,
a real `seq` gap, and traffic on the subject that is not an event at all. The repo's
`lab/` app has a **Run Replay** page that feeds a captured run through a tracker one
message at a time, with controls to reorder, drop from, and pollute the stream — which is
a faster way to understand the contract's edges than reading about them.

The other package in `inflow-js` is covered in
[Part III — Forms](../03-plugins/forms-and-ui.md).

---

## Consumer rules, condensed

- **Demultiplex by `pid`** — the subject carries every process on that engine.
- **Resolve the subject from the registration**, not from the fallback constant.
- **Key node state on `${flow}:${node}`.**
- **Order by `seq`.** Detect gaps; treat a gap as dropped messages.
- **Stop at `proc.finish`.** Nothing follows it.
- **Reject anything without `v: 1`.**
- **Join against the flow definition you already hold** — the stream will not send it.

---

## What this buys the product

| Capability | The event that provides it |
| --- | --- |
| Live "the process is here" highlighting | `node.enter` / `node.exit` |
| Animating the edge actually taken | `edge.select.taken` |
| Explaining *why* that edge | `edge.select.reason` |
| Showing a sub-flow being entered and returned from | `flow.jump` |
| A duration per node, for a slow-step view | `node.exit.durationMs` |
| Run status, error and total duration | `proc.finish` |
| A per-node diagnostic log the author can read | `log`, with `src` |

This is what "glass box" means concretely. Not that the graph is visible — a picture of a
graph is easy — but that **the executed path, and the reason for it, is recoverable after
the fact** for every run.

---

## Next

Part II is complete. Continue to:

- **[Part III — The Plugin Layer](../03-plugins/)** — the one primitive that never
  compiles away.

**Source material:** `inflow-fusion/docs/logs.md` (the normative contract),
[`inflow-js`](https://github.com/Inflowenger/inflow-js) `packages/flow-trace`.

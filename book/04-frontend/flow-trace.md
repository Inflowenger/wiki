# Watching a run — `@inflowenger/flow-trace`

> The engine publishes an event per meaningful thing that happens while a flow runs.
> This is the consumer half of that contract: feed it the stream, get movement and
> completion.

[Observing a run](../02-fusion/observing-a-run.md) specifies the wire format — the
envelope, the seven kinds, the ordering rules. This chapter is the library that implements
the consumer side of it so you do not have to.

```sh
pnpm add @inflowenger/flow-trace
```

**Zero dependencies. No framework.** Browser, Node, or a test.

---

## The whole integration

```ts
import { createFlowTracker } from '@inflowenger/flow-trace'

const tracker = createFlowTracker({ pid })

tracker.on('move',       ({ from, to, edgeId, tags }) => highlightEdge(edgeId))
tracker.on('node:enter', ({ node })                   => markRunning(node.key))
tracker.on('node:exit',  ({ node, status })           => markSettled(node.key, status))
tracker.on('finish',     ({ status, durationMs })     => showResult(status, durationMs))

socket.on('message', (raw) => tracker.ingest(raw))
```

That is the integration. What the tracker absorbs on your behalf is everything between
`socket.on` and those four callbacks: **demultiplexing by `pid`, reordering by `seq`,
validating the envelope, detecting real gaps, and folding events into state.**

`ingest` takes an object *or* a JSON string and **never throws** — anything unusable
arrives on the `skip` event instead of breaking your socket handler. That matters more
than it reads: a socket handler that throws on a malformed frame takes the connection with
it, and the one message you cannot control the shape of is the one a third-party plugin
logged.

For a run you already hold in full:

```ts
import { replay } from '@inflowenger/flow-trace'

const state = replay(lines).get(pid)
// state.status === 'completed' | 'failed' | 'stopped' | 'running'
```

`replay` holds out-of-order events indefinitely and flushes at the end, so a batch is
ordered perfectly however it arrives.

---

## The events it emits

| Event | Fires when | Carries |
| --- | --- | --- |
| `start` | The process begins | `entry` |
| `move` | Control crosses an edge — **the one to use for movement** | `from`, `to`, `edgeId`, `edgeKey`, `tags`, `reason` |
| `node:enter` / `node:exit` | A node starts / settles | `node` (the full `NodeState`), `status`, `error` |
| `jump` | A GoTo transfers into another flow | `to`, `ret` |
| `log` | Diagnostics from user code, a plugin, or the runtime | `level`, `src`, `msg`, `fields` |
| `finish` | The process ends | `status`, `durationMs`, `error` |
| `gap` | Events were lost | `from`, `count` |
| `skip` | A message wasn't a usable v1 event | `reason`, `input` |
| `event` | Any accepted event, **after** it has been applied | the raw `ProcEvent` |

Note the shape of the translation. On the wire the kind is `edge.select`, carrying `taken`
*and* `pruned`. The tracker emits **one `move` per taken edge** — because movement is what
a canvas animates — and folds the pruned ones into edge state, where a path view can grey
them out. `log` is the only kind that passes through essentially as-is, and it has a
taxonomy of its own: see **[The log taxonomy](log-categories.md)**.

`event` is the escape hatch. Subscribe to it when you want a raw log view, or when you need
a kind the tracker does not model yet — you still get ordering and validation for free.

---

## The state it accumulates

```ts
const state = tracker.state(pid)
```

Plain objects — no reactivity, no classes, no getters. Wrap them in `reactive()`, put them
in a Redux store, diff them, or serialise them as-is.

```ts
interface ProcessState {
  pid: string
  status: 'running' | 'completed' | 'failed' | 'stopped'
  entry?: NodeRef             // where the process began
  startedAt?: number
  finishedAt?: number
  durationMs?: number
  error?: string
  nodes: Record<string, NodeState>   // keyed `flow:node`
  edges: Record<string, EdgeState>
  lastSeq: number             // highest seq applied
  pending: number             // events held back waiting for an earlier seq
}
```

| `NodeState` | |
| --- | --- |
| `key` | `flow:node` — the key it is stored under |
| `flow`, `node` | the composite identity, split |
| `type`, `title` | from `node.enter`, when it arrived |
| `status` | `pending` → `running` → `ok` \| `error` |
| `attempts` | how many times this node has run. **`> 1` means a loop or a GoTo re-entry** |
| `enteredAt`, `exitedAt`, `durationMs`, `error` | timings and failure |

`pending` is the state worth knowing about: an inbound edge was taken *toward* this node,
but it has not reported entering yet. It is the difference between "about to run" and
"running", and it is what lets a canvas show control in flight rather than teleporting.

| `EdgeState` | |
| --- | --- |
| `key`, `edgeId` | `edgeId` is the graph's own — empty for edges the runtime synthesised (a GoTo jump) |
| `from`, `to`, `tags` | endpoints, and the tags the runtime matched |
| `taken` | whether control has **ever** crossed this edge. **Sticky** |
| `takenCount` | how many times. `> 1` means a loop; `0` means *evaluated but never followed* |
| `at` | when the most recent decision about this edge was reported |

---

## Five things worth knowing

These are the parts every consumer gets wrong on the first attempt, and the reason this
library exists rather than a page of instructions.

### 1. A node is `flow:node`, never a bare id

Node ids are unique only **within** a flow, and one process spans several via GoTo. In a
real run, `flow:22`'s node `9` is a code node while `flow:33`'s node `9` is a contract —
both live under the same `pid`, both at once when flows run in parallel.

```ts
import { nodeKey } from '@inflowenger/flow-trace'
const key = nodeKey(flow, node)   // or nodeKey(ref)
```

Keying on the raw id **silently merges unrelated nodes**. There is no error; the run just
draws wrong.

### 2. Order comes from `seq`, never `ts`

The engine emits many events per millisecond, so timestamps collide constantly and cannot
sort anything. `seq` is monotonic per `pid` and **gapless at the producer**. The tracker
reorders by it internally, holding out-of-order events until the sequence is contiguous.

### 3. Loops are normal, and state is bounded by the graph

A node can be processed thousands of times. `nodes` and `edges` hold **one entry per
identity**; the history lives in counters — `attempts` and `takenCount`. Nothing
accumulates per pass, so a long-running flow does not leak memory.

**Events still fire on every pass**, which is exactly what animation needs. State is
bounded; the event stream is not.

### 4. `EdgeState.taken` is sticky

A loop can take an edge on one pass and prune it on the next. Once control has crossed an
edge, `taken` stays `true` — the flow really did go that way, and a path view must keep
showing it. Use `takenCount` for how often, and `takenCount === 0` for *"evaluated but never
followed"*, which is how you grey out the rejected side of a branch.

### 5. The stream carries ids, not names

Titles live in the editor. The only one on the wire is `node.enter`'s — and a pruned branch
never enters, so it never gets one. Resolve ids against the graph you already have.

---

## Naming what the stream refers to

The engine says `n_mrv19bz0aownno`; only the saved graph knows that is *upsert*.

Embedding titles in events would ship the editor's vocabulary through the runtime and go
stale on the next rename — so events carry ids, and naming happens where the graph is.
Two exports close that gap:

```ts
import { describeEvent, resolveRefs } from '@inflowenger/flow-trace'

const graph: GraphIndex = {
  node: (flow, id) => store.flows[flow!]?.nodes[id],   // { id, title, type? }
  edge: (flow, id) => store.flows[flow!]?.edges[id],   // { id, source, target, tags }
}

for (const part of resolveRefs(describeEvent(event), graph, { flow: event.flow })) {
  // part.label — the title, the edge's tags, or the id when nothing named it
  // part.known — false for text and unresolved ids, so you can fade them
}
```

- **`describeEvent`** turns any event into one readable line, ids intact. Every consumer
  that shows a stream needs this, and writing it twice means two vocabularies for the same
  run.
- **`resolveRefs`** swaps those ids for what your graph draws. It works on **any** text,
  not just `describeEvent`'s — a message a plugin or a JS node wrote resolves the same way.
- **`parseRefs`** is the split on its own; **`refText`** writes a reference back — bare
  inside its flow, `flow_…/n_…` when it leaves one, which is how a GoTo target stays
  resolvable.

`GraphIndex` is an interface over *whatever you already load*. In FloMorphic the Pinia
store that holds flow graphs simply **is** a `GraphIndex` — it is handed straight to
`resolveRefs` with no adapter (`flomorphic-wapp/src/stores/flowGraphs.ts`).

> A lookup returning `undefined` is **normal**: the flow may still be loading, or the node
> may since have been deleted. The id is then shown as-is, with `known: false` so you can
> fade it.

---

## Gaps, reordering and attaching mid-run

The stream is gapless at the producer, so a hole means one of two things: a message was
genuinely lost, or **you attached mid-run and missed the history**. Both are ordinary.

```ts
createFlowTracker({ pid, reorderWindow: 16 })   // 16 is the default
```

The tracker holds out-of-order events up to `reorderWindow`, then reports a `gap` and moves
on rather than stalling forever. The engine stamps `seq` synchronously but publishes
concurrently, so small reorderings are normal; a lasting hole is not.

- **`flush()`** drains a hole immediately — call it when a stream has ended.
- **`replay()`** uses `Infinity` and flushes at the end, which is why a stored run orders
  perfectly.
- Raise the window to tolerate more reordering at the cost of latency.

> **Filter by `pid`.** The subject carries **every process on that engine**, not a channel
> per run. `createFlowTracker({ pid })` accepts one pid or an array; without it, a tracker
> accumulates state for other people's runs too.

---

## Seeing it before you build on it

The `inflow-js` repo ships a **lab** — a small Vue app that drives both packages by hand,
with no backend, no engine and no inspector.

```sh
pnpm install
pnpm -r build
pnpm --filter @inflowenger/lab dev
```

Its **Run Replay** page feeds a captured engine run to a tracker one message at a time,
with node keys, edge counters and the emitted feed side by side. Its toggles **reorder the
stream, drop an event, and mix in traffic that is not an event at all** — the three cases
you would otherwise only meet in production.

The library's own tests run against `test/fixtures/real-run.jsonl`, an actual captured run
covering a contract branch, a plugin failure, a GoTo across flows, and a loop. The repo's
guidance is to extend that fixture rather than hand-write events, because hand-written
events drift from what the engine really emits — which is a good rule for your tests too.

---

## Next

- **[The log taxonomy](log-categories.md)** — the `log` event is the one kind with a
  classification system of its own, and it is more complete than most people expect.
- **[Building a process product](build-a-process-product.md)** — this library in place, in
  a real app.

**Source material:** [`inflow-js`](https://github.com/Inflowenger/inflow-js)
`packages/flow-trace` (`README.md`, `src/types.ts`, `src/tracker.ts`, `src/refs.ts`);
`inflow-fusion/docs/logs.md` (the normative producer contract);
`flomorphic-wapp/src/stores/flowLogs.ts`, `src/lib/runState.ts`.

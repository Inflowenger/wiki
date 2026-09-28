# The log taxonomy

> *"How many kinds of log are there, and will I be able to tell them apart?"*
>
> The answer is: **four orthogonal axes, seven categories, and a typed field set for every
> one of them.** All of it is exported from `@inflowenger/flow-trace` as TypeScript types.
> There is nothing to guess and nothing to parse out of a string.

This is the chapter to read before deciding that runtime observability is the part you will
have to build. Six of the seven event kinds are structural — a process started, a node
entered, an edge was taken. The seventh, `log`, is the open one, and the fear it provokes
is reasonable: *free-form diagnostics* usually means an unclassifiable soup of strings from
user code, plugins and the runtime, arriving on the same channel, with a frontend left to
regex its way to meaning.

That is not what arrives. `log` carries a **`category`**, and the set is closed and
documented.

---

## Four axes, not one

The single most useful thing to internalise is that an event's *kind*, its *severity*, its
*author* and its *sub-classification* are four independent fields. Conflating any two of
them is the source of every log-view bug.

| Axis | Field | Values | Means |
| --- | --- | --- | --- |
| **What happened** | `kind` | `proc.start` `proc.finish` `node.enter` `node.exit` `edge.select` `flow.jump` `log` | Structure. Orthogonal to severity. |
| **How bad** | `level` | `debug` `info` `warn` `error` | **Severity, and nothing else.** |
| **Who said it** | `src` | `rt` `js` `rego` `contract` `plugin:<title>` `extrinsic:<title>` | An *actor*, not a node title. |
| **What sort of log** | `detail.category` | the seven below, or absent | Sub-class of a `log` event. |

> **`level` never encodes what kind of event this is.** A consumer filtering to
> `level >= warn` must still receive a coherent stream — it must not lose lifecycle events
> because they were classed `info`. Equally, `error` means *this operation failed*; normal
> completion is never `error`.

A log with **no** `category` is an ordinary diagnostic: something a flow author's JS wrote,
a plugin printed, or the runtime noted. Those are the ones you render as a plain line.
Everything else has a shape.

---

## The seven categories

Each is a `LogCategory` value with a matching exported interface describing exactly which
fields ride with it.

| `category` | Level | Fields | What it is |
| --- | --- | --- | --- |
| `progress` | `info` | `ProgressLogFields` | A sub-100% frame from a running plugin job |
| `protocol` | `debug`/`info` | `ProtocolLogFields` | An `inflowv1` transport trace |
| `dep.wait` | `info` / `warn` | `DepWaitFields` | A join node parking on its inbound branches |
| `dep.ready` | `info` | `DepReadyFields` | That join released — every dependency arrived |
| `scope.fanout` | `info` | `ScopeFanoutFields` | A node whose `scope` matched several locations |
| `resume` | `info` / `warn` | `ResumeFields` | A run continuing an earlier one |
| `stop.on.error` | `warn` | `StopOnErrorFields` | A branch cut short because a node did not deliver |

### `progress` — a job reporting as it goes

```ts
interface ProgressLogFields {
  category: 'progress'
  percent: number            // [1, 99] — 100% is node completion, not a frame
  frame: { title: string; content: string; meta?: Record<string, unknown> }
  details?: Record<string, unknown>   // an optional partial payload
}
```

This is what lets a long-running node show a pie chart rather than a spinner. `percent` is
deliberately capped at 99: **100% is `node.exit`, not a frame**, so a UI that draws the
frame and the exit from different events never renders a completed node that is still
running.

`frame` is the human-readable content — render it beside the chart. `meta` is a reserved
open bag for frontend-effective extras (an `items` list, say) that a specific plugin and a
specific UI agree on.

### `protocol` — the plugin handshake

```ts
interface ProtocolLogFields {
  category: 'protocol'
  proto: string
  msg: string
  subject?: string
  jobId?: string
}
```

Init and request subjects, job-id assignment, subscriptions: diagnostics about the
plugin↔core NATS handshake, **not task output**. This is the category most consumers hide
by default and reveal behind a "show transport" toggle — which is precisely why it is
tagged rather than mixed into the ordinary log line.

### `dep.wait` / `dep.ready` — joins, made visible

```ts
interface DepWaitFields {
  category: 'dep.wait'
  depends: string[]     // the whole join
  pending: string[]     // the part still outstanding
  waitedMs?: number     // present only on the give-up variant
}

interface DepReadyFields {
  category: 'dep.ready'
  depends: string[]
  waitedMs: number
}
```

A join node parks until the branches it depends on arrive. These two events are how a UI
shows *why* nothing is happening — the single most common support question about a
parallel flow. What a join *is* — the `Depends` field, present on every node — is
[Part II: Waiting](../02-fusion/waiting.md).

Three readings matter:

- A `dep.wait` **with** a matching `dep.ready` for the same node is an ordinary join.
- A `dep.wait` with **no** matching `dep.ready` is a join that never completed.
- The variant carrying **`waitedMs`** is the runtime giving up, because the process ended
  with branches that will now never arrive. It is emitted at **`warn`**.

A join whose dependencies were already in when it was dequeued emits **neither** event —
absence is not a bug.

### `scope.fanout` — the loop with no loop

```ts
interface ScopeFanoutFields {
  category: 'scope.fanout'
  scope: string        // as written, e.g. `$.orders[*]`
  count: number
  locations: string[]
}
```

A node whose `scope` matched more than one location runs **once per location,
sequentially** — but the engine still emits a **single `node.enter` / `node.exit` pair
around the whole set**. This line is therefore the *only* thing that makes that loop
visible. A UI that reports "1 run" for a node that processed 300 orders is a UI that
ignored this category.

> `locations` is **capped by the producer**, so a large fan-out lists a sample rather than
> every element. **Trust `count`, not `locations.length`.**

### `resume` — a run continuing an earlier one

```ts
interface ResumeFields {
  category: 'resume'
  seededNodes?: number
  joinWatermarks?: number
}
```

The traversal snapshot the previous run left in the context header has been reloaded, so a
join downstream of the resume point sees its already-completed dependencies instead of
locking forever. What produces a continuation in the first place — a delay, a human task, a
webhook — is [park-and-resume](../02-fusion/waiting.md#part-2--continue-after-a-delay-is-a-pattern-not-a-primitive).

- **Applied** (`info`) carries `seededNodes` and `joinWatermarks`.
- **Skipped** (`warn`) carries **neither** — only `msg`. A resume was requested but the
  snapshot was absent, or the flow drifted since it was taken, so the run continues with
  blank state.

That distinction is worth surfacing: a skipped resume is not an error, but it explains a
run that redid work you expected it to skip.

### `stop.on.error` — a branch ended by decision

```ts
interface StopOnErrorFields {
  category: 'stop.on.error'
  pruned: number     // how many outgoing edges were dropped
  code: number
  cause: string
}
```

A node did not deliver and the run was started with `stop_on_error`, so its outgoing edges
were pruned and nothing continues past it.

**The process is not stopped.** Every other branch runs to its end. This is the only record
that a branch ended *by decision* rather than by reaching its last node, and it is emitted
at `warn`. `code` and `cause` name the error that did it — the same error is the event
immediately before this one, but the line is written to read on its own, out of a filter or
a week later.

---

## The error taxonomy, which is not a category

Orthogonal again, and the one most worth getting right. An **error-level** `log` the run
*recorded* carries two extra fields:

```ts
interface ErrorLogFields {
  msg: string
  code: number
  kind: 'node' | 'system'
}
```

Recorded means: entered in the run's ledger in the context header (`_errors`) **as well as**
published on the stream — so a caller who was not watching can still read back what the run
hit.

`kind` is the one to render:

| `kind` | Whose error | What the user can do |
| --- | --- | --- |
| `node` | The flow author's own — their JS or Rego, the node data they wrote, something their node called that did not deliver | **Act on it.** Fix the flow. |
| `system` | The platform's | Nothing in the flow caused it; nothing in the flow mends it. |

> An error-level log carrying **neither** field is a diagnostic somebody logged loudly, not
> one of the run's own errors. **A reader who conflates the two will send flow authors
> after outages they cannot fix** — which is the single highest-value distinction in this
> whole chapter.

And one consequence that surprises people: **a flow does not stop for a node error.** It is
handled and the run carries on, so a run that emitted several of these can still finish
`completed`. A UI that infers failure from the presence of an error log will be wrong
routinely. Infer it from `proc.finish.status`.

---

## What this means for your log view

You can build a genuinely useful runtime log view without inventing a taxonomy, because
the taxonomy arrived with the stream:

| The view wants | Read |
| --- | --- |
| A badge per line | `category`, falling back to `kind` |
| A severity colour | `level` — alone |
| An author chip | `src` |
| A progress ring on a node | `progress.percent` + `frame` |
| "Waiting on 2 of 3 branches" | `dep.wait.pending` vs `depends` |
| "Ran 300 times" on a single-entry node | `scope.fanout.count` |
| A collapsible "transport" section | `category === 'protocol'` |
| "Fix this" vs "tell your admin" | `ErrorLogFields.kind` |
| Run failed | `proc.finish.status` — **never** the presence of an error log |

FloMorphic's log drawer is exactly this table (`flomorphic-wapp/src/stores/flowLogs.ts`):
one `FlowLogMessage` per accepted event, carrying `level`, `category`, `src`, `flow`,
`nodeId` and `nodeTitle` alongside the raw event, with the display line produced by
`describeEvent` and the ids resolved to titles by `resolveRefs`.

```ts
export interface FlowLogMessage {
  id: string
  timestamp: number
  level: LogLevel
  message: string
  event?: ProcEvent
  pid?: string
  seq?: number
  kind?: string
  /** Sub-kind of a `log` event (progress / protocol / dep.*), drives the badge. */
  category?: LogCategory
  src?: string
  flow?: string
  nodeId?: string
  nodeTitle?: string
}
```

One more thing that store does, and which any long-running product needs: **it caps
retained lines** (`MAX_MESSAGES = 5000`), because a looping flow emits without bound. The
tracker's *state* is bounded by the graph; your *display buffer* is not, and that is yours
to bound.

---

## Next

- **[Dynamic forms](plugin-form-builder.md)** — the other package, and the other half of
  what the browser gets for free.

**Source material:** `inflow-js/packages/flow-trace/src/types.ts` (every interface quoted
here is exported); `inflow-fusion/docs/logs.md`;
`flomorphic-wapp/src/stores/flowLogs.ts`.

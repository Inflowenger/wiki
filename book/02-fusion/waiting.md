# Waiting — joins and delays

> Two mechanisms, one theme: a run that has reached a node and **does not continue yet**.
>
> One waits for **other branches** (`Depends`). One waits for **the clock**
> (*Continue After*). Neither is a node type, and that is the point of this chapter.

Every workflow product eventually ships a *Wait for All* card and a *Delay* card, and
almost every one of them implements the two as special node types with bespoke engine
support. Inflowenger has neither. It has **a field on every node**, and **a pattern anyone
can write** — and once you see what each one actually is, you can build your own variants
without asking the platform for anything.

---

## Part 1 — `Depends`: a join is a field, not a node

### It lives on the base node

Here is the whole mechanism, from `inflow-fusion/models/flow.go`:

```go
type Node struct {
    ID        string         `json:"uuid"`
    Type      NodeType       `json:"type"`
    Title     string         `json:"title"`
    Key       string         `json:"key"`
    Scope     string         `json:"scope"`
    Code      *CodeRule      `json:"code,omitempty"`
    GoTo      *GoToRule      `json:"goto,omitempty"`
    Extrinsic *ExtrinsicRule `json:"extrinsic,omitempty"`
    Plugin    *PluginRule    `json:"plugin,omitempty"`
    Contract  *ContractRule  `json:"contract,omitempty"`
    Meta      map[string]any `json:"meta"`
    Tags      string         `json:"tags"`
    Next      []Next         `json:"next"`
    Depends   []string       `json:"depends"` // wait for all these inbound nodes first
}
```

`Depends` sits on the **base struct**, beside `Next` and `Scope` — *not* inside
`VoidRule`, because there is no such thing. It is not reachable only from one primitive.
It is reachable from **all six**.

> **Any node can be a join.** A Plugin node can wait for three branches and *then* call
> Jira. A Code node can wait for two branches and *then* merge their outputs. A Contract
> can wait, then decide. You do not need a barrier node in front of them unless you want
> one.

This is the same design move as `Scope` and `Tags`: put the capability on the envelope, and
every primitive inherits it. Six primitives × universal fields is why the palette never
needs a fourth structural node.

### The semantics, exactly

- `Depends` holds **node ids**, and the node does not run until **every** one of them has
  finished. `Promise.all` for a graph.
- It is **completion**, not data. A join does not inspect whether its dependencies produced
  anything; it waits for them to be *done*. (Their results are already in the shared
  context — see [the computational model](../01-platform/computational-model.md): nothing
  travels along edges, so a join has no tuples to merge.)
- Arrival is **not** the trigger. An inbound edge firing does not release a join; the
  dependency list does.
- A node with `Depends` **empty** runs whenever control reaches it. That is the default,
  and it is what every ordinary node has.

### The distinction that causes every bug: fan-in ≠ join

```
        ┌──► B ──┐
   A ───┤        ├──► D
        └──► C ──┘
```

If `D` has **no** `Depends`, this graph **runs `D` twice** — once when `B` finishes, once
when `C` does. Inbound edges *do not merge*. Two arrows into a node is two arrivals, not
one rendezvous.

If `D` lists `["B","C"]` in `Depends`, it runs **once**, after both.

Drawing the two cases identically is the single most common authoring mistake in any
graph product, and it is worth surfacing in your UI rather than leaving to discovery.
FloMorphic's designer validates exactly three shapes of it
(`flomorphic-api/designer/patch.go`):

| Shape | Level | What it tells the author |
| --- | --- | --- |
| N branches arrive at a node that is **not** a join | `warn` | *"N branches arrive at X, so it runs N times — inbound edges do not merge. If you meant 'when all are done', put a Wait for All in front of it and wire the branches into that."* |
| A join with **fewer than two** inbound edges | `warn` | *"It has nothing to join and is a no-op."* |
| A join waiting on a node that **itself runs more than once** | **`error`** | *"A Wait for All needs every branch it waits on to run the same number of times, or it waits forever and the run stalls until it times out."* |

The third is an `error` rather than a `warn` because it does not misbehave — it **hangs**,
and then fails at the process timeout, hours later, with nothing in the log that obviously
points back at the graph.

> **Scope cardinality is not branching.** A many-valued `scope` (`$.orders[*]`) is a
> **queue inside one node**, not a fan-out: the node runs once per location, sequentially,
> and its single outbound edge fires once after every pass. Putting a join after it is a
> no-op that adds a hop. The only visible record of that inner loop is the
> [`scope.fanout` log category](../04-frontend/log-categories.md#scopefanout--the-loop-with-no-loop).

### Where a product's *Wait for All* card actually comes from

FloMorphic ships a palette card called **Wait for All**. Here is everything behind it.

**1. It lowers to a Void node.** In `inflow/compiler.go`:

```go
const NODE_PROMISEALL = "promissall" // -> void (depends on all inbound nodes)

switch vfn.Type {
case NODE_START, NODE_PROMISEALL, NODE_VOID:
    // Start marker, the fan-in join and void all lower to a Void node.
    node.Type = inflowModels.VoidNodeType
```

Three different palette cards, one primitive. The card is *product vocabulary*; Void +
`Depends` is what exists at runtime.

**2. Its `Depends` is filled by a compiler post-pass, not by the hook.** This is the part
worth stealing:

```go
// Post-pass — edge-derived wiring the per-node hook cannot see (it only gets
// one node): a promissall (fan-in) waits on every inbound node.
for _, n := range f.ViewFlow.Nodes {
    cn, ok := l[n.ID]
    if !ok || cn == nil { continue }
    if n.Type == NODE_PROMISEALL {
        cn.Depends = inboundSources(f.ViewFlow, n.ID)
    }
}

// inboundSources returns the ids of every node with an edge into nodeId.
func inboundSources(flow compiler.VueFlow, nodeId string) []string {
    out := []string{}
    for _, e := range flow.Edges {
        if e.Target == nodeId { out = append(out, e.Source) }
    }
    return out
}
```

**The per-node hook cannot do this.** [The hook](the-compiler-seam.md) is handed one node
at a time and has no view of the graph's edges, by design — that is what keeps it a
`switch` instead of a graph algorithm. Anything **edge-derived** belongs in a post-pass
over the compiled map, where you have both the source graph and the result.

So the author never types a dependency list. They drop a card, wire branches into it, and
the compiler derives `Depends` from the edges they drew. **That is the whole feature** —
about fifteen lines, no engine change, no new primitive.

> **Your product can be cleverer, for free.** Derive `Depends` from inbound edges on *every*
> node and let the author tick a "wait for all inbound" box on the node itself; or let them
> pick a subset, so a node waits for two of its three branches. The field takes an arbitrary
> id list — the barrier-node shape is a UI convention, not a constraint.

### What a join emits

A join is one of the few runtime behaviours that is invisible on the canvas — nothing is
happening, on purpose — so it is reported explicitly on the event stream:

| Event | Means |
| --- | --- |
| `dep.wait` with `depends` + `pending` | Parked. *These* are the branches, *these* are still outstanding |
| `dep.ready` with `depends` + `waitedMs` | Released. Every dependency arrived |
| `dep.wait` carrying **`waitedMs`**, at `warn` | The runtime **gave up**: the process ended with branches that will never arrive |
| *neither* | A join whose dependencies were already in when it was dequeued. Not a bug |

That is enough to render *"waiting on 2 of 3 branches"* in a UI, which is the answer to the
most common support question about any parallel flow. Full field definitions:
[The log taxonomy](../04-frontend/log-categories.md#depwait--depready--joins-made-visible).

### Joins and resumption

A join is also the reason a resumed run needs seeded state. A process that parks and
restarts later is a *new* process; if it re-enters a join whose other branches completed in
the **previous** run, those branches will never be re-dispatched, and the join blocks until
the process timeout.

The engine's traversal snapshot carries **join watermarks** (`joinGen`) precisely for this,
and a continuation passes it back with `WithResume(snapshot)`. The
[`resume` log category](../04-frontend/log-categories.md#resume--a-run-continuing-an-earlier-one)
reports whether the seeding was *applied* (`info`, with `seededNodes` and `joinWatermarks`)
or *skipped* (`warn`, snapshot absent or the flow drifted). The mechanics are walked in
full in [The wire](the-wire.md).

This matters immediately, because the second half of this chapter is a mechanism that parks
and resumes on purpose.

---

## Part 2 — Continue After: a delay is a pattern, not a primitive

*Continue After* is FloMorphic's card for **"stop here, and carry on at 9am tomorrow"** —
a delay, a schedule, a cooling-off period, a 90-day re-check.

The engine has no delay primitive. It does not need one, because a delay is just
**park-and-resume**, and park-and-resume is something a backend can already do with the
pieces in [the backend contract](the-backend-contract.md).

### The shape

```
   live run                                   later
  ────────────────►  ✋ park                  ⏰ ────────────────►
   Extrinsic node       │                      │      resumed run
   svc.continue.at      │                      │      same context
                        ▼                      │      seeded traversal
              scheduled process row ───────────┘
```

Five moving parts, none of them in the engine:

1. A palette card that lowers to an **Extrinsic** node on one of your subjects.
2. A handler for that subject that **records a scheduled row** and answers **stop**.
3. A scheduler in your backend that launches due rows.
4. The **same context document**, so the resumed run continues rather than restarts.
5. A **resume snapshot**, so a join past the park point does not lock.

### 1. It lowers to an Extrinsic

```go
// buildUntilNode lowers a Continue After node to an extrinsic on svc.continue.at.
func buildUntilNode(node *inflowModels.Node, vfn compiler.VueFlowNode, nodeData map[string]any) error {
    node.Type = inflowModels.ExtrinsicNodeType
    evNode := inflowNodes.NewExtrinsicSvcNode(ContinueSubject, /* ... */)
    evNode.ExtrinsicRule.ReqTimeoutSecound = 10

    mode := getStr(nodeData, "mode")
    payload := map[string]any{"mode": mode}
    switch {
    case mode == "at":
        payload["at"] = getStr(nodeData, "at")
        payload["continueAt"] = parseAtMillis(getStr(nodeData, "at"))   // absolute, at compile time
    case delayModeSeconds(mode) > 0:
        payload["delaySeconds"] = int64(getFloat(nodeData, "value") * float64(delayModeSeconds(mode)))
    }
    evNode.ExtrinsicRule.OperationData = payload
    node.Extrinsic = &evNode.ExtrinsicRule
    return nil
}
```

`ContinueSubject` is `"svc.continue.at"` — an ordinary subject in this backend's own
namespace, registered like any other domain action.

Note the split, which is a good general rule for compiling schedules:

| The author wrote | Resolved | Why there |
| --- | --- | --- |
| An **absolute** date (`mode: "at"`) | At **compile** time, to `continueAt` epoch-millis | The instant is fixed; resolving it once is right |
| A **relative** delay (`3 days`) | At **run** time, from `delaySeconds` | "Three days" means three days *from when the run got here*, which compile time cannot know |

The timeout is ten seconds, because the handler only writes a row — the *wait* is not this
request.

### 2. The handler parks the run

`HandleContinueAfter` (`flomorphic-api/inflow/continue.go`) does four things:

```go
// Identity travels on the runtime headers; the nodeId comes off the request body.
flowID    := header.Get("flowId")
contextID := header.Get("contextId")
pid       := header.Get("pid")
instanceID := header.Get("instanceId")          // falls back to pid

// Collapse the schedule to one absolute instant.
scheduledAt := op.ContinueAt
if scheduledAt <= 0 {
    scheduledAt = nowMillis() + op.DelaySeconds*1000
}

// The resumed run starts from this node's own outbound edges.
nexts := body.Node.Next
startNodeIDs := nextNodeIDs(nexts)

rec, err := StartWorkflow(ctx, store, StartParams{
    FlowID: flowID, ContextID: contextID,
    StartNodeIDs: startNodeIDs,
    ScheduledAt:  scheduledAt,
    RecordMeta: map[string]any{
        "origin": "continue_after",
        "sourceFlowId": flowID, "sourceNodeId": nodeID, "sourcePid": pid,
        "mode": op.Mode, "scheduledAt": scheduledAt, "nextNodes": nexts,
    },
    InstanceID: instanceID,
})

// Stop the live run here; the scheduled row resumes the flow later.
return svcHandler.StopHereResponse(map[string]any{
    "indexId": rec.IndexID, "status": rec.Status, "scheduledAt": scheduledAt,
})
```

Three details are load-bearing:

**`StopHereResponse` is what makes it a park.** It is an ordinary
[job/command reply](../03-plugins/jobs-and-commands.md) telling the engine to end this
branch here rather than follow `Next`. Without it the run would schedule a resume *and*
carry straight on — two continuations for one node.

**The outbound edges are read from the request body, not resolved at compile time.** The
compiler deliberately leaves them in the node's compiled `Next`, and the handler recovers
them from the request. So a Continue After that **fanned out** resumes as *the branches it
actually had* — `StartNodeIDs` is a list, and the scheduler relaunches on every one of
them:

> *"Relaunch on every node the row was recorded to start from — a resume that picked up
> several parked branches must not collapse to its first one."*

**`instanceId` survives the park.** Each run keeps its own `pid`, but the park→resume chain
shares one logical instance id, so the process list can show one continuous story instead
of two unrelated rows. It falls back to the pid, which is what a fresh instance's id is
anyway.

### 3. The scheduler launches it, and seeds the join state

At `ScheduledAt`, a loop in your backend builds an ordinary process request
(`inflow/scheduler.go`):

```go
opts := []func(*fuse.Process){
    fuse.WithFlowId(rec.FlowID),
    fuse.WithContextDocument(rec.ContextID),   // ← the same context: a continuation
    fuse.WithMeta(reqMeta),
    fuse.WithPID(rec.PID),
}

// A scheduled Continue After is a continuation: seed the parked run's state.
if sourcePid, _ := rec.Meta["sourcePid"].(string); sourcePid != "" {
    if snap := loadResumeSnapshot(ctx, s.store, sourcePid); snap != nil {
        opts = append(opts, fuse.WithResume(snap))
    }
}

p, _ := fuse.NewProcess(startNodesFor(rec), opts...)
resp, err := p.Exec(ctx)
```

**The snapshot is keyed by `sourcePid` — the run that parked — not by this row's own fresh
pid.** And it is fetched **at launch**, not when the row was written, because the parked
run produced the snapshot *at its end*, after this row already existed. Getting either
wrong gives you a resume that hangs at the first join downstream and fails at the process
timeout. That is the failure mode [The wire](the-wire.md) is largely about.

### The same shape, three features

Once you have this pattern, you have the whole family. FloMorphic writes it twice and gets
three things:

| Feature | Parks on | Resumed by |
| --- | --- | --- |
| **Continue After** | `svc.continue.at` | A time-based scheduler |
| **Human-in-the-loop** | `svc.hitl.add` | A person answering a task |
| **Wait for an external event** | your subject | A webhook, a queue consumer, a callback |

All three are: *Extrinsic node → handler records something → `StopHereResponse` → later,
`NewProcess` over the same context with `WithResume`.* The third row is not implemented in
FloMorphic and needs nothing new to be — which is the point of listing it.

> **A park costs nothing while it waits.** There is no held connection, no timer thread, no
> parked goroutine, no engine state pinned open. The run *ended*; a row in your database is
> the only thing that survives it. That is why a flow can pause for ninety days as cheaply
> as for ninety seconds.

---

## The two mechanisms, side by side

| | **`Depends`** | **Continue After** |
| --- | --- | --- |
| Waits for | Other branches of this run | The clock |
| Lives in | A field on every node | A pattern in your backend |
| Engine support | Native — the traversal blocks | None — the run *ends* and a new one starts |
| Costs while waiting | A blocked node in a live process | Nothing. A database row |
| Survives a restart | Via the traversal snapshot | Inherently — it was never running |
| Compiled from | A post-pass over edges | A per-node hook case |
| On the stream | `dep.wait` / `dep.ready` | `proc.finish`, then a new `pid` with `resume` |
| Your product's card | *Wait for All*, *Join*, *Gather*, *Barrier* | *Delay*, *Wait until*, *Schedule*, *Snooze* |

Both are worth understanding together because a graph that uses **both** is the one that
breaks: a join downstream of a park is exactly the case the resume snapshot exists for, and
exactly the case that hangs silently when it is missing.

---

## For your own product

- [ ] `Depends` exposed — as a barrier card, a per-node checkbox, or both
- [ ] Edge-derived `Depends` filled in a **compiler post-pass**, not the per-node hook
- [ ] Fan-in vs. join surfaced in the authoring UI — the three validations above
- [ ] A many-valued `scope` not mistaken for branching, in docs or validation
- [ ] `dep.wait` / `dep.ready` rendered, so a parked join explains itself
- [ ] A park subject registered, if anything in your domain waits for time, people or events
- [ ] The park handler answering **`StopHereResponse`** — not just recording a row
- [ ] Resume launched over the **same context id**
- [ ] The resume snapshot loaded by the **producing** pid, **at launch time**
- [ ] A multi-branch park resuming on **every** captured start node

---

## Next

- **[The wire](the-wire.md)** — the park/resume machinery end to end, including the
  traversal snapshot and the lost-finish reconciliation.
- **[Tag routing](tag-routing.md)** — the fan-out these joins put back together.
- **[The log taxonomy](../04-frontend/log-categories.md)** — `dep.wait`, `dep.ready`,
  `scope.fanout` and `resume`, as a frontend consumes them.

**Source material:** `inflow-fusion/models/flow.go`, `docs/nodes.md`, `docs/routing.md`,
`docs/nodes/void.md`; `flomorphic-api/inflow/compiler.go`, `inflow/continue.go`,
`inflow/node_builders.go`, `inflow/scheduler.go`, `designer/patch.go`.

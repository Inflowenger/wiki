# The wire: a backend, end to end

> The contract is three methods. The *wire* is what a real backend does with them once it
> accepts that a run it started may finish six hours from now, on a different engine
> instance, after this process has been restarted twice.

[The backend contract](the-backend-contract.md) states the obligation: answer three
questions. That chapter is deliberately small, because the obligation really is small. This
chapter is the opposite — it takes one production implementation and walks all of it,
because the three methods are not the interesting part.

The interesting part is this:

> **`Exec` returns a pid, not a result.**

A `ProcessRequest` is a dispatch, not a call. The POST returns as soon as the engine has
accepted the run, and everything you actually want to know — did it work, what did it
produce, where did it stop, what went wrong on the way — arrives **later**, through three
separate back-channels, at times you do not control:

| What arrives | How | When |
| --- | --- | --- |
| the run's data | `UpdateContext`, repeatedly | during the run, and once at the end |
| the run's traversal state and error ledger | on the **final** `UpdateContext` | at the end |
| the run's verdict | a `proc.finish` event on the log stream | at the end — **or never** |

"At the end" can be milliseconds or days. A flow that parks on a *Human in the Loop* node
ends immediately and its logical continuation starts when a person clicks Approve next
Tuesday. A flow with a *Continue After* node ends now and resumes at 03:00. A run that
outlives your API process leaves you with a row marked `running` and no event coming.

Every design decision below falls out of that one fact.

---

## The reference implementation

Everything in this chapter is real code from **`flomorphic-api`**, the backend behind
[FloMorphic](../04-flomorphic/index.md). It is a Go + Fiber service over SQLite, which is
worth noticing on its own: the engine does not care, so the product picked the most modest
storage that suited it.

The whole wire is one package, `flomorphic-api/inflow/`:

| File | Responsibility |
| --- | --- |
| `port.go` | boot: `InitBackend`, register extrinsic handlers, subscribe the event log |
| `wire.go` | **the three contract methods**, and lifting run outcome onto the process row |
| `process.go` | `StartWorkflow` / `StopWorkflow`, the meta keys, snapshot load/store |
| `events.go` | the `proc.finish` subscriber that closes process rows |
| `scheduler.go` | launches `scheduled` rows when their time arrives |
| `continue.go` | the *Continue After* handler — park now, schedule the resume |
| `hitl_resume.go`, `hitl_context.go` | the *Human in the Loop* park and its resume |
| `trace.go` | reconciliation when a `proc.finish` never arrives |
| `compiler.go`, `node_builders.go` | the [compiler seam](the-compiler-seam.md) — not the wire |

Read it in that order and the design argues for itself. Below, it is grouped by concern
instead.

---

## 1. Booting the wire

`InitBackend` takes your `IInflowService` implementation and the two infra coordinates.
That is the whole of it:

```go
// port.go
func InitInflowConnection(store repository.Store) error {
    return fuse.InitBackend(
        fuse.WithImplementedBackendBy(&InflowWire{store: store}),
        fuse.WithJwtSecretKey(env.GetInfraJWTSecret()), // INFLOW_INFRA_JWT_SECRET
        fuse.WithInfraApi(env.GetInfraApiUrl()),        // INFLOW_INFRA_API
    )
}
```

`InflowWire` is the implementation, and note how little state it holds:

```go
// wire.go
type InflowWire struct {
    store repository.Store
}
```

One dependency: your storage. There is no engine client, no connection, no subject table,
no run registry. The SDK owns the NATS connection; the three methods are callbacks it
invokes.

> **The rule worth repeating:** never dial NATS yourself. The only NATS you touch is the
> `*nats.Msg` handed to these methods and the `nats.Header` handed to an extrinsic handler.

---

## 2. The three methods

### `RetrieveFlow` — *what is this flow?*

The engine asks for a `flowId` and expects a compiled `models.Flow`. This is where your
[compiler](the-compiler-seam.md) runs:

```go
// wire.go
func (w *InflowWire) RetrieveFlow(msg *nats.Msg) {
    parts := strings.Split(msg.Subject, ".")
    flowId := parts[len(parts)-1]

    rec, err := w.store.Workflows().GetByID(context.Background(), flowId)
    if err != nil {
        msg.Respond([]byte(`{}`))
        fmt.Println("inflow: flow not found:", flowId)
        return
    }
    _, compiled, err := FLowCompiler(*rec)   // ← your format → primitives
    if err != nil {
        msg.Respond([]byte(`{}`))
        fmt.Printf("inflow: compile flow %s: %v\n", flowId, err)
        return
    }
    flow := inflowModels.Flow{UUID: flowId, Nodes: []inflowModels.Node{}}
    for _, node := range compiled {
        flow.Nodes = append(flow.Nodes, *node)
    }
    b, _ := sonic.Marshal(flow)
    msg.Respond(b)
}
```

Three things to take from it:

- **The flow is compiled on demand, per run, not on save.** FloMorphic stores the raw Vue
  Flow graph verbatim and compiles it here. That means an edited flow takes effect on the
  next run with no migration step — and it is also exactly why a resume gates on a
  structural signature (§5).
- **`msg.Respond` is not optional.** The engine is doing a NATS request with a timeout. A
  handler that returns without responding does not fail fast; it stalls the run until the
  request timeout expires. Respond on *every* path, including errors.
- **`{}` is the failure answer.** An empty flow is a run that starts and immediately has
  nowhere to go. That is a legible failure; a timeout is not.

For a worked, self-contained `RetrieveFlow` that hand-builds a flow with no compiler and no
database at all, see **`inflow-fusion/backend_sample.go`** — it is the smallest complete
backend in the ecosystem and the best thing to read next to this.

### `RetrieveContext` — *what is this run's context?*

```go
// wire.go
func (w *InflowWire) RetrieveContext(msg *nats.Msg) {
    parts := strings.Split(msg.Subject, ".")
    ctxId := parts[len(parts)-1]

    rec, err := w.store.Contexts().GetByID(context.Background(), ctxId)
    if err != nil {
        if repository.HasPrefix(ctxId, repository.ContextIDPrefix) {
            fresh := &models.ContextRecord{
                ID:        ctxId,
                Context:   "{}",
                UpdatedBy: models.LastChange{By: models.ByFlow, Address: msg.Header.Get("flowId")},
            }
            if cerr := w.store.Contexts().Upsert(context.Background(), fresh); cerr != nil {
                fmt.Printf("inflow: lazily create context %s: %v\n", ctxId, cerr)
            }
        }
        msg.Respond([]byte(`{}`))
        return
    }
    doc := inflowModels.ContextDoc{Header: rec.Header, Data: rec.Context}
    b, _ := sonic.Marshal(doc)
    msg.Respond(b)
}
```

The miss path is the part worth studying. A run launched against a context id with no
backing row does not just need `{}` to start from — it needs **somewhere for its later
writes to land**. `UpdateContext` bails when the row is absent, so answering `{}` without
materializing the row would give you a run that executes perfectly and persists nothing.

So the handler creates an empty row under that id, guarded by an id-prefix check so a
malformed id does not litter the table. Normal launches (the Run dialog, a trigger) create
the row up front; this only fires for ids passed in without one.

> A recurring shape in this package: **the wire repairs what it can and answers legibly
> when it cannot.** It never leaves the engine waiting.

### `UpdateContext` — *take this updated context*

This is the method that carries the weight, and the one that is not what it appears to be.
It looks like a setter. It is also the **run-outcome delivery channel**:

```go
// wire.go
func (w *InflowWire) UpdateContext(msg *nats.Msg) {
    contextId := msg.Header.Get("contextId")

    var incoming inflowModels.ContextDoc
    if err := sonic.Unmarshal(msg.Data, &incoming); err != nil {
        fmt.Printf("inflow: invalid context payload for %s: %v\n", contextId, err)
        return
    }
    if repository.HasPrefix(contextId, repository.ContextIDPrefix) {
        rec, err := w.store.Contexts().GetByID(context.Background(), contextId)
        if err != nil {
            fmt.Printf("inflow: context %s not found\n", contextId)
            return
        }
        rec.Context = incoming.Data
        rec.Header = incoming.Header
        rec.UpdatedBy = models.LastChange{By: models.ByFlow, Address: msg.Header.Get("flowId")}
        if err := w.store.Contexts().Upsert(context.Background(), rec); err != nil {
            fmt.Printf("inflow: failed to update context %s: %v\n", contextId, err)
            return
        }
    }

    // The run's own outcome — traversal snapshot and error ledger — goes on the
    // PROCESS row, per-pid, not on the shared context row above.
    w.storeRunOutcome(msg.Header.Get(MetaPidKey), incoming.Header)

    msg.Respond([]byte(`accepted`))
}
```

Two storage targets, one message. The document goes to the **context row** (shared, keyed
by `contextId`). The run's traversal snapshot and error ledger go to the **process row**
(private, keyed by `pid`). §4 and §5 are about why that split exists and what it fixed.

---

## 3. What rides on the headers

Before any of that works, you need identity on the message — and the engine only tells you
what you told it.

> **The mechanism:** every entry in `ProcessRequest.Meta` is forwarded verbatim as a NATS
> header on every outbound message of that run — the context get, the context set, and
> every extrinsic service call.

That is the *whole* correlation mechanism. The engine adds `contextId` on the context push
of its own accord, and nothing else. So if your handler needs to know which run is calling
it, you must put that fact in the meta at launch:

```go
// process.go
const (
    MetaIndexKey    = "indexId"    // your own row id, echoed back
    MetaFlowKey     = "flowId"
    MetaContextKey  = "contextId"
    MetaPidKey      = "pid"
    MetaInstanceKey = "instanceId" // the logical instance across park→resume
)
```

This is worth being blunt about, because it is the mistake that costs the most:

> **Mint the pid yourself.** The engine will generate one if you do not, but it generates
> it *after* dispatch — by which time it cannot be in the meta, so it cannot be a header,
> so a service handler cannot learn which run called it. In FloMorphic that showed up as
> Human-in-the-Loop tasks recorded with an empty pid, and therefore unresumable.

```go
// process.go — StartWorkflow
pid := uuid.NewString()
reqMeta[MetaPidKey] = pid
// ...
opts := []func(*fuse.Process){
    fuse.WithFlowId(params.FlowID),
    fuse.WithContextDocument(params.ContextID),
    fuse.WithMeta(reqMeta),
    fuse.WithPID(pid),          // ← the same pid the engine will use
}
```

### pid vs. instanceId

These are different, and the difference is the whole async story in one line:

| | Scope | Changes on resume? |
| --- | --- | --- |
| **pid** | one engine execution | **yes** — every run gets a fresh pid |
| **instanceId** | one *logical* workflow instance | no — the whole park→resume chain shares it |

A flow that parks for approval and resumes is **two runs, two pids, one instance**. The
resume is a genuinely new execution entered on the parked node's successors — not a
reawakened one — because the engine held nothing open while it waited.

```go
// process.go
instanceID := strings.TrimSpace(params.InstanceID)
if instanceID == "" {
    instanceID = pid        // a fresh instance is named by its first run
}
reqMeta[MetaInstanceKey] = instanceID
rec.InstanceID = instanceID
```

Report per-pid and a three-step approval chain looks like three unrelated runs. Report per
`instanceId` and it looks like one thing that took two days — which is what the user
thinks it is.

---

## 4. The process row: the ledger the engine does not keep

A fractal instance **forgets a pid the moment its run ends**. There is no run history in
the engine to query. If you want to know what happened, you keep the record.

```go
// models/process.go
type Process struct {
    IndexID     int64          `json:"indexId"`      // your id — auto-increment
    PID         string         `json:"pid"`          // this execution
    InstanceID  string         `json:"instanceId"`   // this logical instance
    FlowID      string         `json:"flowId"`
    ContextID   string         `json:"contextId"`
    StartNodeID string         `json:"startNodeId"`
    Status      ProcessStatus  `json:"status"`
    ResourceURL string         `json:"resourceUrl"`  // which engine took it
    Request     map[string]any `json:"request"`      // the dispatched request, verbatim
    Meta        map[string]any `json:"meta"`         // backend-only, never sent
    Snapshot    map[string]any `json:"snapshot"`     // ← the run's "_sched"
    Errors      map[string]any `json:"errors"`       // ← the run's "_errors"
    Error       string         `json:"error"`        // fatal, if it died
    ScheduledAt int64          `json:"scheduledAt"`
    StartedAt   int64          `json:"startedAt"`
    FinishedAt  int64          `json:"finishedAt"`
    DurationMs  int64          `json:"durationMs"`
}
```

Lifecycle:

```
                    ┌──────────────┐
    ScheduledAt>0 ─►│  scheduled   │──── scheduler fires ──┐
                    └──────────────┘                       │
                           │ StopWorkflow                  ▼
                           │                        ┌──────────────┐
    immediate ─────────────┼───────────────────────►│   running    │
                           │                        └──────┬───────┘
                           ▼                               │
                    ┌──────────────┐        proc.finish     │
                    │   stopped    │◄───────────────────────┤
                    └──────────────┘                        │
                    ┌──────────────┐                        │
                    │   finished   │◄───────────────────────┤
                    └──────────────┘                        │
                    ┌──────────────┐                        │
                    │    failed    │◄───────────────────────┘
                    └──────────────┘
```

### The create-then-update dance

`StartWorkflow` writes the row **twice**, on purpose:

```go
// process.go — abbreviated
// 1. Create first, to learn the auto-increment index.
if err := store.Processes().Create(ctx, rec); err != nil {
    return nil, fmt.Errorf("record process: %w", err)
}

// 2. Echo that index into the request meta, so the run carries its own
//    record identity into every service it calls.
reqMeta[MetaIndexKey] = strconv.FormatInt(rec.IndexID, 10)

// 3. Build the request, store it on the row.
p, err := fuse.NewProcess(startNodeIDs, opts...)
req := p.GetRequest()
rec.PID = req.PID
rec.ResourceURL = p.GetResource()
rec.Request = requestToMap(req)
if err := store.Processes().Update(ctx, rec); err != nil {
    return rec, fmt.Errorf("record process request: %w", err)
}

// 4. Only now dispatch.
resp, err := p.Exec(ctx)
```

The row is **complete and committed before `Exec`**. This is not tidiness — it is the
async contract:

> A `proc.finish` can race back on the event log before `Exec` has even returned. If the
> row is not already there, the finish handler finds nothing to close and the run is
> orphaned in `running` forever.

Write the row first. Always.

### Two metas, and the line between them

| | Goes to the engine? | Holds |
| --- | --- | --- |
| `StartParams.ReqMeta` → `ProcessRequest.Meta` | **yes**, as headers | correlation ids, backend-injected tags |
| `StartParams.RecordMeta` → `Process.Meta` | **no** | resume bookkeeping, origin tags |

`RecordMeta` is where the backend keeps what only it needs:

```go
// continue.go — a scheduled Continue After row
recordMeta := map[string]any{
    "origin":       "continue_after",
    "sourceFlowId": flowID,
    "sourceNodeId": nodeID,
    "sourcePid":    pid,        // ← the run that parked. This is the key to the snapshot.
    "mode":         op.Mode,
    "scheduledAt":  scheduledAt,
    "nextNodes":    nexts,      // ← where the resume re-enters
}
```

`ReqMeta` is **server-assembled only**. It becomes headers on calls into your own service
handlers, so treating it as a place to pass through a request body is handing a client the
ability to forge a run's identity.

Two details on `RecordMeta` that exist for the scheduler's benefit:

```go
// process.go
// The row's StartNodeID column holds ONE node. A resume that picks up every branch
// a parked node fanned out to must not collapse to the first one.
recordMeta[MetaStartNodesKey] = startNodeIDs

// A scheduled row is relaunched from a freshly BUILT request, not the stored one,
// so the resume intent has to survive on the record to reach that later launch.
if params.Resume != nil {
    recordMeta[MetaResumeKey] = true
}
```

---

## 5. The traversal snapshot — `_sched`

This is the mechanism that makes "waits three days, then continues correctly" true, and it
is the part most worth understanding, because the naive version of it is subtly broken.

### Why data is not enough

Node results live in the context document, so a plain restart continues fine **on a linear
path**. The problem is a **join**.

A join does not check whether its dependencies have *data*. It checks their completion
**generation** — a counter, per node. Consider:

```
    ┌─► B ──────────────────────────┐
A ──┤                               ├──► D   (join: waits for B and C)
    └─► C ──► [park 3 days] ──► C' ─┘
```

`B` completed during run 1. Run 2 enters at `C'`. From run 2's point of view, `B` has
generation 0 — it has never run — so `D` waits for it, and `B` is upstream of the resume
point and will never be re-dispatched. The join blocks until the process timeout.

Seeding the previous run's generations is what stops that.

### The shape

The engine builds it at run end and writes it into the context header under `_sched`:

```go
// fractal-core/models/core.go
type ResumeState struct {
    FlowSig  string             `json:"flowSig"`  // structural fingerprint — the gate
    Traverse map[string]NodeGen `json:"traverse"` // "flowScope:nodeId" → completion
    JoinGen  map[string]int     `json:"joinGen"`  // join watermarks
    Flows    []string           `json:"flows"`    // every flow scope the run mounted
}

type NodeGen struct {
    Count  int `json:"c"`   // generation — how many times it completed
    Status int `json:"s"`   // only `done` is ever recorded
}
```

Concretely:

```json
{
  "flowSig": "9f2c41ab77e0d513",
  "traverse": {
    "flow_7x:node0": { "c": 1, "s": 3 },
    "flow_7x:node1": { "c": 1, "s": 3 },
    "flow_7x:node2": { "c": 1, "s": 3 }
  },
  "joinGen": { "flow_7x:node6": 1 },
  "flows": ["flow_7x", "flow_7x::goto_a"]
}
```

Two kinds of `done` are deliberately **dropped** when it is built:

- a `waiting`/`inProgress` node — it would never reach `done` on the next run and would
  hang the drain loop;
- a node released synthetically at shutdown (an abandoned join marked done only so the
  drain could finish) — it never actually ran, so seeding it done would wrongly satisfy its
  dependents.

Only genuinely-completed work is seeded. Everything else re-runs.

### `flowSig`: the gate that makes this safe

Generations are keyed **by node id**. A flow edited between the snapshot and the resume
would attach run 1's generations to run 2's nodes — silently, and wrongly.

So the engine fingerprints the loaded topology (every node key and the ids its edges
target, order-independent) and compares:

```go
// fractal-core/engine/resume.go
func (p *Process) seedSched() (skipReason string) {
    snap := p.req.Resume
    if sig := p.store.signature(); sig != snap.FlowSig {
        logtool.GetLogger().Info("resume: flow definition changed since snapshot — continuing with blank state")
        return "flow definition changed since the snapshot was taken"
    }
    for key, w := range snap.Traverse {
        if w.Status != int(done) {
            continue // defence in depth: only completed state is ever seeded
        }
        p.nodeTraverse[key] = state{count: w.Count, status: done}
    }
    for key, g := range snap.JoinGen {
        p.joinGen[key] = g
    }
    return ""
}
```

> **Drift is not an error.** A mismatch degrades to a blank continue and logs the skip. An
> edited flow cannot resume into a stale plan — but it also does not fail the run.

### Why it is keyed by pid, and not by context

Here is the bug this design exists to fix.

The snapshot *travels* in the context header, because that is the message the engine
already sends. But the context header is **one slot per `contextId`** — and several runs
can be live over the same context at once. Two overlapping runs each write `_sched` to the
same slot, and the second clobbers the first. A resume then seeds a **sibling run's**
traversal state.

So the backend lifts it off the message and stores it **per-pid, on the process row**:

```go
// wire.go
func (w *InflowWire) storeRunOutcome(pid string, header map[string]any) {
    if strings.TrimSpace(pid) == "" || header == nil {
        return
    }
    sched, hasSched := header[schedHeaderKey].(map[string]any)  // "_sched"
    errs, hasErrs := header[errHeaderKey].(map[string]any)      // "_errors"
    if !hasSched && !hasErrs {
        return   // an ordinary mid-run context write — skip
    }
    rec, err := processByPID(context.Background(), w.store, pid)
    if err != nil {
        fmt.Printf("inflow: run outcome: process pid %s not found: %v\n", pid, err)
        return
    }
    if hasSched {
        rec.Snapshot = sched
    }
    if hasErrs {
        rec.Errors = errs
    }
    if err := w.store.Processes().Update(context.Background(), rec); err != nil {
        fmt.Printf("inflow: run outcome: update process %d (pid %s): %v\n", rec.IndexID, pid, err)
    }
}
```

Notice the guard: a message carrying **neither** key is an ordinary mid-run context write
and is skipped, so this only does work on the write that ends a run. And the two are taken
**independently** — a run that errored and a run that did not both write a snapshot, and
neither may be inferred from the other — but written in one update, because they describe
the same instant of the same run.

The engine, for its part, no longer reads `_sched` back from the header at all. The
snapshot now rides **on the resume request**, per-run:

```go
// inflow-fusion/models/core.go
type ProcessRequest struct {
    // ...
    Resume *ResumeState `json:"resume,omitempty"`
}
```

Which is the round trip:

```
  run A ends ──► "_sched" in context header ──► UpdateContext lifts it
                                                      │
                                          process row (pid = A).Snapshot
                                                      │
                            loadResumeSnapshot(A) ────┘
                                                      │
                                    ProcessRequest{ Resume: snap } ──► run B seeds it
```

### Reading it back

```go
// process.go
func loadResumeSnapshot(ctx context.Context, store repository.Store, pid string) *inflowModels.ResumeState {
    if strings.TrimSpace(pid) == "" {
        return nil
    }
    rec, err := processByPID(ctx, store, pid)
    if err != nil || len(rec.Snapshot) == 0 {
        return nil
    }
    b, err := sonic.Marshal(rec.Snapshot)
    if err != nil {
        return nil
    }
    var snap inflowModels.ResumeState
    if err := sonic.Unmarshal(b, &snap); err != nil {
        return nil
    }
    return &snap
}
```

> **The pid you look it up by is the *producing* run's pid** — the parked run, the
> `sourcePid` — never the continuation's own fresh pid, and never the `contextId`.

And note `processByPID`, not "get running by pid". By the time a resume looks the snapshot
up, the run that produced it is long finished:

```go
// process.go
// processByPID returns the process row for an engine pid, across any status —
// unlike GetRunningByPID, whose row is gone once the producing run finished,
// which is exactly the state it is in by the time a resume looks it up.
func processByPID(ctx context.Context, store repository.Store, pid string) (*models.Process, error) {
    items, _, err := store.Processes().List(ctx, repository.ListParams{PID: pid, Limit: 1})
    // ...
}
```

Every failure path here returns `nil`, and `nil` means *blank continue* — the engine logs a
resume-skipped line and seeds nothing. A missing snapshot degrades the resume; it never
fails it.

---

## 6. The error ledger — `_errors`

The snapshot's neighbour in the header, and the answer to a question the run status cannot
answer.

> **In fractal, a node error is not the end of anything.** The node handles it, the flow
> carries on. So a run that hit eleven errors can finish `completed`, and without a ledger
> the row says "finished" and nothing more.

That is the right execution semantics and the wrong reporting default, so the engine keeps
a ledger:

```go
// fractal-core/models/errors.go
type RunErrors struct {
    Pid   string      `json:"pid"`
    Count int         `json:"count"`   // the TRUE total
    Items []NodeError `json:"items"`   // capped — may be shorter than Count
}

type NodeError struct {
    Ts   int64     `json:"ts"`
    Kind ErrorKind `json:"kind"`   // "node" | "system"
    Flow string    `json:"flow"`
    Node string    `json:"node"`
    Src  string    `json:"src"`    // rt / js / rego / plugin:<title>
    Code int       `json:"code"`
    Msg  string    `json:"msg"`
    Loc  string    `json:"loc,omitempty"` // json path, for a fan-out node
}
```

### `kind` is the field that matters

| `kind` | Means | Whose problem |
| --- | --- | --- |
| `node` | the node's own operation failed — the author's js or rego, their node data, a path they addressed, something the node called that did not deliver | **the flow author's.** This is what `StopOnError` refuses to carry on past. |
| `system` | fractal's own machinery failed while serving the node — infra that would not answer, a reply that would not marshal, a context write that did not land | **the platform's.** Nothing in the flow caused it and nothing in the flow mends it. |

Both are recorded, because a run that lost its infrastructure halfway must not be
indistinguishable from a clean one. But they are not the same news, and only one of them is
something a flow author can act on. Surface the distinction in your UI; collapsing it
blames the user for your outage.

### Three properties to design around

**It describes exactly one run.** The engine `delete`s the slot when it loads the document,
so whatever a previous run over the same context left there is gone. `Pid` names the run
that wrote every item.

```go
// fractal-core/engine/process.go
p.docHeader = doc.Header
// ...
delete(p.docHeader, errHeaderKey)
```

**`Count` is the truth; `Items` is a sample.** Entries are capped at 100. A run that
cascades raises an error per location per node, and the whole header ships inside the
context document, which has a publish payload limit — the cap is what keeps a bad run's
document publishable at all. Render `Count`, not `len(Items)`.

**The entries kept are the *first* ones.** A cascade's cause is at the front, so keeping
the head is both cheaper and more useful than keeping the tail.

### `StopOnError`, if you want the other semantics

```go
// inflow-fusion/models/core.go
type Settings struct {
    RequestTimeOut   int64  `json:"svc_req_timeout"`
    ExecuteTimeOut   int64  `json:"proc_timeout"`
    ProcessNodeLimit uint16 `json:"proc_node_limit"`
    StopOnError      bool   `json:"stop_on_error"`
}
```

`StopOnError` is narrower than it sounds. It prunes **this node's outgoing edges** — the
flow does not continue past this spot. It does not interrupt the process: every other
branch runs to its end, the document is committed and pushed as usual. And it reacts to
`kind: "node"` only; a `system` fault the node *survived* does not cut the branch, because
the job may still conclude successfully afterwards.

---

## 7. The ordering that keeps the outcome

`storeRunOutcome` is called **before** `msg.Respond`. That is deliberate and it is the
subtlest line in the file:

```go
// wire.go
w.storeRunOutcome(msg.Header.Get(MetaPidKey), incoming.Header)

msg.Respond([]byte(`accepted`))
```

The engine's final context push is a NATS **request** — it waits for this ack before it
emits `proc.finish`. Two things follow:

1. **The snapshot write is ordered ahead of the finish handler's own update of the same
   row.** Both `storeRunOutcome` and `handleProcFinish` write the process row. Storing
   before the ack puts them in a defined order — no lost update.
2. **Both are on the row before any resume can fire.** A resume is triggered downstream of
   the finish; if the snapshot were written after the ack, a fast resume could read a row
   that has no snapshot yet and silently degrade to a blank continue.

> Respond last. The ack is a scheduling primitive, not a formality.

---

## 8. Park and resume: three shapes

A park is always the same two moves: **the node's handler answers with a stop command**,
and **the backend records where to re-enter**. What differs is who wakes it up.

### The stop command

```go
// continue.go
return svcHandler.StopHereResponse(map[string]any{
    "indexId":     rec.IndexID,
    "status":      rec.Status,
    "scheduledAt": scheduledAt,
})
```

`StopHereResponse` produces `{"_cmd":"stop"}` alongside your data. It deactivates every
outgoing edge of that node, so the branch ends there. The process then finishes normally —
pushes its document, writes `_sched`, emits `proc.finish`. **The engine holds nothing
open.** There is no parked run consuming a slot, which is why a three-day wait costs
nothing.

### Shape A — Human in the Loop (woken by a person)

```go
// hitl_resume.go
func ResumeHumanTask(ctx context.Context, store repository.Store, task *models.HumanTask) (*models.Process, error) {
    if task == nil || task.Mode == models.HumanTaskContinue {
        return nil, nil     // a `continue` node never stopped its flow
    }
    startNodeIDs := nextNodeIDs(task.Nexts)
    if len(startNodeIDs) == 0 {
        return nil, nil     // the node was the end of the line
    }
    // ...
    rec, err := StartWorkflow(ctx, store, StartParams{
        FlowID:       task.FlowID,
        ContextID:    task.ContextID,
        StartNodeIDs: startNodeIDs,
        Resume:       loadResumeSnapshot(ctx, store, task.PID),  // ← by the PARKED run's pid
        InstanceID:   task.InstanceID,                           // ← same logical instance
        RecordMeta: map[string]any{
            "origin":       "hitl_resume",
            "sourceFlowId": task.FlowID,
            "sourceNodeId": task.NodeID,
            "sourcePid":    task.PID,
            "humanTaskId":  task.ID,
            "nextNodes":    task.Nexts,
        },
    })
    // ...
}
```

Two things the task row had to capture at park time for this to work: the **pid** (for the
snapshot) and the **outbound edge list** (for the entry points). A node that fanned out to
three branches resumes as three branches — `StartNodeIDs` is a set, and the engine runs all
of them.

There is also a binding step. The parked node never produced a normal result, because its
handler answered with a stop command — so the person's answers have to be written into the
context by hand, under the node's result key, just before the resume starts:

```go
// hitl_context.go — WriteHumanTaskContext, abbreviated
doc := map[string]any{}
if s := strings.TrimSpace(rec.Context); s != "" {
    if err := json.Unmarshal([]byte(s), &doc); err != nil {
        // present but not a JSON object — leave it alone rather than clobber it
        return fmt.Errorf("hitl context: context %s is not a JSON object, leaving it unchanged: %w", ctxID, err)
    }
}
doc[key] = humanTaskOutcome(task)   // status, answers, questions, messages, text, closedAt
```

After which a downstream node reads the conversation exactly the way it reads any node's
output — `{{$.<key>.text}}`. **That is the general pattern for any park:** whatever the
outside world decided while the engine was not running has to be merged into the document
before the continuation reads it.

### Shape B — Continue After (woken by a clock)

```go
// continue.go — HandleContinueAfter, abbreviated
flowID := header.Get("flowId")        // identity off the headers — see §3
contextID := header.Get("contextId")
pid := header.Get("pid")
instanceID := header.Get("instanceId")

scheduledAt := op.ContinueAt          // absolute wins
if scheduledAt <= 0 {
    scheduledAt = nowMillis() + op.DelaySeconds*1000   // else now + delay
}

rec, err := StartWorkflow(context.Background(), store, StartParams{
    FlowID:       flowID,
    ContextID:    contextID,
    StartNodeIDs: nextNodeIDs(nexts),
    ScheduledAt:  scheduledAt,        // ← records, does NOT dispatch
    RecordMeta:   recordMeta,         // ← carries sourcePid
    InstanceID:   instanceID,
    Resume:       nil,                // ← fetched later. See below.
})
```

`Resume: nil` is the interesting line, and it is not an oversight:

> At the moment this handler runs, **the snapshot does not exist yet.** The parked run is
> still executing — this handler *is* one of its nodes. The snapshot is produced at that
> run's end, after this row has already been recorded. So the row stores `sourcePid` and
> the scheduler fetches the snapshot at fire time.

### Shape C — the scheduler (the clock itself)

Worth reading even if you never write one, because it shows the pattern for anything that
dispatches later.

It does not poll. It treats the processes table as a priority queue:

```go
// scheduler.go
func (s *Scheduler) run(ctx context.Context) {
    for {
        s.dispatchDue(ctx)

        wait := schedulerFallback           // 5m safety net, NOT a polling cadence
        next, err := s.store.Processes().NextScheduled(ctx)
        switch {
        case err == nil:
            if d := time.Until(time.UnixMilli(next.ScheduledAt)); d < wait {
                wait = d
            }
            if wait <= 0 {
                continue    // became due between dispatch and here
            }
        case errors.Is(err, repository.ErrNotFound):
            // nothing waiting — sleep until a Notify or the fallback tick
        default:
            if ctx.Err() != nil {
                return
            }
            fmt.Printf("scheduler: read next scheduled: %v\n", err)
        }

        timer := time.NewTimer(wait)
        select {
        case <-ctx.Done():
            timer.Stop()
            return
        case <-s.wake:      // a new row may now be the nearest
            timer.Stop()
        case <-timer.C:
        }
    }
}
```

One indexed `ORDER BY scheduled_at LIMIT 1`, then sleep until exactly that moment. Re-armed
on three signals: the timer firing, a `Notify()` (coalesced through a 1-buffered channel,
so a burst of new rows causes one re-arm), or the fallback tick. **The first iteration runs
immediately**, so rows that came due while the process was down dispatch at startup — which
is the property you want when "later" can outlive your deployment.

And `launch` is where the deferred snapshot fetch lands:

```go
// scheduler.go
func (s *Scheduler) launch(ctx context.Context, rec *models.Process) error {
    // Claim the row BEFORE dispatching, so a re-entrant pass cannot double-fire it.
    rec.Status = models.ProcessRunning
    rec.StartedAt = nowMillis()
    if err := s.store.Processes().Update(ctx, rec); err != nil {
        return fmt.Errorf("claim row: %w", err)
    }
    // ...
    opts := []func(*fuse.Process){
        fuse.WithFlowId(rec.FlowID),
        fuse.WithContextDocument(rec.ContextID),
        fuse.WithMeta(reqMeta),
        fuse.WithPID(rec.PID),      // reuse the stored pid, so proc.finish closes THIS row
    }
    // NOW the snapshot exists: keyed by the run that parked here, not this row's own pid.
    if sourcePid, _ := rec.Meta["sourcePid"].(string); sourcePid != "" {
        if snap := loadResumeSnapshot(ctx, s.store, sourcePid); snap != nil {
            opts = append(opts, fuse.WithResume(snap))
        }
    }
    p, err := fuse.NewProcess(startNodesFor(rec), opts...)   // ← every recorded start node
    // ...
}
```

Three decisions in there worth stealing:

- **Claim before dispatch.** Flipping to `running` first makes double-firing impossible.
- **Rebuild the request, do not replay it.** A fresh engine is picked at fire time, because
  the instance captured at schedule time may be gone hours later. The stored `Request` is a
  record, not a plan.
- **Reuse the stored pid** so the eventual `proc.finish` closes this same row.

### The whole Continue After sequence

```
 run A                     backend                    scheduler          run B
   │                          │                           │                │
   ├─ node: svc.continue.at ─►│ HandleContinueAfter       │                │
   │                          ├─ StartWorkflow(ScheduledAt)                │
   │                          │    row #42  status=scheduled                │
   │                          │    meta.sourcePid = A                       │
   │                          ├─ notifyScheduler() ──────►│ re-arm timer   │
   │◄──── {"_cmd":"stop"} ────┤                           │                │
   ├─ edges pruned; run ends  │                           │                │
   │                          │                           │                │
   ├─ set.context (pid=A) ───►│ UpdateContext             │                │
   │                          ├─ context row ← data       │                │
   │                          ├─ storeRunOutcome(A)       │                │
   │                          │    row A.Snapshot = _sched │                │
   │                          │    row A.Errors   = _errors│                │
   │◄──── "accepted" ─────────┤                           │                │
   ├─ proc.finish (pid=A) ───►│ handleProcFinish          │                │
   │                          │    row A status=finished  │                │
   ╵                          │                           │                │
                     · · · · · · six hours · · · · · ·    │                │
                              │                           ├─ timer fires   │
                              │◄── loadResumeSnapshot(A) ──┤                │
                              │    row A.Snapshot ────────►│                │
                              │                           ├─ claim row #42 │
                              │                           ├─ POST /engine ─►│
                              │                           │   Resume: snap  ├─ seeds joins
                              │                           │                 ├─ runs on
```

---

## 9. Closing the row — and when the event never comes

The last back-channel. `proc.finish` is a **fire-and-forget** event on a shared stream
carrying every event for every process on the engine:

```go
// events.go
func handleProcFinish(store repository.Store, msg *nats.Msg) {
    f, ok := logs.CaptureProcFinish(msg.Data)
    if !ok {
        return // not a proc.finish — normal on a shared subject
    }
    rec := resolveFinishedRecord(store, msg, f.Pid)
    if rec == nil {
        fmt.Printf("process: proc.finish for pid=%s has no running row to close\n", f.Pid)
        return
    }
    rec.Status = finishStatus(f.Status)
    rec.FinishedAt = f.Ts
    if rec.FinishedAt == 0 {
        rec.FinishedAt = nowMillis()
    }
    rec.DurationMs = f.DurationMs
    rec.Error = f.Error
    // ...
}
```

`proc.finish` carries **only the pid** — no meta. So the row is resolved by *the single
`running` row for that pid*, which is precisely why the record keeps an `indexId` of its
own: a pid can back several rows over time, but only one is ever `running`.

```go
// events.go
func resolveFinishedRecord(store repository.Store, msg *nats.Msg, pid string) *models.Process {
    if msg.Header != nil {
        if raw := msg.Header.Get(MetaIndexKey); raw != "" {   // defensive fast path
            if idx, err := strconv.ParseInt(raw, 10, 64); err == nil {
                if rec, err := store.Processes().GetByIndex(context.Background(), idx); err == nil {
                    return rec
                }
            }
        }
    }
    rec, err := store.Processes().GetRunningByPID(context.Background(), pid)
    if err != nil {
        return nil
    }
    return rec
}
```

Status mapping is a small, total function:

| `proc.finish` status | Row status |
| --- | --- |
| `FinishCompleted` | `finished` |
| `FinishStopped` | `stopped` |
| `FinishFailed` | `failed` |
| anything else | `finished` |

### The lost finish

Fire-and-forget means exactly that. If your API was restarting when the event went out, it
is gone — and you are left with a row marked `running` for a run that ended, on an engine
that has already **forgotten the pid**.

This is where it gets genuinely awkward, and the honest answer is a third source of truth.
Infra retains a pid's trace for a week:

```go
// trace.go
type InfraTrace struct {
    ID       string `json:"id"`
    PID      string `json:"pid"`
    Resource string `json:"resource"`
    StartAt  int64  `json:"start_at"`
    FinishAt int64  `json:"finish_at"`
    Data     struct {
        Status     string `json:"status"`
        DurationMs int64  `json:"durationMs"`
    } `json:"data"`
}

func (t *InfraTrace) Finished() bool { return t != nil && t.FinishAt > 0 }
```

`GET {INFLOW_INFRA_API}/inflow/trace/{pid}`, with three outcomes kept carefully distinct
because the caller acts differently on each:

| Result | Means |
| --- | --- |
| `(record, nil)` | infra has a trace — check `FinishAt` for ended vs. live |
| `(nil, nil)` | infra positively has **no** record (404, or 200 with null data) — no evidence the run is alive |
| `(nil, err)` | the lookup itself failed — **you could not determine the status** |

`StopWorkflow` is the place this pays off. Stopping means different things depending on
where the row is:

```go
// process.go — StopWorkflow, structure
switch rec.Status {
case models.ProcessFinished, models.ProcessStopped, models.ProcessFailed:
    return rec, nil     // idempotent — a double-stop is a harmless no-op
}

if rec.Status == models.ProcessScheduled {
    // Never dispatched — no engine process exists for its pid, so /ps/stop would
    // error. Flip the row (dropping it out of the scheduler's queue) and re-arm.
    rec.Status = models.ProcessStopped
    rec.FinishedAt = nowMillis()
    // ... update ...
    notifyScheduler()
    return rec, nil
}

if _, err := fuse.StopProcess(ctx, rec.PID, rec.ResourceURL); err != nil {
    // ... the interesting branch ...
}
```

And when the engine refuses the stop, the backend does **not** guess:

```go
// process.go — abbreviated
trace, terr := fetchInfraTrace(ctx, rec.PID)
switch {
case terr != nil:
    // Could not determine the status — do NOT guess. Surface the original error.
    return rec, fmt.Errorf("stop process %d (pid=%s): %w", indexID, rec.PID, err)

case trace.Finished():
    // Infra recorded a finish_at: the run really did end. Reconcile the row to what
    // the trace says and treat the stop as a satisfied no-op.
    rec.Status = finishStatus(trace.Data.Status)
    rec.FinishedAt = trace.FinishAt
    rec.DurationMs = trace.Data.DurationMs
    // ... update ...
    return rec, nil

case trace != nil:
    // start_at set, finish_at 0 — infra still sees it live. The engine and infra
    // disagree, but we have positive evidence it is up: do not clobber the row.
    return rec, fmt.Errorf("stop process %d (pid=%s): infra trace shows it still running: %w",
        indexID, rec.PID, err)

default:
    // No trace anywhere AND the engine has forgotten the pid: no evidence it is
    // alive. Best-effort mark it stopped so the row does not sit `running` forever.
    rec.Status = models.ProcessStopped
    rec.FinishedAt = nowMillis()
    // ... update ...
    return rec, nil
}
```

There is also a decode quirk worth knowing, because it looks like a bug and is not. A
graceful stop answers `{"data":"OK"}` — a status string where `ProcessResponse` expects
`{PID}` — so `StopProcess` surfaces a sonic `MismatchTypeError`. That decode error is the
*only* way a 2xx-but-non-pid body is reported, so it unambiguously means **accepted**:

```go
// process.go
var mismatch *decoder.MismatchTypeError
if errors.As(err, &mismatch) {
    rec.Status = models.ProcessStopped
    rec.FinishedAt = nowMillis()
    // ... update ... — the run's own finish will reconcile later
    return rec, nil
}
```

> **The general principle:** with three independent back-channels and no transactions
> across them, a backend's job is to know *which of them it has evidence from* and to act
> only on evidence. "I could not tell" and "it is not running" are different answers, and
> conflating them is how rows get silently wrong.

---

## 10. Build it in this order

Each step is independently testable, and each one is useful before the next exists.

| # | Step | You can now |
| --- | --- | --- |
| 1 | `RetrieveFlow` over a hardcoded flow (copy `backend_sample.go`) | run a flow |
| 2 | `RetrieveContext` / `UpdateContext` over a map | run a flow that computes something |
| 3 | The compiler hook, replacing the hardcoded flow | run *your users'* flows |
| 4 | The process row + `StartWorkflow`, with meta and a self-minted pid | list runs |
| 5 | `SubscribeProcessEvents` | see runs end |
| 6 | `storeRunOutcome` — `_errors` half only | see what a run hit |
| 7 | An extrinsic handler answering `StopHereResponse` | park a run |
| 8 | `storeRunOutcome` — `_sched` half — plus `loadResumeSnapshot` and `WithResume` | **resume a run correctly through a join** |
| 9 | The scheduler | park on a clock |
| 10 | `fetchInfraTrace` reconciliation | survive a lost finish |

Steps 1–3 are the [contract](the-backend-contract.md). Steps 4–10 are the wire, and they
are all consequences of `Exec` returning a pid.

---

## 11. Failure modes, and what they look like

| Symptom | Cause | Fix |
| --- | --- | --- |
| Run stalls, then times out at the first node | a handler returned without `msg.Respond` | respond on every path, errors included |
| Run executes perfectly, persists nothing | `UpdateContext` bailed — no context row | materialize on the `RetrieveContext` miss (§2) |
| A service handler sees an empty pid | pid not minted before dispatch | `uuid.NewString()` → meta **and** `WithPID` (§3) |
| Rows stuck in `running` forever | `proc.finish` lost, or the row was written after `Exec` | write the row before `Exec`; reconcile via infra trace (§9) |
| A resume hangs at a join until timeout | no snapshot seeded | check `loadResumeSnapshot` used the **producing** pid (§5) |
| A resume seeds the wrong state | snapshot keyed by `contextId`, clobbered by an overlapping run | key it by pid, on the process row (§5) |
| A resume silently starts blank | `flowSig` drift — the flow was edited | expected; surface the resume-skipped log to the user |
| Resume enters only one of several branches | start set collapsed to `StartNodeID` | keep the full set in `RecordMeta[startNodeIds]` (§4) |
| A scheduled row fires twice | dispatched before being claimed | flip to `running` first (§8) |
| A run reports success but the output is wrong | node errors — a flow does not stop for them | render `_errors.count`, split `node` from `system` (§6) |
| `Count` and `len(Items)` disagree | the 100-entry cap on a cascading run | render `Count` (§6) |

---

## 12. What the wire buys

Set the code in this chapter against what it delivers:

**You wrote:** three methods, a process table, an event subscriber, a snapshot
load/store pair, a timer loop, and a trace lookup.

**You got:** durable graph execution across crashes and restarts; parallel branches with
correct joins; parks that cost nothing while they wait, resumable hours or days later
through a join, gated against flow drift; a per-run error ledger that distinguishes the
author's mistakes from yours; and horizontal scale by starting a second engine.

> The engine is the moat you did not have to dig. The wire is the drawbridge, and this
> chapter is all of it.

---

## Next

- **[Observing a run](observing-a-run.md)** — the other half of the event stream: turning
  `inflow.event.log` back into movement on your canvas.
- **[Build your own workflow product](build-a-workflow-product.md)** — the assembly
  instructions this chapter is the hardest step of.
- **[How FloMorphic was built](../04-flomorphic/how-it-was-built.md)** — the same code as a
  product case study.

**Source material:** `flomorphic-api/inflow/` — `wire.go`, `process.go`, `events.go`,
`scheduler.go`, `continue.go`, `hitl_resume.go`, `hitl_context.go`, `trace.go`, `port.go`;
`flomorphic-api/models/process.go`; `inflow-fusion/backend_sample.go`,
`inflow-fusion/models/core.go`, `inflow-fusion/inflow/process.go`;
`fractal-core/engine/resume.go`, `fractal-core/engine/errstamp.go`,
`fractal-core/models/errors.go`, `fractal-core/models/core.go`.

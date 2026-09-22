# The backend contract

> Your backend answers three questions. That is the whole obligation.

## The interface

```go
type IInflowService interface {
    RetrieveFlow(msg *nats.Msg)    // what is this flow?
    RetrieveContext(msg *nats.Msg) // what is this run's context?
    UpdateContext(msg *nats.Msg)   // take this updated context
}
```

Implement those three methods over whatever storage you like, and your backend is a
first-class participant in the platform.

The engine subscribes nothing from your database. It has no driver, no schema, no
migration, no connection string. It asks, over NATS:

| Subject template | Direction | Handler | Purpose |
| --- | --- | --- | --- |
| `inflow.req.flow.get.{flowId}` | engine → backend | `RetrieveFlow` | return the compiled `models.Flow` |
| `inflow.req.context.get.{contextId}` | engine → backend | `RetrieveContext` | return the current `models.ContextDoc` |
| `inflow.req.context.set.{contextId}` | engine → backend | `UpdateContext` | persist an updated `ContextDoc` |

These defaults live in `svcHandler.DefaultGetFlowSvc` / `DefaultGetContextSvc` /
`DefaultSetContextSvc`, and can be replaced with your own `svcHandler.SvcTopic` patterns.

**Why request/reply instead of a shared database:** it is the single decision that makes
one engine reusable across unrelated backends. Each one answers three questions however it
stores the data — SQLite, Postgres, an in-memory map in a test — and the engine cannot
tell the difference.

---

## Booting a backend

```go
func main() {
    svc := &MyInflowService{db: myDB} // implements IInflowService

    err := inflow.InitBackend(
        context.Background(),
        svc,
        inflow.WithInfraApi(os.Getenv("INFLOW_INFRA_API")),
    )
    if err != nil {
        log.Fatal(err)
    }
    select {}
}
```

`InitBackend` does four things:

1. Fetches NATS credentials for the `inflow` account from Infra's REST API.
2. Connects to NATS with them.
3. Subscribes to the three default subjects above.
4. Calls `ReloadResources` — asks Infra for the list of registered engine instances and
   loads them into a round-robin pool.

Two environment variables bind everything:

| Env var | Purpose |
| --- | --- |
| `INFLOW_INFRA_API` | Base URL of the Infra REST API (e.g. `http://localhost:8022`) |
| `INFLOW_INFRA_JWT_SECRET` | HMAC secret shared with Infra, used to mint the bearer token |

There is no login flow. **Possession of the secret is the credential.** The SDK mints a
local HS256 JWT with the claim `{"admin": true}` and sends it as
`Authorization: Bearer <JWT>` on every REST call.

> **The rule that matters most:** never dial NATS yourself. Not `nats.Connect`, not a
> hand-built `*nats.Conn`, not a hand-assembled `inflow.v1.*` subject. The SDK owns a
> single pooled, credentialed connection per account/space; it mints the right scoped
> credentials, builds the correct subjects, and handles isolation, timeouts and
> reconnection. Bypassing it means duplicated credential logic, leaked connections, and
> plugins reached on the wrong account. The only NATS you touch directly is the callback
> surface the SDK hands you — the `*nats.Msg` in your `IInflowService` methods and the
> `nats.Header` in a `svcHandler` handler.

---

## Starting a run

```go
proc := inflow.NewProcess(startNodeId,
    inflow.WithFlowId(flowId),
    inflow.WithContextDocument(contextId),
)
resp, err := proc.Exec(ctx)
// resp.Data.PID — the process id
```

`Exec` picks the next engine instance from the round-robin pool and POSTs a
`ProcessRequest` to `{engine}/engine`.

```go
type ProcessRequest struct {
    Context      ContextTopicsPattern // subjects the engine uses for this run's context
    Flow         FlowEngine           // subject to fetch the flow definition
    PID          string               // process id, UUID-generated if empty
    StartNodeIds []string             // where the run enters
    Settings     Settings             // timeouts, node execution cap
    Meta         map[string]string    // free-form; also fills subject templates
    Resume       bool                 // continue an earlier run over the same context
}

type Settings struct {
    RequestTimeOut   int64  // per NATS request, seconds (default 5)
    ExecuteTimeOut   int64  // whole process, seconds (default 3600)
    ProcessNodeLimit uint16 // safety cap on nodes visited (default 500)
}
```

Stop a run early:

```go
inflow.StopProcess(ctx, pid, resourceUrl)
```

### Start nodes

`StartNodeIds` is where the run enters, and the arity is meaningful:

- **A single id** is trigger semantics: that node does **not** itself run — its successors
  are queued, and it is recorded as done.
- **Multiple ids** (or any resume) queue the named nodes directly, so they **do** run as
  tasks.

---

## Choosing an engine

`InitBackend` loads every registered engine instance into a round-robin pool, and each
`Exec` takes the next one. For most deployments that is the whole scaling story: start a
second Fractal, and load spreads without a code change.

When you need a specific instance — a Fractal with a particular tag, a GPU host, a
debugging target — **pin** it:

| Call | Effect |
| --- | --- |
| `inflow.PinResource(nameOrUrl)` | every dispatch uses only that resource |
| `inflow.UnpinResource()` | back to round-robin |
| `inflow.AddResource(...)` | add an instance by hand |
| tag a resource `inflow.PinResourceTag` (`pinned-resource`) | pin it declaratively |
| `inflow.GetResourceCandidList()` | the current live pool |
| `inflow.ReloadResources()` | re-ask Infra for the registry |

A pin lasts until `UnpinResource` or the next `ReloadResources`.

---

## The context document

```go
type ContextDoc struct {
    Data   string         `json:"data"`   // opaque to the SDK — typically JSON per-node scope data
    Header map[string]any `json:"header"`
}
```

`Data` is yours. The SDK does not parse it; conventionally it is JSON, addressed by
JSONPath (`$.OPA`, `$["doc appendix"]`) — the tree that nodes read and write under their
`Scope`.

`Header` is **per-context memory that survives across runs**. Store it and return it
verbatim. It is a `map[string]any`, so new reserved keys stay forward-compatible. Two
kinds of entry live there, both engine-managed and both opaque to you:

- **Node registry** — one entry per call site (keyed `"<scope>:<nodeId>"`), holding what a
  node left for its next run. A plugin's `jobId` lives here, which is how a job that
  outlived the process is reconnected; it is handed back to the plugin on its next
  handshake as `_registry`.
- **Traversal snapshot** (`_sched`) — the previous run's completed node generations and
  join watermarks, written on every finish, read back only on a resume.

Neither needs anything from you beyond storing the header as-is.

---

## Resuming a run

This is how a flow waits for a human for three days.

`Resume` continues an earlier run over the **same context** (same `contextId`, typically
the same `PID`). The pattern:

1. A gate node — *continue after*, *human in the loop*, *wait for event* — returns
   `{"_cmd":"stop"}`, which deactivates every outgoing edge. The process ends.
2. Your backend schedules whatever it likes. A timer, a webhook, a person clicking
   *Approve*. **The engine knows nothing about this** and is not running during it.
3. Later, you start a new process whose `StartNodeIds` are the **successors** of that
   terminated node, with `inflow.WithResume()`.

```go
inflow.NewProcess(successorIds,
    inflow.WithFlowId(flowId),
    inflow.WithContextDocument(contextId),
    inflow.WithResume(),
).Exec(ctx)
```

**Why the flag matters.** Node results already live in the context document, so a plain
restart continues fine on a linear path. But a *join* downstream of the resume point
checks the completion **generation** of its dependencies, not their data — and a
dependency that completed before the stop is not re-run from the resume point. Without
`Resume`, that join waits until the process times out. With it, the engine seeds the
traversal snapshot the previous run left in the context header, so the join sees its
already-completed dependencies and fires.

`Resume` is additive: omit it and the request behaves exactly as before. It takes effect
only when a matching snapshot is present **and** the flow definition is unchanged since
the snapshot was taken — the engine gates on a structural signature and falls back to a
blank continue on drift. An edited flow cannot resume into a stale plan.

---

## Exposing domain logic: extrinsic services

Beyond the three questions, you can expose any business logic as a callable step:

```go
svcHandler.ImplHandlerOnSubject(
    "exports_db",                                       // logical name
    svcHandler.SvcTopic("my.internal.svc.persist.*"),   // subject pattern
    func(header nats.Header, data []byte) ([]byte, error) {
        table := strings.Split(header.Get("recv_subject"), ".")[4]
        // ... your domain logic ...
        return []byte(`{"status":"saved"}`), nil
    })
```

Internally this subscribes on the pattern with `{param}` placeholders replaced by `*`, and
sets a `recv_subject` header to the exact subject the message arrived on — which is how
**one handler serves many logical topics**.

Handlers are tracked in a process-local registry (`svcHandler.GetSvc` / `GetAllSvcs`) keyed
by the logical name. That is what lets a compiler hook resolve a friendly name from the
canvas back to a subject pattern, instead of hardcoding NATS subjects in the frontend.

---

## The REST surface, for reference

**Your backend → Infra:**

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/account/inflow/cred` | fetch the NATS credential for the `inflow` account |
| `GET` | `/inflow/resource?per_page={n}` | list registered engine instances |
| `GET` | `/account/id/{accountId}` | fetch an account, used to mint scoped plugin credentials |

Envelopes are `{"data": ..., "error": ...}`; a non-null `.Error` or non-2xx status is a
failure.

**Your backend → an engine instance:**

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `{engineUrl}/engine` | start a process (`ProcessRequest` body) |
| `POST` | `{engineUrl}/engine/stop/{pid}` | stop a running process |

`engineUrl` comes from the pool. Missing scheme is prefixed `http://`; missing port is
suffixed with `models.INFLOW_REST_PORT` (`9001`).

---

## Next

- **[Compiling a canvas](compiling-a-canvas.md)** — the shipped Vue Flow / React Flow
  compiler, worked end to end.

**Source material:** `inflow-fusion/README.md`, `inflow-fusion/docs/infra.md`,
`inflow-fusion/docs/architecture.md`, `inflow-fusion/docs/plugin-svc-calls.md`.

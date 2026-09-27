# Tag routing — one mechanism, three deciders

> Branching is not a node type. It is one runtime mechanism that every node can use.

## The rule

> A node emits a **list of tags** when it finishes. Every outgoing transition carries the
> tags it accepts, fixed at compile time. The engine deactivates every outgoing edge, then
> re-activates only the ones whose tags match — and follows those.

```
        rule result / service reply / plugin command
                     │  ["approve"]
                     ▼
   ┌──────────┐  Next{Tags:["approve"]}  ──► fires
   │  node    │  Next{Tags:["reject"]}   ──► Active = -1
   └──────────┘  Next{Tags:["escalate"]} ──► Active = -1
```

The route is chosen at **run time**. The set of routes that could *ever* fire was drawn at
**author time**. That is the whole "contract" idea, and it is the platform's answer to the
question everyone asks about agents:

> The decision is dynamic. The possibilities are static and inspectable.

A model, a policy engine or a service can choose among the ports you drew. **None of them
can invent a port you did not draw.**

---

## Who can emit tags

Three primitives, with **identical** filter semantics. Contract is the common case, not
the only one.

| Decider | How the tags arrive | Evaluated where |
| --- | --- | --- |
| **Contract** | the JS/OPA rule's result *is* the tag list | in-engine |
| **Extrinsic** | the service reply carries `_cmd: "next_tags"` + `_next_filter` | your backend |
| **Plugin** | the running job sends the `next_tags` command | an external process |

All three can also **declare** their ports at author time — see
[declared outbound ports](#declared-outbound-ports--author-time-tag-routing) below. The
runtime mechanism is unchanged; the declaration only makes the possibilities visible on the
canvas before anything runs.

### Contract — a compiled rule decides

```go
rule := nodes.NewJsRuleLogicNode(
    nodes.WithContractLogicCode(`input.amount > data.threshold ? ["escalate"] : ["approve"]`),
    nodes.WithContractConditions(map[string]any{"threshold": 10000}),
)
```

Deterministic, outside any model, over the node's scope. Cheap and auditable. This is
where policy belongs.

### Extrinsic — your backend decides

`svcHandler` ships the response helpers. The engine applies the command to the node's
edges *before* writing the reply into context.

```go
svcHandler.ImplHandlerOnSubject("risk", svcHandler.SvcTopic("svc.risk.check"),
    func(header nats.Header, data []byte) ([]byte, error) {
        if risky(data) {
            return svcHandler.FilterNextResponse(map[string]any{"score": 91}, []string{"escalate"})
        }
        return svcHandler.FilterNextResponse(map[string]any{"score": 12}, []string{"approve"})
    })
```

| Helper | Wire form | Effect |
| --- | --- | --- |
| `FilterNextResponse(data, tags)` | `{..., "_cmd":"next_tags", "_next_filter":["escalate"]}` | keep only edges tagged `escalate` |
| `StopHereResponse(data)` | `{..., "_cmd":"stop"}` | deactivate **every** edge — the branch ends here |
| plain bytes | no `_cmd` | no-op; edges stay as compiled |

The keys are `models.SvcCmdResposeKey` (`_cmd`) and `models.SvcCmdResponseNextFilterKey`
(`_next_filter`); the commands are `models.CmdNextFilter` and `models.CmdStop`. A missing
or malformed command is **deliberately a no-op** — a handler that knows nothing about
routing cannot accidentally break a flow.

`StopHereResponse` is worth noting for a second reason: it is the mechanism behind a
*pause*. A gate node stops the branch, your backend schedules whatever it likes, and later
a new process resumes from that node's successors. See
[the backend contract](the-backend-contract.md#resuming-a-run).

### Plugin — an external process decides, mid-job

A plugin node is a live process speaking `inflowv1`. Routing is just another job command,
sent while the job is still running:

```go
p.AddAction(sdkv1.Action{Method: "classify", RequestHandler: func(job sdkv1.Job) {
    job.CmdNextFilter([]string{"escalate"})   // subject: <job>/next_tags
    job.Done(map[string]any{"reason": "amount over limit"})
}})
```

**This is what makes a model node route the diagram.** An LLM plugin binds its
functions/tools to the node's tagged output ports. When the model answers with a tool
call, the plugin translates that choice into `CmdNextFilter([...])`, and the engine fires
only that port. The model picks among ports you drew; everything downstream of each port
is already defined.

The same is true of an MCP node, or any plugin that chooses its own continuation.

<a id="the-exception-port"></a>
#### The exception port — fail, and still route

`CmdNextFilter` and the terminal commands are not mutually exclusive. A plugin can route to a
reserved tag and **then** end the job as failed:

```go
if top.Score < minConfidence {
    job.CmdNextFilter([]string{"_exception"})
    job.DoneWithErrorData("no answer cleared min_confidence", map[string]any{
        "top_answer": top.Option, "score": top.Score,
    })
}
```

The edge tagged `_exception` still fires, so the branch continues to a handler the author drew
— route to a fallback model, escalate to a person, log and stop — while the node is recorded
as an error in the run. **A node that cannot decide says so, loudly, and the graph decides what
happens next.** This is the pattern behind FloMorphic's Jev node, and it is why a decider node
never has to silently pick a low-confidence answer.

---

## Declared outbound ports — author-time tag routing

Tag routing chooses an edge at run time. `Action.Outbound` declares the *possibilities* at
author time, so the choice is visible before anything runs:

```go
p.AddAction(sdkv1.Action{
    Method: "classify",
    Outbound: []sdkv1.OutboundPort{
        {Title: "Escalate", Tags: []string{"escalate"}, Description: "amount over limit"},
        {Title: "Approve",  Tags: []string{"approve"}},
    },
    RequestHandler: handler,
})
```

The declaration is served on `@actions`, so the host renders **one output port per entry** —
labelled by `Title`, explained by `Description` — and stamps every edge drawn from that port
with its tags. At run time the handler still calls `CmdNextFilter(port.Tags)`; edges carrying
other tags are skipped. `Action.Tags` carries the same idea for grouping, with a reserved
`class` key that lets one binary host several logical products (a Google plugin bundling Docs,
Sheets and Calendar actions) and tell them apart on the list.

It is deliberately **not essential**: the same branching is achievable by following this node
with a Contract that fans out on the committed result. Declaring `Outbound` only keeps the port
topology, its tags and its documentation on the action instead of wiring them by hand on the
canvas. Leave it nil for a single-output action.

---

## Semantics, exactly

- **Deactivate-then-match.** All `Next` entries are set `Active = -1`; an entry is
  re-activated if *any* of its `Tags` is in the emitted list.
- **Multiple tags = fan-out.** `["a","b"]` fires every edge tagged `a` or `b`, **in
  parallel**. Join them again with a Void node using `Depends`.
- **No match = dead end.** If nothing matches, nothing continues on that branch — which is
  why an untagged catch-all "else" edge is worth drawing.
- **Untouched by default.** A linear node that emits no tags follows its `Next` exactly as
  compiled. Tagging is opt-in, per node.
- **It is recorded.** The tags that did the selecting are kept as the *select reason* and
  emitted with the `edge.select` event — so a run explains not just which edge fired but
  **why**. See [Observing a run](observing-a-run.md).
- **Not on Code.** Code writes a value into context; it does not route.

---

## Why this is a primitive and not a feature

Every workflow builder eventually ships a drawer of comparison nodes: greater than, less
than, contains, is empty, matches, between. Inflowenger ships **none** of them, and ships
the primitive they are all made of.

A product author declares a node in their palette — *Greater than*, two fields, two output
ports — and the compiler hook turns it into a Contract whose rule is a small JS or Rego
body built from those fields:

```go
// one case per palette node, in the compiler hook
case "op_gte":
    n := nodes.NewJsRuleLogicNode(
        nodes.WithContractLogicCode(`input[data.field] >= data.value ? ["true"] : ["false"]`),
        nodes.WithContractConditions(map[string]any{
            "field": data["field"], "value": data["value"],
        }),
    )
    node.Type = models.RuleNodeType
    node.Contract = &n.ContractRule
```

The runtime never learns what "greater than" means. The person on the canvas never sees a
line of JavaScript — they see two fields and a `true`/`false` output. Add fifty operators
and the engine is unchanged.

> **Contract is the base class for decision nodes.** New operators are *authored* at
> compile time, not *implemented* in the engine.

The argument extends past comparisons:

| Product-level concept | What it actually is |
| --- | --- |
| A policy gate | a Rego **Contract** |
| A guardrail | a Rego or JS **Contract** |
| An approval gate | an **Extrinsic** that answers with tags |
| A retry-or-fail branch | a **Contract** over the previous node's output |
| An agent's next step | a **Plugin** that answers with tags |
| A loop | a backward edge, plus a **Contract** that decides when to stop |

One mechanism. Three deciders. No runtime changes.

---

## The loop, since it always comes up

There is no loop node. A loop is an edge pointing backward plus a condition:

```
   ┌──────────────────────────────────────────┐
   │                                          │
   ▼                                          │
 [ do work ] ──► [ Contract: done? ] ──"no"───┘
                        │
                      "yes"
                        ▼
                   [ continue ]
```

Because the whole process iterates over one durable context object, the loop survives a
wait, an approval or a crash. The iteration state is not in a variable on a stack — it is
in the context document, which your backend persists. A `ProcessNodeLimit` in the run
settings (default 500) is the safety cap that stops a runaway.

---

## Next

- **[The backend contract](the-backend-contract.md)** — the three questions your backend
  answers, the process lifecycle, and how a run is paused and resumed.

**Source material:** `inflow-fusion/docs/routing.md`, `inflow-fusion/docs/nodes/contract.md`,
`inflow-fusion/docs/nodes/extrinsic.md`, `inflow-fusion/docs/nodes/plugin.md`.

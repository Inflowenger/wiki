# The primitive node reference

> Six node types. There is no seventh, and this chapter is the argument for why.

The engine executes exactly one thing: a flat `map[string]*models.Node`. Every node in
that map is one of six types. Everything a product's palette offers — an HTTP node, a
send-email node, a wait-for-approval node, a vector search, a model call — is one of these
six with specific configuration.

---

## The set

| Primitive | `NodeType` | Role in one line | Compiles away? |
| --- | --- | --- | --- |
| **Void** | `VoidNodeType` | Structure — start markers, joins, barriers, dead ends | yes |
| **Code** | `CodeNodeType` | Computation — run JS or OPA/Rego over scoped context | yes |
| **Contract** | `RuleNodeType` | Decision — evaluate a rule, emit tags, fire matching branches | yes |
| **Extrinsic** | `ExtrinsicNodeType` | Call a service *you* own, over one NATS request/reply subject | yes |
| **Plugin** | `PluginNodeType` | A live external process — the open-ended escape hatch | **no** |
| **GoTo** | `GoToNodeType` | Composition — jump into another flow and return | yes |

Five of the six are compiled artifacts: after compilation nothing of your vocabulary
remains, only a rule body, a subject, or a jump target. **Plugin is the exception.** It
never compiles away, because it is a real, long-lived process with its own connections,
its own background loops and its own configuration UI. That is why it is the richest node
in the system, and why Part III is devoted to it.

---

## The shared shape

Every node, whatever its type, has the same skeleton:

```go
type Node struct {
    ID        string         `json:"uuid"`
    Type      NodeType       `json:"type"`
    Title     string         `json:"title"`
    Key       string         `json:"key"`      // where this node's output is written into context
    Scope     string         `json:"scope"`    // a JSONPath into the context it reads/writes under
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

type Next struct {
    FlowId string         `json:"flowId"` // defaults to the current flow
    NodeId string         `json:"nodeId"`
    Tags   []string       `json:"tags"`   // which Next entries a tag emission selects
    Active int8           `json:"active"` // 0: active, -1: inactive
    Meta   map[string]any `json:"meta"`
}
```

Exactly one of `Code` / `GoTo` / `Extrinsic` / `Plugin` / `Contract` is populated, matching
`Type`.

### The three universal fields

Before you pick a type, three things are already true of any node in any vocabulary:

| Field | The question it answers |
| --- | --- |
| **Title** | What is this called? |
| **Key** | Where does its output go in the context? |
| **Scope** | What slice of context does it see? |

A product's canvas can present these however it likes; they map straight through. This is
why an arbitrary bespoke-looking node is structurally identical to every other node.

---

## Void — structure

A no-op. It runs nothing and writes nothing.

```go
n := nodes.NewVoidNode(nodes.WithUniqId[*nodes.VoidNode]("start"))
```

Three uses carry it:

- **Start markers.** The entry point of a flow.
- **Dead ends.** A branch that deliberately stops.
- **Joins.** This is the important one. Combined with `Depends`, a Void node is a
  synchronisation barrier: when a node fans out into parallel branches, a Void listing all
  of them in `Depends` holds until every one has finished, merges their results into the
  shared context, and continues once. `Promise.all` for a graph.

## Code — computation

Run a body of JavaScript or OPA/Rego against the node's scope, write the result to `Key`.

```go
js  := nodes.NewJsNode(`input.a = input.b * input.b; input`)
opa := nodes.NewOpaNode(`result = {"f": 60}`, "result",
    nodes.WithCriteriaData(map[string]any{"threshold": 10}))
```

For JS, the scoped context arrives as `input` and the expression's value is the output.
For OPA, the scope is `input`, the condition key/values are `data`, and a named result
binding is extracted.

Code **does not route.** It computes and writes. That is the only difference between Code
and Contract.

## Contract — decision

Evaluate a rule whose result *is a list of tags*. The engine deactivates every outgoing
transition, then re-activates only those whose `Tags` match.

```go
rule := nodes.NewJsRuleLogicNode(
    nodes.WithContractLogicCode(`input.amount > data.threshold ? ["escalate"] : ["approve"]`),
    nodes.WithContractConditions(map[string]any{"threshold": 10000}),
)
```

Deterministic, evaluated in-engine, outside any model. This is the branching node, the
policy node and the guardrail node — all three are the same primitive with a different
rule body. Full mechanics in [Tag routing](tag-routing.md).

## Extrinsic — reach your own system

Publish to a NATS subject your backend owns; the reply becomes the node's output.

```go
ext := nodes.NewExtrinsicSvcNode("my.internal.svc.persist.orders")
```

On the backend side you register the handler once:

```go
svcHandler.ImplHandlerOnSubject("exports_db",
    svcHandler.SvcTopic("my.internal.svc.persist.*"),
    func(header nats.Header, data []byte) ([]byte, error) {
        table := strings.Split(header.Get("recv_subject"), ".")[4]
        // ... write the row ...
        return []byte(`{"status":"saved"}`), nil
    })
```

Note the wildcard. **One registration serves many logical capabilities** — the handler
recovers the concrete parameter from the `recv_subject` header, so
`svc.persist.orders` and `svc.persist.tasks` are two nodes on the canvas and one door in
your backend.

This is the cheapest way to make existing business logic available to a graph: a handful
of lines, no new process, nothing moved.

## Plugin — the open end

Hand off to a live external process speaking `inflowv1` over NATS.

```go
plugin, _ := nodes.NewPluginNode("jira", nodes.WithIdleWaitMinutes(30))
```

The plugin is not part of the engine and not part of your backend. It is your process,
deployed on your cadence, with narrowly-scoped credentials that let it see only its own
subjects. It carries its own configuration form, streams progress back to the canvas,
reads and writes the running context by JSON path, and can route its own outbound ports.

Everything about this node is [Part III](../03-plugins/).

## GoTo — composition

Jump into another (or the same) flow like a subroutine, and come back.

```go
g := nodes.NewGotoNode()
g.From("flow-a", "node-3")
g.To("flow-a", "node-8")
```

This is reuse. A sub-flow authored once — an approval procedure, an enrichment pipeline —
is called from many places. Because the target may be the *same* flow, GoTo is also one of
the ways a cycle is expressed.

---

## Why the set is closed

The design argument is an axis argument. A workflow builder needs four things, and each
has exactly one home:

| Axis | Covered by |
| --- | --- |
| Arbitrary computation | **Code** — JS or OPA over the scoped context |
| Decision and control flow | **Contract** (tagged branching), **GoTo** (reuse), **Void** (structure) |
| Reaching your own system | **Extrinsic** — one request/reply subject into your backend |
| Reaching the outside world, long-running, stateful, event-driven | **Plugin** — a live process |

Those four axes span the space. Anything you can name falls into one of them:

- *"I need an HTTP node."* → Plugin (it holds connections), or Extrinsic (if your backend
  already makes the call).
- *"I need a wait-for-approval node."* → Extrinsic that parks the run, plus a resume.
- *"I need a loop."* → an edge pointing backward, plus a Contract deciding when to stop.
- *"I need parallel fan-out and a join."* → tag emission with multiple tags, plus a Void
  with `Depends`.
- *"I need a sub-workflow."* → GoTo.
- *"I need a node that calls a model and picks its own next step."* → Plugin that emits
  tags.

The claim is not that six types are *aesthetically* sufficient. It is that the four axes
are exhaustive for this problem, and each axis has a primitive.

---

## What a node with N outputs really is

This is the reduction people find least obvious, so it is worth stating plainly.

```
 designer's mental model            primitive reality
 ───────────────────────            ─────────────────────────────────
 node with 1 output          ──►    any linear primitive + one Next
 node with N labelled        ──►    N tagged Next entries; whichever
   outputs / conditions             primitive runs emits matching tags
 node that reads "inputs"    ──►    a scope/context read inside the node
 node with a config form     ──►    form values stored in your node data,
                                    read by the compiler hook
```

An **input handle** on a canvas is not data. It means only *"execution can arrive here."*
What a node actually reads is governed by `Scope` and the context, not by the wire. So an
arbitrary number of imagined "inputs" all reduce to: this node runs, and it reads context.

**Output handles** are what become transitions. Each edge from an output handle compiles
to one `Next` entry, carrying that handle's tags.

"Three outputs: success / retry / reject" is not three node types. It is one node with
three tagged `Next` entries.

---

## Building a flow by hand

Most flows are compiled, but the structure is plain enough to write directly — which is
what tests and fixtures do:

```go
flow := models.Flow{
    UUID: "f-123",
    Nodes: []models.Node{
        {ID: "n0", Type: models.VoidNodeType, Next: []models.Next{{NodeId: "n1"}}},
        {ID: "n1", Type: models.CodeNodeType,
         Code: &models.CodeRule{Lang: "js", LogicRule: "input"}},
    },
}
flow.ValidateNext() // fills FlowId on any Next entry that omitted it
```

---

## Next

- **[Tag routing](tag-routing.md)** — the one cross-cutting mechanism, and the three
  primitives that can drive it.

**Source material:** `inflow-fusion/docs/nodes.md`, `inflow-fusion/docs/nodes/*.md`,
`inflow-fusion/models/flow.go`.

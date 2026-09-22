# The compiler seam

> One function is the entire difference between "a workflow engine" and "*your* workflow
> product."

## The problem this solves

A workflow product has a vocabulary. n8n has *Webhook*, *IF*, *Set*, *HTTP Request*.
GitHub Actions has *job*, *step*, *uses*, *needs*. FloMorphic has *LLM*, *MCP*, *Rule*,
*Human in the Loop*. Venapce has *collect fleet*, *promote to issue*.

Those vocabularies are the product. They are what the user thinks in, what the
documentation teaches, what the sales page shows. They are also *completely different from
one another*, and none of them is a thing an execution engine could sensibly be built
around.

The naive resolution is to make the engine understand the vocabulary — a node type in the
editor becomes a node type in the runtime, one for one. It works, briefly. Then the
fiftieth node type ships, the engine has fifty branches in a switch statement, every new
product feature is a runtime release, and two products can never share a runtime because
their vocabularies collide.

Inflowenger resolves it the other way. **The engine's vocabulary is fixed and tiny. Your
vocabulary is unlimited and lives entirely in a compiler.**

---

## The seam

```
  AUTHOR              SAVE                 COMPILE                  EXECUTE
  ────────            ────────             ─────────────            ─────────────
  a user builds       your format is       a per-node HOOK          a Fractal fetches
  something in        persisted            lowers each of your      the node map and
  your vocabulary     VERBATIM             node types to one of     walks it, asking
  (canvas, YAML,      (no translation      six primitives;          your backend for
  form, API call)     on save)             edges/transitions        flows and context
                                           become `Next` entries    as it goes
       ↑                   ↑                       ↑                        ↑
     YOURS               YOURS               YOURS (one function)      ALREADY WRITTEN
```

Four stages. You own three of them. The fourth — the one that is months of work — you do
not write.

The critical property is in stage two: **your format is persisted verbatim.** The
frontend does no translation. A Vue Flow canvas saves a Vue Flow graph, positions and
viewport included. A YAML product saves the YAML. Whatever round-trips back into your
editor is exactly what left it, because compilation is a *separate*, later, one-way step.

That separation is why the editor never has to be lossy, and why you can change what a
node type compiles to without migrating a single saved document.

---

## The contract

A compiler in `inflow-fusion` is three things. There is deliberately no Go interface
forcing this shape — each compiler lives in its own subpackage — but every one follows it.

### 1. A graph type: your external representation

```go
type VueFlow struct {
    Nodes []VueFlowNode
    Edges []Edges
}
```

Typically `Nodes` + `Edges`, but that is convention, not requirement. If your source
format encodes flow some other way — a YAML map of steps with `needs:` dependencies, a
linear script with labels and jumps — that is your graph type. It does not have to be
nodes and edges at all.

### 2. A constructor that takes a hook

```go
cmpr := compiler.NewVueFlowCompiler(
    compiler.WithEachNodeFunc(myNodeBuilder),
)
```

The hook has this shape:

```go
func(YourNodeType) (*models.Node, error)
```

**This is the seam.** It is the only place in the entire system where knowledge of your
product's vocabulary exists.

### 3. A `Compile` method

```go
Compile(startNodeId string, graph X) (map[string]*models.Node, map[string]error)
```

It walks your graph from a start node, invokes your hook once per node, and — critically —
populates each resulting node's `Next` from your format's transitions. It returns the flat
node map the engine executes, plus per-node errors.

The only thing the engine actually cares about is that **`Next` ends up correctly
populated.** Position, dimensions, colours, labels, collapsed state, and every other
editor-only field in your graph type never leave your compiler package.

---

## What the hook actually does

A hook reads one of your nodes and returns one primitive. In practice it is a switch:

```go
func myNodeBuilder(vfn compiler.VueFlowNode) (*models.Node, error) {
    data := vfn.Data.(map[string]any)

    // The three universal fields, true of any node in any vocabulary.
    node := models.Node{
        ID:    vfn.ID,
        Title: data["title"].(string),
        Key:   data["key"].(string),   // where this node's output lands in context
        Scope: data["scope"].(string), // the JSONPath slice it reads/writes under
    }

    switch vfn.Type {
    case "code":
        node.Type = models.CodeNodeType
        n := nodes.NewJsNode(data["logic_rule"].(string))
        node.Code = &n.CodeRule

    case "contract":
        node.Type = models.RuleNodeType
        n := nodes.NewJsRuleLogicNode(
            nodes.WithContractLogicCode(data["logic_rule"].(string)),
        )
        node.Contract = &n.ContractRule

    case "http_request":                       // ← YOUR vocabulary
        node.Type = models.PluginNodeType      // ← lowered to a primitive
        n, _ := nodes.NewPluginNode("http", nodes.WithIdleWaitMinutes(5))
        node.Plugin = &n.PluginRule

    case "greater_than":                       // ← YOUR vocabulary
        node.Type = models.RuleNodeType        // ← also a primitive
        n := nodes.NewJsRuleLogicNode(
            nodes.WithContractLogicCode(`input[data.field] >= data.value ? ["true"] : ["false"]`),
            nodes.WithContractConditions(map[string]any{
                "field": data["field"], "value": data["value"],
            }),
        )
        node.Contract = &n.ContractRule
    }
    return &node, nil
}
```

Two rules keep a hook healthy:

1. **Use the `nodes.*` builders**, not hand-constructed rule structs. The builders keep
   compiled output consistent with what the engine and the rest of the SDK expect.
2. **Never set `Next` in the hook.** The compiler owns transitions. A hook that writes
   `Next` is fighting the walker.

---

## Why this makes the "six primitives" claim real

Look again at the `greater_than` case above. A user on the canvas sees a node called
*Greater than* with two fields and `true`/`false` outputs. They never see JavaScript.

The runtime never learns what "greater than" means. Add fifty comparison operators —
*less than*, *contains*, *is empty*, *matches*, *between* — and the engine is
**byte-for-byte unchanged**. Each one is a case in a hook and a card in a palette.

This is the difference between a feature and a primitive. Every workflow builder
eventually ships a drawer full of comparison nodes. Inflowenger ships none of them, and
ships the primitive they are all made of.

> **Decision nodes are *authored* at compile time, not *implemented* in the engine.**

The same argument runs past comparisons. A policy gate is a Rego `Contract`. An approval
gate is an `Extrinsic` that answers with tags. An agent's next step is a `Plugin` that
answers with tags. One mechanism — see [Tag routing](tag-routing.md) — three deciders, and
none of them required a runtime change.

---

## Two products, one runtime

The seam is also what lets unrelated products share an engine.

| | FloMorphic | A hypothetical CI product |
| --- | --- | --- |
| Authoring surface | Vue Flow canvas | `.ci.yaml` in a repo |
| Vocabulary | LLM, MCP, Rule, Doc Store, HITL | job, step, matrix, artifact |
| Saved verbatim as | `view_flow` (the raw graph) | the YAML text |
| Compiler | `compilers/vueFlow` + its hook | a YAML compiler + its hook |
| Compiles to | the same six primitives | the same six primitives |
| Runtime changes needed | **none** | **none** |

Neither product knows the other exists. Both run on the same Fractal instances. A plugin
written for one works on the other, because plugins target the protocol, not the product.

---

## Writing a new compiler

Reach for one when your source format is not a `{nodes, edges}` graph. A different graph
library (Cytoscape.js, Litegraph, a custom canvas), a non-visual DSL, or any struct that
describes steps and transitions between them.

1. Create a package under `compilers/<name>`.
2. Define structs for your external graph/node/edge shape.
3. Add a `NewXCompiler` constructor and a hook option, following the pattern above.
4. Implement `Compile`, resolving your format's transitions into `models.Next` entries
   (`NodeId`, `Tags`, `Meta`).
5. Reuse the `nodes.*` builders inside the hook.

[Compiling a YAML DSL](compiling-a-yaml-dsl.md) walks exactly this, end to end, for a
format with no graph library at all.

---

## Next

- **[The primitive node reference](node-primitives.md)** — the six things a hook can
  return, and the argument for why there is no seventh.
- [Compiling a canvas](compiling-a-canvas.md) — the shipped compiler, worked through.
- [Compiling a YAML DSL](compiling-a-yaml-dsl.md) — the same seam on a text format.

**Source material:** `inflow-fusion/docs/compilers/README.md`,
`inflow-fusion/docs/nodes/from-frontend.md`, `inflow-fusion/docs/routing.md`.

# Compiling a canvas — Vue Flow / React Flow

The shipped compiler, `compilers/vueFlow`, worked end to end. This is the concrete case
the [compiler seam](the-compiler-seam.md) describes abstractly.

> **It works for React Flow too.** Vue Flow's data model intentionally mirrors React
> Flow's: both describe a graph as `{ nodes, edges }`, where each node carries
> `id`/`type`/`data`/`position` and each edge carries
> `id`/`source`/`target`/`sourceHandle`/`targetHandle`. A React Flow export decodes into
> the same structs unchanged. The package name reflects which frontend it was first built
> against, not a dependency. Reach for a new compiler only if your export genuinely
> diverges — a custom edge shape, non-handle-based routing.

---

## The moving parts

```go
type VueFlow struct {
    Nodes []VueFlowNode
    Edges []Edges
}

type VueFlowNode struct {
    ID   string
    Type string // YOUR frontend's node type string, e.g. "code", "contract", "pluginNative"
    Data any    // YOUR frontend's arbitrary per-node form data
    // ...position/dimension fields, irrelevant to compilation
}

type Edges struct {
    ID           string
    Source       string
    Target       string
    SourceHandle string      // which output handle this edge left from
    TargetHandle string
    Data         EdgePayload // Tags []string, EdgeType string
    Label        string
}
```

Note what is *not* here: no list of allowed node types, no schema for `Data`. `Type` is a
free string and `Data` is `any`. The compiler is deliberately ignorant of your vocabulary.

---

## Using it

```go
cmpr := compiler.NewVueFlowCompiler(compiler.WithEachNodeFunc(myNodeBuilder))
nodeMap, errsByNodeId := cmpr.Compile(startNodeId, vueFlowGraph)
```

Two lines. The work is in `myNodeBuilder`.

---

## What `Compile` does

Starting from `startNodeId`, it walks the graph depth-first along outgoing edges:

1. Call your **hook** on the current `VueFlowNode` to get a `*models.Node`.
2. For every edge whose `Source` is this node, append a `models.Next`:
   - `NodeId` ← edge `Target`
   - `Tags` ← edge `Data.Tags`
   - `Meta` ← `{"edgeId": ..., "label": ..., "edgeHandle": edge.SourceHandle}`
3. Recurse into each `Next.NodeId` not already visited.
4. Return `map[nodeId]*models.Node`, plus any per-node errors your hook returned.
   Compilation stops walking further from a node once its hook errors.

Two consequences are worth drawing out.

**Branching needs no special handling in the compiler.** A frontend node with multiple
output handles — `success`, `error`, `else` — is just several edges with different
`SourceHandle` values and different tags. Which tagged `Next` entries actually fire is
decided at run time, by whichever primitive the node became. See
[Tag routing](tag-routing.md).

**`Data.Tags` is carried for *every* edge**, whatever primitive it leaves. The same wiring
therefore serves a Contract rule, an Extrinsic reply that answers with tags, and a plugin
job that routes itself. You draw ports once; who decides among them is a separate
question.

---

## The pipeline in a real product

The reference implementation is the **inflow-inspector** Vue app with `inspector-api`
behind it. FloMorphic's canvas follows the identical shape.

### 1. Author

The user drags nodes from a palette. Each palette entry is seeded with the right backend
fields — a type string plus a `baseData` shape — and configured through a drawer. Then
they connect handles.

The important detail is on the **Contract** node. Its default right-side output handle is
removed; instead the user adds any number of **handler** handles at the bottom, each with:

- a set of **tags** (`tag1, tag2, …`), and
- a colour (cosmetic).

When an edge is drawn from a handler, it **inherits that handler's tags** into
`edge.data.tags` on connect. That is the entire authoring surface for branching.

### 2. Save

`saveDiagram()` ships the **raw Vue Flow graph** — `{ nodes, edges }` plus viewport — to
the backend as `view_flow` via `POST /flow`.

> The frontend does **no** translation to `models.Node`. It persists its own shape
> verbatim. This is the property that lets the editor round-trip losslessly and lets you
> change what a node compiles to without migrating saved documents.

### 3. Compile

The backend runs the Vue Flow compiler with its hook. This is the only place
product-specific knowledge lives.

### 4. Execute

The resulting `map[string]*models.Node` is what an engine instance fetches — via
`RetrieveFlow` on your backend — and walks.

---

## Writing the hook

A hook switches on `VueFlowNode.Type` and constructs the matching `nodes.*` builder,
pulling whatever fields your form stored in `Data`:

```go
func myNodeBuilder(vfn compiler.VueFlowNode) (*inflowModels.Node, error) {
    data := vfn.Data.(map[string]any)   // map[string]any in practice — it came from JSON

    node := inflowModels.Node{
        ID:    vfn.ID,
        Title: data["title"].(string),
        Key:   data["key"].(string),
        Scope: data["scope"].(string),
    }

    switch vfn.Type {
    case "code":
        node.Type = inflowModels.CodeNodeType
        n := inflowNodes.NewJsNode(data["logic_rule"].(string))
        node.Code = &n.CodeRule

    case "contract":
        node.Type = inflowModels.RuleNodeType
        n := inflowNodes.NewJsRuleLogicNode(
            inflowNodes.WithContractLogicCode(data["logic_rule"].(string)),
        )
        node.Contract = &n.ContractRule

    case "extrinsic":
        node.Type = inflowModels.ExtrinsicNodeType
        // resolve a LOGICAL service name from the canvas to its real subject
        topic, _ := svcHandler.GetSvc(data["serviceTopic"].(string))
        n := inflowNodes.NewExtrinsicSvcNode(topic.MakeReqSubjectWithParams(...))
        node.Extrinsic = &n.ExtrinsicRule

    case "pluginNative":
        node.Type = inflowModels.PluginNodeType
        n, _ := inflowNodes.NewPluginNode(data["subject_prefix"].(string))
        node.Plugin = &n.PluginRule
    }
    return &node, nil
}
```

Note the `extrinsic` case. If your frontend should reference a backend-registered service
by a **logical name** rather than hardcoding a NATS subject, resolve it through
`svcHandler.GetSvc(name)` inside the hook, filling any subject placeholders with
`SvcTopic.MakeReqSubjectWithParams`. The canvas stays free of transport details.

---

## The mapping table, from a real implementation

How the inspector's frontend node types reduce, and which `node.data` fields carry the
configuration:

| Frontend type | `node.data` fields | Primitive | Notes |
| --- | --- | --- | --- |
| `startNode` / `void` | — | **Void** | pure marker / no-op |
| `code` | `lang` (`js`/`opa`), `logic_rule`, `opa_result` | **Code** | `NewJsNode` / `NewOpaNode` |
| `contract` | `lang`, `logic_rule`, `opa_result`, `conditions[]`, `handlers[]` | **Contract** | handlers → tagged `Next` |
| `extrinsic` | `serviceTopic`, `timeout`, `operationData{}` | **Extrinsic** | `operationData` values may carry `{{ }}` runtime variables |
| `pluginNative` | `subject_prefix`, `request`, `idle_min`, `body{}`, `infra_isolated.account` | **Plugin** | maps to `PluginRule` |
| `my_a_ext` | `extension_raw`, `settings{}` (JSON Forms) | **Plugin / Extrinsic** | a packaged extension instance |
| `goto` | goto target/return fields | **GoTo** | |
| `custom` | title/key/scope only | (any) | the bare skeleton before a type is chosen |

`my_a_ext` deserves a callout: it is how a **higher-level, packaged node** appears. The
extension ships a JSON-Schema/UI-Schema form (`extension_raw`), the user fills it in
(`settings`), and the hook turns that into the underlying primitive. On the canvas it
feels like a first-class custom node. Underneath it is still one of the six.

---

## Runtime variables

An `operationData` value on an Extrinsic node may be a template the engine resolves just
before publishing:

| Form | Meaning |
| --- | --- |
| `{{$.ticket.id}}` | an absolute JSONPath into the run's context |
| `{{$this.id}}` | relative to the slice this pass is scoped to |

This is what lets one canvas node be configured once and behave correctly across an
iteration, without the author writing code.

---

## Worked example: a "save finding to DB" node

The full chain, for the extrinsic path:

1. **Backend registers a service:**
   ```go
   svcHandler.ImplHandlerOnSubject("exports_db",
       svcHandler.SvcTopic("svc.add.issue.{TABLE_NAME}"), handler)
   ```
2. **Author.** The user drops an Extrinsic node, opens its drawer, sets `serviceTopic`
   (resolved from the logical name `exports_db`), an `operationData` payload — possibly
   containing `{{$this.id}}` — and a `timeout`.
3. **Save.** The raw graph ships to the backend.
4. **Compile.** The hook reads those fields and builds `nodes.NewExtrinsicSvcNode(subject, …)`
   → `models.ExtrinsicRule`.
5. **Execute.** The engine publishes to the subject; the handler writes the row and replies
   `{"status":"saved …"}`; that reply becomes the node's output, written into context at
   the node's `Key`.

A "send Slack message" node, a "call HTTP API" node and an "approval gate" node are the
same story with a different primitive. **Nothing new in the engine is ever required.**

---

## Debugging a graph

`GetOutboundsEdges(nodeId)` and `GetInboundsEdges(nodeId)` on `*VueFlowCompiler` let you
inspect a node's edges directly when a walk produced — or omitted — a transition you did
not expect.

---

## Next

- **[Compiling a YAML DSL](compiling-a-yaml-dsl.md)** — the same seam, with no canvas and
  no graph library at all.

**Source material:** `inflow-fusion/docs/compilers/vueflow.md`,
`inflow-fusion/docs/nodes/from-frontend.md`, `inflow-fusion/compilers/vueFlow/`.

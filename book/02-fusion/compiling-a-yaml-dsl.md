# Compiling a YAML DSL

> No canvas. No graph library. No edges. The same seam, and the same engine.

The [previous chapter](compiling-a-canvas.md) compiled a visual graph, which is the easy
case — a `{nodes, edges}` export already *is* a graph, so the walk is obvious. This chapter
takes the case people assume cannot work: a **text file**, shaped like a GitHub Actions
workflow, that a user writes by hand or that a wizard mints for them.

If this compiles, the claim generalises. There is nothing graph-shaped about YAML.

> **Status of the code in this chapter.** `compilers/vueFlow` is the compiler that ships
> today. The YAML compiler below is a **worked reference implementation** written against
> the published compiler contract — it is the design you would follow to add
> `compilers/yamlDSL`, not a package you can `go get`. Everything it calls (`nodes.*`,
> `models.*`, the `Compile` signature) is the real SDK surface.

---

## The source format

A deliberately familiar shape. If you have written a GitHub Actions workflow, you can read
this:

```yaml
name: nightly-audit
on:
  schedule: "0 2 * * *"

jobs:
  collect:
    steps:
      - id: fetch_hosts
        uses: postgres
        with:
          query: "select id, hostname from fleet where active = true"

      - id: scan
        uses: scrapli
        with:
          hosts: "{{$.collect.fetch_hosts}}"
          command: "show running-config"

  triage:
    needs: [collect]
    steps:
      - id: classify
        run: |
          input.findings = input.scan.filter(f => f.severity > 3);
          input

      - id: gate
        if: "input.findings.length > 0"
        then: [raise]
        else: [done]

  raise:
    needs: [triage]
    steps:
      - id: create_ticket
        uses: jira
        with:
          project: "SEC"
          summary: "{{$.triage.classify.findings.length}} findings"

  done:
    steps:
      - id: finish
        run: "input"
```

Four constructs carry everything:

| Construct | Meaning |
| --- | --- |
| `jobs.<name>.steps[]` | an ordered sequence of steps; step *n* flows into step *n+1* |
| `needs: [...]` | this job waits for all named jobs to finish |
| `uses: <plugin>` | call a plugin node |
| `run: <js>` | run a computation |
| `if` / `then` / `else` | branch |

That is a complete workflow language, and every construct maps to a primitive.

---

## The mapping

Before writing code, decide the mapping. This table *is* the design:

| YAML construct | Primitive | How |
| --- | --- | --- |
| a job | **Void** entry node | a marker the job's first step follows |
| `needs: [a, b]` | **Void** with `Depends` | the join barrier — waits for both |
| `run:` | **Code** (`js`) | the block becomes the logic rule |
| `uses:` | **Plugin** | the value is the plugin's subject prefix |
| `with:` | plugin/extrinsic body | values may carry `{{$...}}` runtime variables |
| `if` / `then` / `else` | **Contract** | rule returns `then` tags or `else` tags |
| step *n* → step *n+1* | a `Next` entry | implicit sequencing |
| the end of a job | `Next` into each dependent job's entry node | resolved from `needs` |

Two things are notable.

**`needs` is not an edge, it is a `Depends`.** In a visual graph a join is drawn. In YAML
it is declared by name on the *downstream* side. The compiler inverts it: it reads every
job's `needs`, and emits both the forward `Next` entries (from each named job's last step)
and the `Depends` list on the join node. The author writes the dependency once, in the
place it reads naturally, and the compiler produces the graph the engine needs.

**`if` is a Contract, not a property.** There is no "conditional step." The condition
becomes a node, and `then`/`else` become tags on its outgoing transitions. This is the
same reduction [the compiler seam](the-compiler-seam.md) makes for a *Greater than* palette
card — the runtime never learns what `if` means.

---

## The graph type

Step 1 of the [compiler contract](the-compiler-seam.md#the-contract): structs for your
external representation. These are plain YAML bindings, nothing engine-specific.

```go
package yamlDSL

type Workflow struct {
    Name string         `yaml:"name"`
    On   map[string]any `yaml:"on"`
    Jobs map[string]Job `yaml:"jobs"`
}

type Job struct {
    Needs []string `yaml:"needs"`
    Steps []Step   `yaml:"steps"`
}

type Step struct {
    ID   string         `yaml:"id"`
    Uses string         `yaml:"uses"`  // plugin subject prefix
    Run  string         `yaml:"run"`   // JS body
    With map[string]any `yaml:"with"`  // arguments
    If   string         `yaml:"if"`    // condition expression
    Then []string       `yaml:"then"`  // tags when true
    Else []string       `yaml:"else"`  // tags when false
}
```

Compare this to `VueFlow{Nodes, Edges}`. Structurally unrelated — and that is the point.
The compiler contract does not require nodes and edges. It requires that you can *walk*
your format and *resolve transitions*.

---

## The compiler

Step 2 and 3: a constructor that takes a hook, and a `Compile` method.

```go
type YamlCompiler struct {
    eachStep func(Step, string) (*models.Node, error) // the hook: (step, jobName)
}

func NewYamlCompiler(opts ...func(*YamlCompiler)) *YamlCompiler {
    c := &YamlCompiler{}
    for _, o := range opts {
        o(c)
    }
    return c
}

func WithEachStepFunc(f func(Step, string) (*models.Node, error)) func(*YamlCompiler) {
    return func(c *YamlCompiler) { c.eachStep = f }
}
```

`Compile` does the structural work — everything that is about *this format*, not about
*this product*:

```go
func (c *YamlCompiler) Compile(startJob string, wf Workflow) (map[string]*models.Node, map[string]error) {
    out  := map[string]*models.Node{}
    errs := map[string]error{}

    // ── Pass 1: every job gets an entry node. `needs` becomes Depends. ───────────
    for jobName, job := range wf.Jobs {
        entry := &models.Node{
            ID:    entryID(jobName),
            Type:  models.VoidNodeType,
            Title: jobName,
        }
        for _, need := range job.Needs {
            entry.Depends = append(entry.Depends, lastStepID(wf, need))
        }
        out[entry.ID] = entry
    }

    // ── Pass 2: steps become nodes, chained in declaration order. ───────────────
    for jobName, job := range wf.Jobs {
        prev := out[entryID(jobName)]

        for _, step := range job.Steps {
            node, err := c.eachStep(step, jobName)   // ← THE HOOK
            if err != nil {
                errs[step.ID] = err
                break
            }
            node.ID = stepID(jobName, step.ID)
            out[node.ID] = node

            // implicit sequencing: previous step flows into this one
            prev.Next = append(prev.Next, models.Next{NodeId: node.ID})

            // a conditional step's tags become its outgoing transitions
            if step.If != "" {
                node.Next = branchTargets(wf, jobName, step)
            }
            prev = node
        }
    }

    // ── Pass 3: a job's last step flows into every job that needs it. ──────────
    for jobName, job := range wf.Jobs {
        for _, need := range job.Needs {
            last := out[lastStepID(wf, need)]
            last.Next = append(last.Next, models.Next{NodeId: entryID(jobName)})
        }
        _ = job
    }

    return out, errs
}
```

Three passes, and none of them knows what a `jira` step or a `postgres` step *is*. That
knowledge lives in exactly one place.

---

## The hook

Step 4: the seam. This is the only function that knows your vocabulary.

```go
func myStepBuilder(step Step, jobName string) (*models.Node, error) {
    node := &models.Node{
        Title: step.ID,
        Key:   step.ID,                       // output lands at <scope>.<step id>
        Scope: "$." + jobName,                // each job owns a slice of context
    }

    switch {
    // ── a conditional step → Contract ──────────────────────────────────────────
    case step.If != "":
        node.Type = models.RuleNodeType
        n := nodes.NewJsRuleLogicNode(
            nodes.WithContractLogicCode(
                fmt.Sprintf(`(%s) ? data.then : data.else`, step.If),
            ),
            nodes.WithContractConditions(map[string]any{
                "then": step.Then,
                "else": step.Else,
            }),
        )
        node.Contract = &n.ContractRule

    // ── a computation step → Code ──────────────────────────────────────────────
    case step.Run != "":
        node.Type = models.CodeNodeType
        n := nodes.NewJsNode(step.Run)
        node.Code = &n.CodeRule

    // ── a plugin step → Plugin ─────────────────────────────────────────────────
    case step.Uses != "":
        node.Type = models.PluginNodeType
        n, err := nodes.NewPluginNode(step.Uses, nodes.WithIdleWaitMinutes(30))
        if err != nil {
            return nil, err
        }
        n.PluginRule.Body = step.With       // `with:` becomes the request body
        node.Plugin = &n.PluginRule

    default:
        return nil, fmt.Errorf("step %q: needs one of run/uses/if", step.ID)
    }
    return node, nil
}
```

Twenty-five lines. That is the entire distance between "a YAML file" and "something this
runtime executes."

Notice what the hook did with `if`: it built a JS rule whose *result is a tag list*, taking
the tag lists themselves from `data`. The author wrote `then: [raise]`, and the rule returns
`["raise"]` when the condition holds. The engine then fires only the transition tagged
`raise`. No new primitive, no engine change — exactly the mechanism from
[Tag routing](tag-routing.md).

---

## Wiring it up

```go
var wf yamlDSL.Workflow
if err := yaml.Unmarshal(fileBytes, &wf); err != nil {
    return err
}

cmpr := yamlDSL.NewYamlCompiler(yamlDSL.WithEachStepFunc(myStepBuilder))
nodeMap, errs := cmpr.Compile("collect", wf)
if len(errs) > 0 {
    return fmt.Errorf("compile: %v", errs)
}

// store it; serve it from RetrieveFlow when the engine asks
flow := models.Flow{UUID: flowID, Nodes: flatten(nodeMap)}
flow.ValidateNext()
myDB.SaveCompiledFlow(flowID, flow)
```

From here it is identical to the canvas path: the engine asks `RetrieveFlow`, gets this
node map, and walks it. See [the backend contract](the-backend-contract.md).

---

## What the author gets for free

The YAML above is roughly forty lines. Compiled, it is a durable, distributed process
with properties its author never asked for and did not implement:

| Property | Where it came from |
| --- | --- |
| **Parallelism** — `collect` and `done` have no dependency, so they run concurrently | the engine's traversal |
| **A real join** — `triage` waits for all of `collect`'s steps | `Depends` on the entry Void node |
| **Durability** — the run survives a crash and resumes | the context document + `Resume` |
| **Long waits** — a step can park the flow for days | `_cmd: stop` + a later resume |
| **Observability** — every node start, finish and edge selection is an event | the process event stream |
| **Isolation** — the `jira` step cannot see the `postgres` step's traffic | scoped plugin credentials |
| **An integration catalog** — `uses: jira`, `uses: postgres`, `uses: qdrant` already exist | the plugin ecosystem |

That last row is the one to sit with. The author of this YAML product wrote a struct, a
three-pass walker and a twenty-five-line hook. They did not write a Jira integration, a
Postgres integration or a Qdrant integration — those are existing plugins that speak
`inflowv1`, and **they work because the protocol is the contract, not the product.**

---

## Generalising further

Nothing in this chapter depended on YAML, or on jobs and steps. The recipe is:

1. **Define the graph type** — whatever your users author.
2. **Decide the mapping** — one row per construct, ending at one of the six primitives.
3. **Write `Compile`** — walk your structure, resolve transitions into `Next`, and hoist
   any declared dependencies into `Depends`.
4. **Write the hook** — one `switch`, one `nodes.*` builder per construct.

Formats that fall out of the same recipe:

| Source format | The walk | Notes |
| --- | --- | --- |
| A JSON data model minted by a form wizard | iterate the declared steps | often the simplest case — no branching to resolve |
| A BPMN / XML export | follow `sequenceFlow` elements | gateways → Contract; tasks → Plugin/Extrinsic |
| A state machine (`states`, `transitions`) | transitions are already tags | almost a direct mapping |
| A linear script with labels and `goto` | labels are node ids | jumps → `Next`, or the **GoTo** primitive |
| An existing n8n / Zapier export | its own nodes/connections shape | an import path into your product |
| A prompt-authored plan from an LLM | validate, then walk | the model proposes; the compiler validates; the graph decides |

The last row is a real pattern rather than a flourish: because compilation is a separate,
inspectable step, a generated workflow can be **rejected at compile time** if it references
a node type you do not offer. The model cannot invent a capability.

---

## The honest limits

Three things the seam does not do for you, stated plainly:

- **Validation is yours.** The compiler reports hook errors per node, but *"is this a
  sensible workflow?"* — unreachable steps, cycles you did not intend, a `needs` pointing
  at a job that does not exist — is your compiler's job. The engine will faithfully execute
  a graph that makes no sense.
- **Your format's expressiveness is your problem.** If your DSL cannot express a join, the
  compiler cannot invent one. The primitives are sufficient; your surface syntax might not
  be.
- **Round-tripping is a design choice.** Persisting the source verbatim (as the canvas path
  does with `view_flow`) keeps the editor lossless. If you instead persist only the
  compiled node map, you have thrown away the author's document. Persist both, or persist
  the source.

---

## Next

- **[Build your own workflow product](build-a-workflow-product.md)** — the assembly
  instructions for everything around the compiler.

**Source material:** `inflow-fusion/docs/compilers/README.md` (the contract this chapter
implements), `inflow-fusion/docs/nodes.md`, and the blog post
*Build Your Own Workflow Product — the Runtime Is the Part You Don't Have to Write*.

# The node palette

> **Status: outlined.**

Fifteen canvas nodes, grouped by intent. **Every one is annotated with the runtime
primitive it lowers to — and that annotation is visible in the product, not hidden in the
compiler.** That transparency is itself part of the argument.

## Flow

| Node | Lowers to | What it does |
| --- | --- | --- |
| **Start** | `Void` | The entry marker. Exactly one per flow. |
| **Wait for All** | `Void` | A synchronisation barrier — `Promise.all` for branches. Holds until all inbound branches finish, merges their results, continues once. |
| **Continue After** | `Extrinsic · svc.continue.at` | Park the run and resume later — `now + delay`, or an absolute time. |
| **Goto** | `GoTo` | Jump into another (or the same) flow like a subroutine, and come back. |

## AI & Logic

| Node | Lowers to | What it does |
| --- | --- | --- |
| **LLM** | `Plugin` | One turn of a model conversation held on the node's scope, streamed to the canvas. **Bound functions become output ports.** |
| **Jev** | `Plugin` | A calibrated *decider*, not a reasoner. Evaluates a state template against typed questions (choice / score / noul) and routes each question's top answer to its own port (`<question>.<option>`). One round-trip, no free text, no refusal. |
| **MCP** | `Plugin` | An MCP *client*. "Tool only" calls one tool with typed arguments, no model. "With LLM" drives a model bound to the server's tools and runs the agentic loop internally. |
| **Rule** | `Contract` | Evaluate JS or OPA/Rego over the scoped context; each handler is a tagged output port. The branching, policy and guardrail node. |
| **JS** | `Code · js` | A JavaScript step over the scoped context. |
| **OPA** | `Code · opa` | A Rego policy over the scope (as `input`) plus condition key/values (as `data`). |

## Stores

| Node | Lowers to | What it does |
| --- | --- | --- |
| **Doc Store** | `Extrinsic · svc.store.doc.*` | Read (validated read-only query) or write documents in a referenced Document store. |
| **Vector Store** | `Extrinsic · svc.store.vec.*` | Index or search a referenced Vector store — embedding, top-k, optional partition namespace. |
| **Cast / Mapping** | `Plugin` | Build a value by mapping each target key of a store's schema to a static value or a JSONPath resolved at run time. |

## Integrations

| Node | Lowers to | What it does |
| --- | --- | --- |
| **HTTP** | `Plugin` | An HTTP/REST request. Connection config comes from a settings profile; `{{$.a.b}}` tokens in every string field resolve against the live flow context at run time. |

## Human

| Node | Lowers to | What it does |
| --- | --- | --- |
| **Human in the Loop** | `Extrinsic · svc.hitl.add` | Pause the flow for a person. Poses questions, records a Human Task, resumes when answers arrive. |

## Sections planned

**1. Reading the table as evidence.** Fifteen product nodes, and **all six primitives are
exercised** — Void, Code, Contract, Extrinsic, Plugin, GoTo. No runtime changes were needed
for any of them.

**2. The three universal fields.** `title`, `key`, `scope` — mirrored onto the compiled
primitive, exactly as [Part II](../02-fusion/node-primitives.md#the-universal-fields)
describes.

**3. "Bound functions become output ports."** The LLM node in detail — the clearest real
instance of [tag routing](../02-fusion/tag-routing.md) driven by a model. The model picks
among ports you drew; it cannot invent a third edge.

**4. The decider, not the reasoner.** The Jev node as the second worked instance of the same
argument. Where the LLM binds a model's callable *functions* to ports, Jev binds a template's
declared *answers* to ports — and the answers are fixed at author time, with calibrated
probabilities attached at run time. It cannot decline to answer; a question with no confident
match returns `other`, which is a port you drew. Same `CmdNextFilter`, no free text, one
round-trip. Two nodes, one mechanism — see [the AI harness](ai-harness.md).

**5. Around the canvas.** The entities a real system needs beside the graph: Contexts,
Memory (vector + document stores), Prompts, Node settings profiles, Processes, Human tasks,
Extensions.

**6. Adding a node kind.** The palette is data plus a hook — a catalog entry on the frontend
and a case in the compiler, not a runtime change. If the behaviour is not expressible with
compiled primitives, it becomes a plugin node instead.

## Source material

`FloMorphic/getting-started/docs/nodes.md`, `docs/concepts.md`.

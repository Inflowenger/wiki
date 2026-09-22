# The computational model

> **Status: outlined.**

## The four parts

The model has four concepts, and the platform's entire vocabulary reduces to them.

| Concept | One-liner | Computer analogy |
| --- | --- | --- |
| **Context** | The memory. Everything enters the system as context. | RAM |
| **Workflows** | The logic. Logic as a visible, traceable graph. | Program |
| **Fractals** | The processors. Runtime instances that execute graphs. | Process / OS instance |
| **Adapters** | Connect to the world. | Drivers / I/O |

## The OS analogy, drawn explicitly

| Traditional computer | Inflowenger |
| --- | --- |
| A process instance | **Fractal runtime** (and a node can be an embedded flow) |
| The operating system | **Inflowenger runtime** |
| Extensions, drivers and interrupts | **Plugins**, Fractal instances |

## Sections planned

**1. Context as a JSON-path-addressable tree.** `$.OPA`, `$["doc appendix"]`, `$this`.
How a node's `Scope` narrows what it sees and `Key` decides where its output lands. Why
this beats passing values along edges: a join does not have to merge tuples, and a node
added later can read something written long before it.

**2. Why "everything enters as context."** Triggers, webhooks, form submissions, a
scheduled tick — all of them become a context document, and from there the system has one
data model rather than N entry-point shapes.

**3. Workflows as the program, and what that buys.** Visible logic, traceable execution,
safer change. Cross-reference [The thesis](../00-preface/the-thesis.md).

**4. Fractals as processors — and the fractal property.** Why the name: a node can itself
be an embedded flow, so the same structure holds at every scale. A flow of flows is still
a flow. This is what GoTo composition means conceptually.

**5. Adapters as the edge of the system.** Plugins are the concrete form. The point is
that the runtime has *no* built-in integrations — reaching the world is always an adapter,
which is why no integration is privileged.

**6. What this model deliberately does not have.** No global variables, no ambient service
locator, no implicit ordering beyond the graph. Constraints that make runs reproducible.

## Source material

`inflow-plugin-sdk/docs/architecture.md`, `inflow-plugin-sdk/docs/inflow-ecosystem.md`,
the marketing site's `ComputationalModelSection.vue` and `WorkflowGraphSection.vue`.

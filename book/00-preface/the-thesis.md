# The thesis: software as a graph over living context

> **Status: outlined.** Thesis and section plan are committed; prose is not yet written.

## The claim

Business logic is not code distributed across services. It is a **workflow graph over
living context**, changeable by an operator without a redeploy.

The Inflowenger material calls this **Software V3**, and **context-oriented software
development**. This chapter states it properly and honestly — including what it costs.

## Sections planned

**1. What breaks in V2.** Logic scattered across services, ORMs and handlers. Nobody can
point at "the approval rule." Changing it is a release. Explaining it to an auditor is an
archaeology project.

**2. The three consequences of making the graph the program.**
- *Business logic becomes visible* — it is a diagram because it **is** the diagram, not a
  diagram someone drew about the code and forgot to update.
- *Execution becomes traceable* — every run records the path taken and the reason
  ([Observing a run](../02-fusion/observing-a-run.md)).
- *Change becomes safer* — the blast radius of an edit is a node, and the possibilities were
  drawn in advance.

**3. Context as the memory.** Everything enters the system as context. A run is an
iteration over one durable context object that survives a wait, an approval or a crash.
This is what "living context" means concretely — contrast with a request/response call
stack that dies with the process.

**4. Why this matters more now.** The AI layer made it urgent rather than merely tidy. A
model is a *bounded participant* in a process, not the process. The graph is where the
bound lives. → *The model proposes, the graph decides.*

**5. Flow engineering, and what it is not.** Not "flow" in the linear-automation sense
(mailbox → step → step → send). Flow engineering in the sense the technique was reaching
for: breaking work into states and transitions, with control on a layer you can see.
Underneath, a **durable cyclic graph** — nodes are capabilities, edges are the decisions
the model is permitted to make, a loop is an edge pointing backward plus a condition.

**6. The honest costs.** A graph is not free. Debugging a distributed graph differs from
stepping through a function. Versioning a flow that has live runs against it is a real
problem. Some logic is genuinely better as code, and Code and Extrinsic nodes exist
precisely so you are not forced to draw it.

## Source material

`inflow-vue/inflow-nuxt/content/blog/` — *Context-Oriented Software Development*,
*Flow Engineering Is an Architecture, Not a Prompting Trick*, *Glass-Box Agents Are Not
Optional*, *The Model Proposes, the Graph Decides*, *Yes, It's a Graph. That's Why We Call
It Flow Engineering.*

## Next

- [Glossary](glossary.md) · [Part I — What Inflowenger is](../01-platform/index.md)

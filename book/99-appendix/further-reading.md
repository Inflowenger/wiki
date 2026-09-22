# Further reading

> **Status: outlined.** The blog corpus is the narrative companion to this book; several
> chapters draw directly on it.

## The argument, in essays

| Post | Relevant to |
| --- | --- |
| *Context-Oriented Software Development* | [The thesis](../00-preface/the-thesis.md) |
| *Flow Engineering Is an Architecture, Not a Prompting Trick* | [The thesis](../00-preface/the-thesis.md) |
| *Yes, It's a Graph. That's Why We Call It Flow Engineering.* | [The thesis](../00-preface/the-thesis.md) |
| *Build Your Own Workflow Product — the Runtime Is the Part You Don't Have to Write* | **[Part II](../02-fusion/build-a-workflow-product.md)** |
| *The Model Proposes, the Graph Decides* | [Tag routing](../02-fusion/tag-routing.md) |
| *Glass-Box Agents Are Not Optional* | [Observing a run](../02-fusion/observing-a-run.md) |
| *Your Evaluator Is a Node. Your Loop Is an Edge.* | [Coverage](../02-fusion/coverage.md) |
| *Cardinality Is the Loop You Didn't Write* | [Coverage — awkward cases](../02-fusion/coverage.md#the-awkward-cases-stated-plainly) |
| *Pausing Is Easy. Resuming Is the Hard Part.* | [The backend contract](../02-fusion/the-backend-contract.md#resuming-a-run) |
| *Two Ways to Extend an Agent Platform* | [Part III](../03-plugins/), [AI harness](../04-flomorphic/ai-harness.md) |
| *Integration Count Is a Vanity Metric Now* | [The plugin catalog](../03-plugins/catalog.md) |
| *You Extend FloMorphic the Way You Install a Browser Extension* | [Builtin nodes are plugins](../04-flomorphic/builtin-nodes.md) |
| *An Agent Can't Act on What It Can't Name* | [AI harness](../04-flomorphic/ai-harness.md) |
| *Reason Once, Reuse Many Times* | [AI harness](../04-flomorphic/ai-harness.md) |
| *Your Company Doesn't Need Its Own LLM. It Needs a Brain.* | [Part IV](../04-flomorphic/) |
| *Stop Chat. Start Work.* | [Part IV](../04-flomorphic/) |
| *Build a Claims Adjudicator You Can Actually Audit* | [Part IV](../04-flomorphic/) |
| *FloMorphic Is Now an MCP Server* | [Driving it over MCP](../04-flomorphic/mcp.md) |
| *Venapce: A Nervous System for Security Governance* | [Part V](../05-venapce/) |
| *Your Fleet Is a Live Database. Ask It Something.* | [Collectors](../05-venapce/collectors.md) |
| *Your Semantic Layer Is a Flow* | [Part V](../05-venapce/) |

## Worked flows

`FloMorphicProject/flow-cookbook/` — runnable examples:
`naive-rag`, `eval-loop`, `linux-fleet-http-audit`, `qdrant-migrate`.

## External

- [NATS](https://nats.io) — the message substrate
- [JSON Forms](https://jsonforms.io) — the form renderer `x-inflow-ui` extends
- [Vue Flow](https://vueflow.dev) / [React Flow](https://reactflow.dev) — the canvas
  libraries the shipped compiler targets
- [Open Policy Agent / Rego](https://www.openpolicyagent.org) — the policy language behind
  OPA Code and Contract nodes

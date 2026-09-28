# The AI harness

> **Status: outlined.**

Why RAG, agent loops, tool use, memory and guardrails need **no runtime support** — they are
compositions of nodes over one shared context.

## Sections planned

**1. The reframe.** Every capability the field has developed becomes an ordinary node on one
graph, over one shared context. Nothing is a framework feature; everything is a node and an
edge.

**2. Declared answers are the ports.** The platform's answer to "how do you let a model
choose a path without losing control of it": bind the model's callable **functions** to the
node's outbound ports. The model proposes; the graph decides. It picks among ports drawn in
advance and cannot invent a third edge.

  FloMorphic ships **two** instances of this, and the pairing is the point:

  | | **LLM** | **Jev** |
  | --- | --- | --- |
  | Decides with | generated text | a calibrated template over typed questions |
  | What is bound to ports | the model's callable **functions** | the template's declared **answers** (`<question>.<option>`) |
  | Can it refuse to answer | yes — hallucinate, waffle, call nothing | **no** — no confident match answers `other`, which is a port |
  | Cost | one model turn, streamed | one 70–500 ms round-trip, no free text |
  | Both | `CmdNextFilter` — [tag routing](../02-fusion/tag-routing.md) over ports the author drew | same |

  Same mechanism, same guarantee, two very different nodes. The LLM node is the *reasoner* —
  open-ended, slow, generative. The Jev node is the *decider* — closed-set, fast, calibrated.
  Most agent work that people bolt onto a model is really a decider that got implemented as a
  reasoner, and paying a model turn to choose between five known options is the symptom. Move
  it to a decider and it becomes one edge on a graph you can inspect.

**3. Context, working memory and long-term memory.** The run's context document is working
memory. Document and vector stores are long-term memory, reached through Extrinsic nodes.
The distinction is architectural, not a library's opinion.

**4. RAG is two nodes.** A vector-store search and a model call. Shown concretely, and
deliberately anticlimactic.

**5. Agent loops.** A backward edge plus a condition. The evaluator is a node; the loop is an
edge. Why this is more inspectable than a framework's hidden `while` loop.

**6. Tool use vs. MCP.** Two ways to extend an agent platform, and why they are not the same
axis. When a plugin is right and when an MCP server is. (There is a third axis orthogonal to
both — *reasoner* vs. *decider*, §2 above — and conflating it with tool use is what produces
agents that spend a model turn to pick from a dropdown.)

**7. Guardrails and policy.** A Rego Contract, evaluated outside the model, before or after
it. Out-of-model reasoning as a first-class position rather than a prompt instruction. Policy
lives in the graph in a second place too: a decider node's `min_confidence` is not a prompt
instruction ("be careful") but a real threshold the node enforces — below it, the node routes
the reserved `_exception` port and reports failure with its scores attached, rather than
quietly returning its best bad guess. The threshold is a field on the node; the fallback is an
edge you drew. See
[the exception port](../02-fusion/tag-routing.md#the-exception-port--fail-and-still-route).

**8. Human in the loop.** The approval gate as an Extrinsic that parks the run. Pausing is
easy; **resuming is the hard part** — and the resume machinery is in
[the backend contract](../02-fusion/the-backend-contract.md#resuming-a-run).

**9. Observability as a precondition, not a feature.** Glass-box agents are not optional.
Cross-reference [Observing a run](../02-fusion/observing-a-run.md).

**10. Multi-agent orchestration.** Several model nodes on one graph over one context — which
is what "multi-agent" reduces to once the orchestration is visible.

## Source material

`FloMorphic/getting-started/docs/ai-harness.md`, `FloMorphicProject/flow-cookbook/`
(`naive-rag`, `eval-loop`), and the blog posts *Your Evaluator Is a Node*, *Glass-Box Agents
Are Not Optional*, *Two Ways to Extend an Agent Platform*, *Pausing Is Easy. Resuming Is the
Hard Part.*

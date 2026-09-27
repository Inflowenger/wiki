# Part IV — FloMorphic

> **The runtime's first product — and the proof that nothing is reserved.**
>
> **Status: outlined.** Chapter theses and section plans are committed; prose is pending.

Everything in Parts I–III is a claim about what the runtime *could* support. FloMorphic is
the claim surviving contact with a real, non-trivial product.

**The load-bearing fact:** FloMorphic has **nothing special reserved for it in the
runtime.** Its fifteen canvas nodes all lower to the same six primitives. Its AI
capabilities — the LLM node, the MCP node, the Jev decider — are ordinary plugins with no
privileged access, which is exactly why yours can be too.

> Not a product. An opportunity. FloMorphic is not sold as the finished answer to your
> problem — it is the means to build your own.

## The three layers

```
┌──────────────────────────────────────────────────────────────────────────┐
│  LAYER 3 — PRODUCT                                                       │
│  FloMorphic · flomorphic-wapp (canvas) + flomorphic-api (Go backend)     │
│  Intent-level nodes: LLM · MCP · Rule · Stores · Human-in-the-loop       │
└───────────────────────────────────┬──────────────────────────────────────┘
                                    │  compiles down to
┌───────────────────────────────────▼──────────────────────────────────────┐
│  LAYER 2 — SDK & REFERENCE                                               │
│  inflow-fusion (Go SDK) · inflow-inspector (+ inspector-api)             │
│  Six primitives · the compiler seam · the backend contract               │
└───────────────────────────────────┬──────────────────────────────────────┘
                                    │  executed by
┌───────────────────────────────────▼──────────────────────────────────────┐
│  LAYER 1 — RUNTIME (headless)                                            │
│  Infra (coordination, NATS, credentials) · Fractal (execution engine)    │
└──────────────────────────────────────────────────────────────────────────┘
```

## Chapters

| Chapter | What it establishes |
| --- | --- |
| [What FloMorphic is](what-it-is.md) | The AI harness thesis, and what it attaches to |
| [The node palette](the-palette.md) | Fifteen nodes, and what each one lowers to |
| [Builtin nodes are plugins](builtin-nodes.md) | The clearest evidence nothing is reserved: same SDK, same protocol, only the UI path differs |
| [The AI harness](ai-harness.md) | Why RAG, agent loops, tool use and guardrails need no runtime support |
| [How it was built](how-it-was-built.md) | The walkthrough: FloMorphic as a worked Part II |
| [Driving it over MCP](mcp.md) | The API is itself an MCP server |

## Source material

`FloMorphic/getting-started` README and `docs/` · `flomorphic-api/README.md` ·
`flomorphic-wapp/README.md` · `builtin-plugins/README.md` · the blog corpus.

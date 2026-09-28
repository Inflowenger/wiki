# Part VII — Architecture in Practice

> **What the architecture buys you, without reading the closed source.**
>
> **Status: outlined.**

Infra and Fractal are not open source. That is a fact worth confronting directly rather
than working around, because the question it raises is legitimate: *if I cannot read the
engine, what exactly am I relying on?*

The answer this part gives is that **scalability and customization are properties of the
architecture, not of the implementations** — and the architecture is fully visible. Three
elements produce them, and you can reason about all three from their contracts:

| Element | What it contributes |
| --- | --- |
| **Infra** | identity, spaces, scoped credentials, the engine registry |
| **Fractal** | stateless-with-respect-to-your-domain execution, horizontally replicable |
| **Plugins** | isolated, independently deployed, protocol-bound extension |

Nothing in this part requires reading Infra's or Fractal's source. Everything in it is
derivable from the contracts documented in Parts I–III.

## Chapters

| Chapter | What it establishes |
| --- | --- |
| [Spaces and isolation](spaces-and-isolation.md) | Multi-tenancy as an architecture, not a `WHERE` clause |
| [Scaling out](scaling.md) | Laptop → cluster, and where each bottleneck actually is |
| [Customization](customization.md) | Every seam that is open, and what each one costs |
| [Attaching a system you already run](attaching-legacy.md) | The three doors into an existing estate |

## Source material

`inflow-fusion/docs/architecture.md` and `docs/infra.md` ·
`inflow-plugin-sdk/docs/inflow-ecosystem.md` ·
`FloMorphic/getting-started/docs/architecture.md` · `Inflowenger/getting-started` README.

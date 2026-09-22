<div align="center">

# The Inflowenger Book

### Where context becomes computation

**The knowledge base for the Inflowenger runtime, the `inflow-fusion` SDK,
the `inflowv1` plugin protocol, and the products built on them — FloMorphic and Venapce.**

`Inflowenger` · `inflow-fusion` · `inflowv1` · `FloMorphic` · `Venapce`

</div>

---

## What this is

Inflowenger is a **runtime for context processing** — a substrate for building software
whose logic is a workflow graph rather than code distributed across services.

That sentence is easy to write and hard to believe. This book exists to make it
checkable. It answers, in order:

1. **What Inflowenger actually is** — the computational model, the six primitives, the
   two headless services (Infra and Fractal) that run them.
2. **How anything becomes a flow** — the part most people assume you cannot generalise.
   A canvas graph, a YAML file, a JSON DSL, a GitHub-Actions-style workflow file: all of
   them lower to the same primitive node map through one seam, the **compiler hook**.
   This is the `inflow-fusion` story, and it is the centre of the book.
3. **How the runtime is extended** — the `inflowv1` protocol and the plugin SDKs, the one
   node type that never compiles away.
4. **That the claim survives contact with a real product** — **FloMorphic**, an AI harness
   built end to end on the runtime with nothing reserved for it.
5. **That it survives a second time, one layer up** — **Venapce**, a security-governance
   product whose entire business logic lives in FloMorphic workflows.
6. **How it scales and bends** — what Infra, Fractal and the plugin isolation model buy
   you, architecturally, without reading their source.

> **On the closed parts.** Infra and Fractal are not open source. This book does not
> document their internals and does not need to: it documents their *contracts* — the
> REST endpoints, the NATS subjects, the guarantees — which is exactly what you build
> against. Everything you must write to run on this platform is open and covered here.

---

## Who this is for

| If you are… | Start at |
| --- | --- |
| Evaluating whether the claims hold up | [Part 0 — The Thesis](book/00-preface/the-thesis.md) |
| Building a **workflow product** on the runtime | [Part II — The Fusion Layer](book/02-fusion/) |
| Writing a **plugin node** | [Part III — The Plugin Layer](book/03-plugins/) |
| Trying to understand what FloMorphic *is* | [Part IV — FloMorphic](book/04-flomorphic/) |
| Responsible for running this in production | [Part VI — Architecture in Practice](book/06-architecture/) |
| Lost in the repository sprawl | [Appendix — Ecosystem map](book/99-appendix/ecosystem-map.md) |

---

## Table of contents

The full, ordered table of contents is **[SUMMARY.md](SUMMARY.md)** — it is also the
book's spine for any static-site or PDF build.

| Part | Subject | State |
| --- | --- | --- |
| **0** | [Preface](book/00-preface/) — the thesis, how to read this, glossary | outlined |
| **I** | [The Platform](book/01-platform/) — what Inflowenger is: model, primitives, Infra, Fractal | outlined |
| **II** | [The Fusion Layer](book/02-fusion/) — **any source format → a running flow** | **written** |
| **III** | [The Plugin Layer](book/03-plugins/) — `inflowv1`, the SDKs, forms, jobs | outlined |
| **IV** | [FloMorphic](book/04-flomorphic/) — the runtime's first product | outlined |
| **V** | [Venapce](book/05-venapce/) — a product built on the product | outlined |
| **VI** | [Architecture in Practice](book/06-architecture/) — scale, isolation, customization | outlined |
| — | [Appendix](book/99-appendix/) — ecosystem map, repositories, further reading | outlined |

**Outlined** chapters carry their thesis, their section plan, and pointers to the source
material they will be written from. They are a commitment to content, not content.

---

## The one-paragraph version

A headless control plane (**Infra**) owns identity, credentials and an embedded NATS bus,
and keeps a registry of live execution engines. A headless engine (**Fractal**) takes a
compiled node map and walks it — durably, across crashes, through loops, pausing for days.
Your backend imports **`inflow-fusion`**, a Go SDK, and thereby owns the data: it answers
three questions over NATS (*what is this flow?*, *what is this run's context?*, *take this
updated context*) and exposes domain logic as callable steps. Between your authoring
surface and the engine sits a **compiler** whose per-node **hook** is the only place your
product's vocabulary lives. Six primitives are all the engine can execute; the sixth,
**Plugin**, is a live external process speaking **`inflowv1`** over NATS, and is the open
end through which everything else arrives.

```
   your authoring surface            ← yours (canvas, YAML, DSL, API)
        │  save verbatim
        ▼
   your backend + inflow-fusion      ← yours (data, domain logic, the compiler hook)
        │  answers 3 questions over NATS
        ▼
   Fractal (engine)  ←── registry ──  Infra (control plane, NATS, credentials)
        │
        └── Plugin node ──NATS──▶ your plugin process (inflowv1)
```

---

## Contributing

This book is assembled from the source repositories listed in the
[appendix](book/99-appendix/repositories.md). Where a chapter and a source repo's own
docs disagree, **the source repo wins** — open an issue here so the chapter is corrected.

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

Apache License 2.0 — see [LICENSE](LICENSE). "Inflowenger", "FloMorphic" and "Venapce"
are trademarks; Apache-2.0 does not grant permission to use them beyond describing this
work's origin.

---

<div align="center">
<sub>Part of the Inflowenger platform · Where context becomes computation.</sub>
</div>

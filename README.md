<div align="center">

# The Inflowenger Book

### Where context becomes computation

**The knowledge base for the Inflowenger runtime, the `inflow-fusion` SDK,
the `inflowv1` plugin protocol, the `inflow-js` frontend packages, and the products
built on them — FloMorphic and Venapce.**

`Inflowenger` · `inflow-fusion` · `inflowv1` · `inflow-js` · `FloMorphic` · `Venapce`

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
4. **What the browser gets for free** — `inflow-js`: a live run rendered on your canvas,
   and a working configuration form for an integration you have never seen. Two npm
   packages, and the chapter on giving *your* users the ability to define their own
   processes.
5. **That the claim survives contact with a real product** — **FloMorphic**, an AI harness
   built end to end on the runtime with nothing reserved for it.
6. **That it survives a second time, one layer up** — **Venapce**, a security-governance
   product whose entire business logic lives in FloMorphic workflows, and which ships its
   own features as **installable flow packages**.
7. **How it scales and bends** — what Infra, Fractal and the plugin isolation model buy
   you, architecturally, without reading their source.
8. **What it is like to actually use** — one continuous build, install to a system that
   answers a customer request against its own contracts. The part that is a *session*
   rather than an explanation.

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
| Letting **your users define their own processes** | [Part IV — Building a process product](book/04-frontend/build-a-process-product.md) |
| Building the **browser half** — canvas, forms, live runs | [Part IV — The Frontend Layer](book/04-frontend/) |
| Trying to understand what FloMorphic *is* | [Part V — FloMorphic](book/05-flomorphic/) |
| Responsible for running this in production | [Part VII — Architecture in Practice](book/07-architecture/) |
| Starting from zero and wanting to *build something* | [Part VIII — A guided build](book/08-guided-build/) |
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
| **IV** | [The Frontend Layer](book/04-frontend/) — `inflow-js`: **live runs and dynamic forms in the browser** | **written** |
| **V** | [FloMorphic](book/05-flomorphic/) — the runtime's first product | outlined |
| **VI** | [Venapce](book/06-venapce/) — a product built on the product | **written** |
| **VII** | [Architecture in Practice](book/07-architecture/) — scale, isolation, customization | outlined |
| **VIII** | [A guided build](book/08-guided-build/) — **install → the brain of an organization**, end to end | **written** |
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
end through which everything else arrives. In the browser, two npm packages
(**`inflow-js`**) turn the engine's event stream into movement on your canvas and a
plugin's declared schema into a working form.

```
   your frontend  + inflow-js        ← yours (the UI) · flow-trace + plugin-form-builder
        │  save verbatim      ▲  live run + plugin forms
        ▼                     │
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

# How FloMorphic was built on the runtime

> **Status: outlined.**
>
> This chapter is [Part II](../02-fusion/build-a-workflow-product.md) worked as a case
> study. Same eight steps, a real product, and the decisions actually taken.

## Sections planned

**1. The authoring surface.** Vue 3 + Vite + TypeScript + Vue Flow + Tailwind v4 + Pinia.
The canvas saves the **raw Vue Flow graph verbatim** as `view_flow` — no translation on
save. It runs standalone (browser-local) or connected, which is why authoring works with no
platform behind it.

**2. The backend.** `flomorphic-api`: Go 1.27 + Fiber v3, SQLite + `sqlite-vec` via sqlc.
It implements `IInflowService` over its own storage. Note how modest the storage choice is —
the engine does not care, so the product picked what suited it.

**3. The compiler hook.** `inflow/compiler.go` + `inflow/node_builders.go`, walking a Vue
Flow graph via `inflow-fusion`'s `compilers/vueFlow`. Fifteen palette nodes, one case each
(start and Wait-for-All share a `Void` case; the two stores share a store case). This is the
seam, in production.

**4. The extrinsic handlers.** `svc.store.doc.*`, `svc.store.vec.*`, `svc.hitl.add`,
`svc.continue.at` — the four service families backing the store, human-in-the-loop and
continue-after nodes. Each is a handler in the API process; each is a palette card.

**5. The plugin nodes.** LLM, MCP, Cast, HTTP, Jev — ordinary plugins.
→ [Builtin nodes are plugins](builtin-nodes.md)

**6. The pause/resume design.** How *Continue After* and *Human in the Loop* park a run and
how the API schedules the resume. This backend's `inflow/` package is walked line by line in
[The wire](../02-fusion/the-wire.md) — the park shapes, the traversal snapshot, the error
ledger and the lost-finish reconciliation — so this section summarises rather than repeats
it.

**7. The frontend packages it consumes.** `@inflowenger/flow-trace` for run movement on the
canvas; `@inflowenger/plugin-form-builder` for third-party plugin drawers.

**8. What FloMorphic did *not* have to build.** The durable engine, the resume machinery,
the isolation model, the credential minting, the plugin catalog, the three SDK languages.
Set against what it did build: a canvas, a backend, fifteen hook cases, four handler
families and five plugins.

**9. The Extensions feature.** Register a plugin from a Git repo; the backend mints a
credential and runs it, so it joins the ecosystem and contributes nodes to the palette.
This is a product feature built directly on
[spaces and scoped credentials](../06-architecture/spaces-and-isolation.md).

**10. What it proves, and what it does not.** It proves the seam works for a non-trivial AI
product with no runtime changes. It does not prove the runtime is ideal for every domain —
[Part V](../05-venapce/) is the second data point, and the
[coverage chapter's awkward cases](../02-fusion/coverage.md#the-awkward-cases-stated-plainly)
are the honest ledger.

## Source material

`flomorphic-api/README.md` and `inflow/`, `flomorphic-wapp/README.md`,
`FloMorphic/getting-started/docs/concepts.md` and `docs/architecture.md`,
`builtin-plugins/README.md`.

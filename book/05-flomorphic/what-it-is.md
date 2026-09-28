# What FloMorphic is

> **Status: outlined.**

**A platform for building your own agents and AI-native systems — low-code, on a runtime
you own.** Not a library you import. Not a SaaS you send your data to.

## Sections planned

**1. The AI harness idea.** Every capability the field has developed becomes an ordinary
node on one graph, over one shared context: `RAG`, `context engineering`, `agent loops`,
`tool use`, `MCP`, `long-term memory`, `working memory`, `guardrails & policy`,
`human-in-the-loop`, `multi-agent orchestration`, `out-of-model reasoning`,
`observability`.

**2. The part that matters most — it attaches to the system you already run.** Your backend
is not rewritten, replaced or migrated. It **joins**: it answers the graph, and the graph
calls it. A system in production for a decade can become AI-native without a rewrite, and
can grow from one laptop to a cluster without changing shape.
→ [Part VII — Attaching a system you already run](../07-architecture/attaching-legacy.md)

**3. Flow engineering as the discipline.** Not "flow" in the linear-automation sense.
Breaking work into states and transitions, with the model as a *bounded participant* and
control on a layer you can see. Underneath: a durable cyclic graph iterating over one
durable context object that survives a wait, an approval or a crash.

**4. Open at the seams that matter.** The palette is data plus a compiler hook; the AI nodes
are ordinary plugins with no privileged access; the backend is storage-agnostic; nothing in
the runtime is reserved for FloMorphic. There is no hosted-only capability holding the
interesting part back.

**5. What ships.** Canvas (`:8088`) + API (`:8026`) + builtin plugin nodes, as **one
image**, baked per-arch, building nothing at run time. Why one container is deliberate, and
how to run your own plugin set instead.

**6. The stack.** API: Go 1.27 + Fiber v3, SQLite + `sqlite-vec` via sqlc, the Vue Flow →
primitive compiler, and the `svc.*` handlers backing the store / HITL / continue nodes.
Canvas: Vue 3 + Vite + TypeScript + Vue Flow + Tailwind v4 + Pinia; runs standalone
(browser-local) or connected.

**7. The optional runtime.** Leave `INFLOW_INFRA_API` unset and the API runs CRUD-only —
enough to design and save workflows. Set it, and *Run* goes live.

**8. Installing it.** One command; the installer asks where to install, whether to use an
already-running platform or install a new one, and (optionally) port overrides.

## Source material

`FloMorphic/getting-started/README.md`, `docs/architecture.md`, `flomorphic-api/README.md`,
`flomorphic-wapp/README.md`.

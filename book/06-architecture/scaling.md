# Scaling out

> **Status: outlined.**

## The claim

The same artifact runs on one laptop and in a horizontally-scaled cluster **without changing
shape**. This chapter examines that claim component by component, and names where the real
bottlenecks are.

## Sections planned

**1. Scaling the engine.** Start a second Fractal; it registers with Infra;
`ReloadResources` picks it up; new runs round-robin across both. **No code change.** Because
the engine holds no domain state — everything durable is in the context document your
backend owns — instances need not coordinate with each other.

**2. Directing traffic.** Tags (`FRACTAL_TAGS`) and pinning (`PinResource`,
`PinResourceTag`) for when round-robin is not what you want: a GPU host, a network-segmented
instance, a debugging target.

**3. Scaling Infra.** Enterprise deployments may run Infra as a **cluster with multiple
instance endpoints** — which is why `INFRA_URL` is always explicitly required and never
assumed.

**4. Scaling plugins.** A plugin is your process, deployed on your cadence. Recorded open
question: multiple instances sharing a `PLUGIN_ID` for horizontal scale (NATS queue
groups?). Stated as open rather than answered.

**5. Scaling your backend.** The part you own, and — honestly — **the most likely
bottleneck**. The engine asks it for the flow and the context on essentially every step, so
context read/write throughput is the number that matters. Caching compiled flows is the
first optimisation; they change far less often than contexts.

**6. Where the real limits are.** A frank ordering: your context store first, then NATS
throughput, then plugin capacity, then engine count. The engine is rarely the constraint,
which is convenient given it is the part you cannot profile.

**7. What durability costs.** Every node completion is a context write. That is the price of
crash-resumability, and it is worth naming rather than discovering.

**8. Deployment shapes.** Single container (FloMorphic's baked image), split stacks,
Kubernetes considerations, and what has to be shared (the `inflow_net` network, the API
Secret Key).

## Source material

`inflow-fusion/docs/architecture.md`, `docs/infra.md`,
`FloMorphic/getting-started/docs/architecture.md`,
`inflow-plugin-sdk/docs/inflow-ecosystem.md`.

# Built under FloMorphic

> **Status: outlined.** This is the chapter that makes Part V worth its place in the book.

## The thesis

Venapce is **the first product built the FloMorphic way, end to end**: all of its business
logic lives in FloMorphic workflows, and Venapce itself is only the view.

## Sections planned

**1. What Venapce did not write.** No `IInflowService` implementation. No compiler hook. No
`NewProcess` calls. No node types. No plugin SDK integration for its core logic. It
consumes a host, rather than becoming one.

**2. What it did write.** A Go + Fiber backend, a Vue panel, a Postgres schema of two
tables, a BI proxy, and — the actual product — **a set of FloMorphic workflows, one per
feature**.

**3. "Every feature is a flow."** What that means operationally: adding a posture check is
authoring a workflow, not shipping a release. The domain expert takes the pen. This is the
platform's core promise cashed out at the product level rather than asserted.

**4. The bridge.** How Venapce reaches FloMorphic: the API URL, the shared JWT secret and
the infra host — settable from the panel, stored in Venapce's database, overriding the
environment. Note that this is an ordinary API integration, which is the point: tier 3
requires no special coupling.

**5. Where plugins still appear.** Venapce's *senses* are plugins — osctrl, Scrapli,
GitHub. So tier 3 does not mean "no plugins"; it means the **logic** is flows while the
**reach** is plugins. → [Collectors](collectors.md)

**6. The LLM node reasoning in the open.** Why an AI node inside a per-feature flow is a
different proposition from an AI assistant bolted onto a dashboard: the steps are drawn, the
ports are finite, and the run is recorded.

**7. What this generalises to.** The tier-3 pattern for anyone else: if a FloMorphic-shaped
host already exists in your domain, you may not need to build on the runtime at all. Build
on the product, and inherit its canvas, its palette, its plugins and its observability.

**8. The honest limits of tier 3.** You inherit the host's palette and its release cadence.
A capability the host does not expose is a plugin (tier 2) or a fork. Choosing tier 3 is
choosing speed over control, and it is the right trade far more often than teams assume.

## Source material

`Venapce/getting-started/README.md`, `plugin-catalog/docs/venapce.md`, the blog post
*Venapce: A Nervous System for Security Governance*.

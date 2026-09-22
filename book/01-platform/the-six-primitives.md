# The six primitives

> **Status: outlined.** The full reference is written in Part II — this chapter is the
> platform-side framing, not a duplicate.

## Purpose of this chapter

Part II documents the primitives as things a **compiler hook returns**. This chapter frames
them as things the **runtime executes**, and makes the closure argument on its own terms.

## Sections planned

**1. The set, and the compile/no-compile split.** Five compiled primitives; Plugin as the
one live process. Why that asymmetry is the whole extensibility story.

**2. The axis argument for closure.** Computation / decision / reach-your-own-system /
reach-the-world. Four axes, each with a primitive. Why the claim is about the axes being
exhaustive, not about six being aesthetically nice.

**3. What a product actually ships instead of primitives.** Fifty comparison operators,
an HTTP node, an approval gate — all authored, none implemented in the engine.

**4. The higher-level-node pattern.** A packaged extension ships a form; the user fills it
in; the hook lowers it. From the canvas it is a first-class custom node; underneath it is
still one of the six.

**5. Reading the claim skeptically.** What *would* falsify it — a capability with no
plausible lowering. Work through the usual candidates (parallel map over a collection,
timeouts, retries, sub-second scheduling) and show where each lands.

→ Full reference: [Part II — The primitive node reference](../02-fusion/node-primitives.md)

## Source material

`inflow-fusion/docs/nodes.md`, `inflow-fusion/docs/nodes/from-frontend.md`,
`FloMorphic/getting-started/docs/concepts.md`.

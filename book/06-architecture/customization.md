# Customization — the seams that are open

> **Status: outlined.**

A catalogue of every point where the platform is designed to bend, **and what each one
costs**. The second half of that sentence is the part most architecture documents omit.

## The seams

| Seam | What you change | Cost | Chapter |
| --- | --- | --- | --- |
| **The compiler hook** | your entire node vocabulary | one function | [II](../02-fusion/the-compiler-seam.md) |
| **A new compiler** | your entire authoring format | a package | [II](../02-fusion/compiling-a-yaml-dsl.md) |
| **Extrinsic handlers** | what your backend exposes to a graph | a few lines each | [II](../02-fusion/the-backend-contract.md) |
| **Storage** | where flows and contexts live | implement 3 methods | [II](../02-fusion/the-backend-contract.md) |
| **Plugins** | reach anything the primitives can't | a process | [III](../03-plugins/) |
| **A new SDK language** | who can write plugins | conformance work | [III](../03-plugins/plugin-sdk-spec.md) |
| **Plugin forms** | how a node is configured | JSON Schema + `x-inflow-ui` | [III](../03-plugins/forms-and-ui.md) |
| **Native node drawers** | a first-party node's UI | a frontend component | [IV](../04-flomorphic/builtin-nodes.md) |
| **Workflows on an existing host** | product behaviour, with no code | authoring | [V](../05-venapce/built-on-flomorphic.md) |
| **Event consumers** | what observability looks like | a subscriber | [II](../02-fusion/observing-a-run.md) |

## Sections planned

**1. Picking the right seam.** A decision tree: *can it be a flow? → a hook case? → an
extrinsic handler? → a plugin?* In that order, because the cost rises at each step.

**2. What is deliberately not customizable.** The primitive set, the `inflowv1` protocol,
the event schema, the backend contract. These are fixed **so that** everything above can
move — a stable core is what makes plugin portability and multi-product reuse possible.
Flexibility everywhere is flexibility nowhere.

**3. Versioning and drift.** `inflowv1` → a future `inflowv2` alongside it, not replacing
it. Event schema `v: 1` and the reject-unknown-version rule. What a product owes its users
when a flow definition changes under live runs.

**4. The extension pattern in a product.** How a host lets a user register a plugin from a
Git repo, mint a credential and run it, so it joins the ecosystem and contributes nodes to
the palette. A product feature assembled from spaces + scoped credentials + the catalog.

**5. Forking versus extending.** When the seams are not enough, and what you actually lose
by going around them.

## Source material

Parts II–V of this book; `inflow-fusion/docs/compilers/README.md`,
`plugin-catalog/docs/`, `FloMorphic/getting-started/docs/nodes.md`.

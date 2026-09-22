# Repositories

> **Status: outlined.** Links will be completed and verified before publication.

## Which repo answers which question

| You want to… | Read |
| --- | --- |
| Understand the whole mental model | this book, Parts 0–I |
| Know what each primitive is and how to build one | `inflow-fusion` → `docs/nodes.md`, `docs/nodes/` |
| See how *any* frontend node reduces to a primitive | `inflow-fusion` → `docs/nodes/from-frontend.md` |
| Understand infra, engines, backends and plugins together | `inflow-fusion` → `docs/architecture.md` |
| Get the exact REST endpoints and NATS subjects | `inflow-fusion` → `docs/infra.md` |
| Compile a canvas graph into engine nodes | `inflow-fusion` → `docs/compilers/` |
| Observe a running flow | `inflow-fusion` → `docs/logs.md` |
| Build a plugin | `plugin-catalog` → `docs/build-a-plugin.md`, then the SDK's own docs |
| Read the plugin wire protocol | `go-plugin-sdk` → `docs/protocol-inflowv1.md` |
| Design a plugin form with dependent fields | `plugin-catalog` → `docs/dependent-fields.md` |
| Render a plugin form in your own frontend | `inflow-js` → `packages/plugin-form-builder` |
| Animate a run on your own canvas | `inflow-js` → `packages/flow-trace` |
| Work on FloMorphic's API / canvas | `flomorphic-api` / `flomorphic-wapp` READMEs |
| Install the platform by itself | `Inflowenger/getting-started` |
| Install FloMorphic | `FloMorphic/getting-started` |
| Install Venapce | `Venapce/getting-started` |

## Normative sources

Where this book and a source repository disagree, **the source repository wins.** These are
the normative references, by subject:

| Subject | Normative source |
| --- | --- |
| Node structs and `Next` semantics | `inflow-fusion/models/flow.go` |
| The compiler contract | `inflow-fusion/docs/compilers/README.md` |
| The backend contract and wire reference | `inflow-fusion/docs/infra.md` |
| Tag routing semantics | `inflow-fusion/docs/routing.md` |
| The process event schema | `inflow-fusion/docs/logs.md` |
| The `inflowv1` protocol | `go-plugin-sdk/docs/protocol-inflowv1.md` |
| Job commands | `go-plugin-sdk/docs/jobs-and-commands.md` |
| Form builder / `x-inflow-ui` | `go-plugin-sdk/docs/form-builder.md`, `inflow-js` |
| The plugin listing bar | `plugin-catalog/CONTRIBUTING.md` |

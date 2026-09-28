# Builtin nodes are plugins

> **Status: outlined.**
>
> The single most load-bearing fact in Part V: **FloMorphic's builtin nodes are ordinary
> plugins.** Same SDK, same protocol, no privileged access. The only thing they do
> differently is skip the generic form builder.

## The claim, concretely

FloMorphic's AI capabilities are not built into the runtime, and not built into
FloMorphic's backend either. They are five independent Go modules in
[`FloMorphic/builtin-plugins`](https://github.com/FloMorphic/builtin-plugins), each
importing `go-plugin-sdk` exactly as a third-party plugin would.

| Plugin | Actions | What it does |
| --- | --- | --- |
| `llm` | `run` | A streamed LLM turn over OpenAI / OpenRouter / OpenAI-compatible / Gemini / Anthropic, **with tool-call routing to outbound ports**. |
| `mcp` | `run`, `call_tool`, `getToolsList` (meta) | An MCP client node. `run` drives a model over MCP tools; `call_tool` calls one tool with no model. |
| `cast` | `run` | Assembles a JSON object from key/value mappings, resolving `{{$.a.b}}` tokens against the live flow context. |
| `http` | `run` | An HTTP/REST request with connection config from a settings profile, resolving `{{$.a.b}}` in every string field. |
| `jev` | `run` | Evaluates a state template against typed questions (choice / score / noul) and routes each question's top answer to its outbound port. |

Each pins the SDK at a released tag in its **own** `go.mod`, builds independently, and reads
its infra connection from a local `.env.inflow`. **Nothing is shared at build time** — the
MCP node carries its own copy of the LLM provider glue so the two stay decoupled.

```sh
cd llm
go build -o bin/plugin .
./bin/plugin
```

That is the same command a stranger's plugin uses. There is no internal build, no linked
library, no back door into the API.

## The one real difference: the UI path

A plugin node needs a configuration UI. There are two ways to get one, and this is where
builtin and third-party diverge — **only here**.

| | Third-party plugin | Builtin node |
| --- | --- | --- |
| Declares a form over `@form` / `@settings` | yes | minimally, or not at all |
| Rendered by | the **generic form builder** — JSON Forms + `x-inflow-ui`, via `@inflowenger/plugin-form-builder` | a **bespoke native drawer** hand-built in the canvas app |
| Who writes the UI | the plugin author, as a JSON Schema | the product's frontend team, as a Vue component |
| Fidelity | whatever the schema vocabulary expresses | anything Vue can do |
| Cost to add | none — the form ships with the plugin | a frontend change and a release |

So: **a builtin node is a plugin without the UI builder.** Its execution path — the
`inflow.cpu.*` subjects, the job handshake, progress frames, context reads and commits,
`next_tags` routing — is byte-for-byte the protocol every other plugin speaks. Its
*configuration* path is bespoke, because a first-party node can afford a hand-built drawer
and a generic one cannot.

## Sections planned

**1. Why this asymmetry is the right trade.** A product's flagship nodes deserve a
first-class UI; an ecosystem of hundreds of plugins needs a UI that costs the author
nothing. Supporting both is why `x-inflow-ui` exists *and* why the canvas has native
drawers. Neither path is a downgrade of the other.

**2. What a builtin node gives up.** Portability of its UI. A native drawer lives in
FloMorphic's canvas; move the plugin to another host and it falls back to whatever form it
declares. The *node* is portable; the bespoke drawer is not.

**3. What it does not give up.** Nothing on the execution side. `llm` reached by another
host over `inflowv1` works, because it is a plugin.

**4. The evidence this is not theatre.** Third-party plugins in the catalog do things
builtin nodes do — hold connections, stream progress, route their own ports, read and write
context. No capability is fenced off. The
[coverage tables](../02-fusion/coverage.md#4-reaching-the-outside-world) show catalog
plugins and builtin nodes occupying the same rows.

**5. Tool-call routing, the shared mechanism.** `llm` binds its functions to outbound ports;
`jev` routes each question's top answer to `<question>.<option>`. Both are
`CmdNextFilter` — [tag routing](../02-fusion/tag-routing.md) — available to any plugin.

**6. Building your own "builtin".** If you build a product on the runtime, this is the
pattern: ship your flagship nodes as plugins with native drawers, and let everyone else's
plugins render through the form builder. You get a polished palette without closing the
door behind you.

**7. Why the installed image bakes them in.** They are what the canvas's stock nodes *run
as*, so the image always includes them; `PLUGINS_ENABLED=0` leaves a canvas + API with no
platform behind it. Baking a plugin into an image is a packaging choice, not a privilege.

## The takeaway

> Anything FloMorphic's own nodes can do, your plugin can do. The difference is who draws
> the form.

## Source material

`FloMorphic/builtin-plugins/README.md` and the five module directories,
`inflow-js/packages/plugin-form-builder`, `plugin-catalog/docs/concepts.md`.

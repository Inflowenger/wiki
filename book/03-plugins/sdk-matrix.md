# The SDK matrix — Go, Node, Python

> **Status: outlined.**

| Language | Package | Status |
| --- | --- | --- |
| **Go** | `Inflowenger/go-plugin-sdk` | **Stable** — the reference `inflowv1` implementation and the mainstream path. Go 1.27+. |
| **Node.js / TypeScript** | `@inflowenger/node-plugin-sdk` | **Stable** — on npm, Node 18+. Tracks the Go SDK feature-for-feature. |
| **Python** | `inflowenger-plugin-sdk` | **Beta** — on PyPI, Python 3.11+. The Python port of the Go SDK. |

> **Wire-identical.** You pick a language, not a different protocol. A plugin written in any
> of the three is indistinguishable to the runtime.

## Sections planned

**1. Choosing.** Go for the reference behaviour and the richest examples; Node where the
ecosystem you are wrapping is JS; Python where it is a data/network library (Scrapli, cloud
SDKs).

**2. The same plugin, three times.** One small plugin written in each SDK, side by side, so
the shared shape is visible.

**3. What "tracks feature-for-feature" means, and where it does not yet.** An honest
per-feature table across the three.

**4. Writing an SDK in a fourth language.** `inflowv1` is a plain NATS protocol; the SDK is
a convenience. → [Plugin SDK specification](plugin-sdk-spec.md) is the conformance list.

**5. Building with an AI coding agent.** The Go SDK ships an Agent Skill
(`skills/inflow-plugin/SKILL.md`) teaching Claude Code and similar tools to use the SDK
correctly. Because the SDK is imported as a library, install the skill in *your* plugin
project:

```bash
mkdir -p .claude/skills
cp -r "$(go env GOMODCACHE)"/github.com/\!inflowenger/go-plugin-sdk@*/skills/inflow-plugin .claude/skills/
```

**6. The documentation rule for plugin repos.** If your repo ships a `SKILL.md`, a
`MANUAL.md`, or any file an agent or person reads to start the plugin, it **links** to your
README's `## Run` section rather than restating a command — one source of truth, so the
agent, the human and the host's generated one-liner all start it the same way.

## Source material

`plugin-catalog/docs/sdks.md`, `plugin-catalog/README.md`, the three SDK repositories.

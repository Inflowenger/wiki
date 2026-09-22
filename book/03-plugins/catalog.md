# The plugin catalog

> **Status: outlined.** The live catalog is
> [`Inflowenger/plugin-catalog`](https://github.com/Inflowenger/plugin-catalog); this
> chapter explains what it is *for*, and the table below will drift — treat the repo as
> authoritative.

## Why a catalog matters architecturally

A plugin targets the **`inflowv1` protocol**, not a product. So every plugin in the catalog
works on *any* host that implements the protocol. A new product built on the runtime starts
with a full integration list rather than an empty one — which is the single largest
practical consequence of the protocol-not-product decision.

## Snapshot

| Plugin | SDK | Runs on |
| --- | --- | --- |
| ClickHouse · MySQL · Qdrant | Node | any host |
| Jira · MongoDB · Postgres | Go | any host |
| Scrapli *(beta)* | Python | any host |
| Gmail · Google Workspace · Telegram · GitHub *(beta)* | Node/Go/Python | **FloMorphic only** ★ |
| osctrl | Go | **Venapce** |

★ These reach FloMorphic's central **Connect / OpenConnector** proxy over the
`flomorphic.svc.oc.*` NATS subjects, which only FloMorphic provides. The dependency is
recorded as `hostDependency` in the catalog's `index.json`.

## Sections planned

**1. "Any host" vs. a host dependency.** What makes a plugin portable, and the honest
exception — a plugin that needs a host-specific service runs on that host alone. Why
OpenConnector exists (centralised OAuth) and what it costs in portability.

**2. Running a plugin.** The standard command per language; the `.env.inflow` triad
(`PLUGIN_ID`, `INFRA_CRED`, `INFRA_URL`); and, on a host that provisions for you, the
generated one-liner plus the `./plugin.sh` helper
(`start · stop · restart · status · logs · update`).

**3. Getting listed.** The bar, the entry template, `plugins/index.json`, the PR. The plugin
stays in your account; the catalog only links.

**4. Trust.** A listing is not an endorsement or a security review. What a consumer should
check themselves.

**5. Machine-readable.** `plugins/index.json` as the source a host reads to populate an
extensions browser.

## Source material

`plugin-catalog/README.md`, `CONTRIBUTING.md`, `docs/publishing.md`, `docs/run-a-plugin.md`,
`plugins/index.json`.

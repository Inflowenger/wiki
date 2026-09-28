# Collectors and the plugin roadmap

> **The senses.** Venapce's collectors return **raw data frames. No opinions, no scoring,
> no stored secrets.** All judgment happens one layer up, in FloMorphic workflows.
>
> The live roadmap is `plugin-catalog/docs/venapce.md` and `docs/security-collectors-plan.md`
> — treat those as authoritative; this chapter explains the shape.

---

## Why "no opinions" is load-bearing

A collector that scores its own findings has made the judgment **unauditable and
unchangeable**. You cannot see why it decided, and you cannot alter the decision without a
vendor release.

Returning frames keeps the decision in a flow the operator can read and edit. It is the
same argument as [tag routing](../02-fusion/tag-routing.md) one layer down: draw the
possibilities, decide at runtime, record the reason.

Concretely, it is why `osquery.query` returns **the rows the node reported** and nothing
else. Whether a row is a finding, at what severity, with what confidence, is a `stage` row
and a flow — not a return value.

---

## Two kinds of sense, and the distinction matters

Venapce reaches the world two ways, and they are worth keeping apart because only one of
them is "a collector plugin" in the catalogue sense.

### 1. In-process — Venapce's own plugin

The **host layer** — the osquery fleet — is reached by the `inflowv1` plugin that ships
*inside* `venapce-api` ([see the previous chapter](built-on-flomorphic.md#3-the-correction-venapce-is-its-own-plugin)):

| Action | |
| --- | --- |
| `osquery.query` | Dispatch osquery SQL to **one enrolled node** via osctrl — run → poll → collect |
| `osquery.queryByTags` | The same, to **everything carrying a tag** — a fleet-wide sweep |
| `osquery.meta.nodes` / `.tags` / `.environments` | The pickers behind the query form |

It is in-process because the osctrl client, its credentials and the database the rows land
in are already the backend's. There is no settings profile and nothing to provision.

> **"Your fleet is a live database."** This is the osquery model, and it is the whole
> reason the host layer works this way: *ask the fleet a question* rather than consume a
> vendor's pre-decided alert list. `queryByTags` is that sentence as an action — one SQL
> statement, every Linux box carrying a tag, rows back.

### 2. Catalogue collectors — ordinary `inflowv1` plugins

Everything else Venapce reaches comes from plugins in
[the catalogue](../03-plugins/catalog.md), with nothing security-specific about them:

| Collector | Reaches | Via |
| --- | --- | --- |
| **Scrapli** | network devices | SSH / NETCONF, via the Python SDK *(beta)* |
| **GitHub (OpenConnector)** | code and repository posture | the GitHub OC plugin *(beta)* |
| *(wave 1)* | AWS · Azure · GCP, **AWS first** | under the security-collectors plan |
| *(later waves)* | OpenSearch · LDAP · VulnIntel · Wazuh · NetBox | planned |

These are `inflowv1` plugins like any other — which is why the catalogue's rules apply
unchanged, and why a plugin written for something else is available to Venapce the day it
is published.

> The network-device study is also a small piece of platform history: **its need for Python
> is what drove the Python SDK**, on which Scrapli then shipped. A product requirement
> produced an SDK language, and the SDK language is now available to everybody.

---

## The feasibility-study method

A collector is not built until a study says what it would cost. The studies
([osctrl](../99-appendix/repositories.md), GitHub OC, the security-collectors plan) are
**published rather than internal**, which is unusual enough to be worth stating as method:

1. What does this system actually expose, and at what granularity?
2. What would a frame from it look like — and is it genuinely opinion-free?
3. What does the SDK need that it does not have? *(This is the question that produced the
   Python SDK.)*
4. What does the operator have to provision before any of it works?

Publishing them means a reader can check the reasoning behind "shipped", "beta" and
"planned" rather than trusting a roadmap colour.

---

## Where a collector's output goes

Worth tying back, because it is the seam the whole product turns on:

```
  collector plugin          FloMorphic flow                 Venapce
  ───────────────►  frames  ──────────────►  judgment  ───► stage / findings / issues
   no opinions              the rules you drew              rows you can act on
```

A collector writes nothing to Venapce directly. It returns frames; a flow decides; the
flow writes through `db.stages.upsert` / `db.findings.upsert` / `db.issues.upsert`. The
level it writes to is [the flow author's choice](what-it-is.md#the-data-model-a-three-level-pipeline-and-it-is-optional),
which is why the pipeline is optional.

And a package of such flows, with the collector it needs declared in `requires.plugins`,
is an [operation](operations.md).

---

## Next

Part VI is complete. Continue to:

- **[Part VII — Architecture in Practice](../07-architecture/)** — scale, isolation and the
  seams that stay open.

**Source material:** `plugin-catalog/docs/venapce.md`, `docs/security-collectors-plan.md`,
`docs/osctrl-feasibility.md`, `docs/github-oc-feasibility.md`;
`venapce-api/internal/plugin/osquery/`, `internal/osctrl/query.go`; and the blog post
*Your Fleet Is a Live Database. Ask It Something.*

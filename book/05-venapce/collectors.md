# Collectors and the plugin roadmap

> **Status: outlined.** The live roadmap is `plugin-catalog/docs/venapce.md` and
> `docs/security-collectors-plan.md` — treat those as authoritative; this chapter explains
> the shape.

## The senses

Venapce's collectors return **raw data frames. No opinions, no scoring, no stored secrets.**
All judgment happens one layer up, in FloMorphic workflows. That division is the design.

| Collector | Reaches | Via |
| --- | --- | --- |
| **osctrl** | hosts / endpoints | osquery agents, through an osctrl plugin |
| **Scrapli** | network devices | SSH/NETCONF, via the Scrapli plugin *(beta)* |
| **GitHub (OpenConnector)** | code and repository posture | the GitHub OC plugin *(beta)* |
| *(planned)* | the three major clouds | under feasibility study |

## Sections planned

**1. Why "no opinions" is load-bearing.** A collector that scores its own findings has made
the judgment unauditable and unchangeable. Returning frames keeps the decision in a flow the
operator can read and edit. This is the same argument as
[tag routing](../02-fusion/tag-routing.md) at the data layer.

**2. Your fleet is a live database.** The osquery model: ask the fleet a question rather than
consume a vendor's pre-decided alert list.

**3. The feasibility-study method.** What each study establishes before a collector is built,
and why the studies are published rather than internal.

**4. The roadmap.** What is shaped, what is under study, what is requested.

**5. Collectors as ordinary plugins.** Nothing about a security collector is special — they
are `inflowv1` plugins like any other, which is why the catalog's rules apply unchanged.
→ [Part III — The plugin catalog](../03-plugins/catalog.md)

## Source material

`plugin-catalog/docs/venapce.md`, `docs/security-collectors-plan.md`,
`docs/osctrl-feasibility.md`, `docs/github-oc-feasibility.md`, and the blog post
*Your Fleet Is a Live Database. Ask It Something.*

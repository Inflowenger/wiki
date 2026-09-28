# Operations — features as installable flow packages

> A feature of Venapce is not a release. It is **a folder** — a manifest and a few workflow
> exports — that installs into FloMorphic with your organisation's values substituted in.

This is the newest idea in Part VI and the one most likely to matter outside security. Once
your product's business logic lives in workflows, *shipping a feature* stops being a
deployment and becomes a **package install** — and that needs a package format, a manifest,
readiness checks and an upgrade path, exactly like any other package manager.

Venapce built one. It is worth reading whether or not you care about posture management.

---

## The problem it solves

Tier 3 buys you enormous speed and hands you one problem in exchange: **your users now have
to author the flows.**

"Every feature is a flow" is liberating for the domain expert who wants to change a rule,
and useless for the operator who just wants their Linux fleet audited this afternoon. The
canvas is not the deliverable. The *working feature* is.

An operation closes that gap:

```
  a package                    installed                    running
 ┌──────────────┐            ┌────────────────┐           ┌──────────────┐
 │ operation    │  params    │  flows landed  │  schedule │  rows into   │
 │   .json      │ ────────►  │ in FloMorphic  │ ────────► │  stage /     │
 │ flows/*.json │  readiness │  (bound by key)│  or on-row│  findings /  │
 │ README.md    │  checks    │                │           │  issues      │
 └──────────────┘            └────────────────┘           └──────────────┘
```

---

## The package

A folder holding an **`operation.json`** manifest plus every file it names:

```
linux-fleet-http-audit/
├── operation.json
├── README.md
├── flows/
│   └── http-audit.flow.json     ← a FloMorphic workflow export
└── docs/
    └── http-audit.md
```

The manifest format is defined once, as a JSON Schema in the panel
(`venapce-wapp/public/schemas/venapce-operation.schema.json`); the Go side reads the fields
it acts on and **keeps the rest as-is, so a newer manifest with extra keys still installs**.

### The manifest

| Key | |
| --- | --- |
| `schema` | must be `1` |
| `id` | a slug — **one install per id** |
| `name`, `version` | version is semver, validated |
| `description`, `readme`, `tags`, `scale` | catalogue metadata (`scale`: `vps` \| `fleet` \| `smb` …) |
| `params` | the operator knobs — see below |
| `flows` | one or more workflow specs — see below |
| `requires` | what the estate must already provide — see below |

Validation returns **every problem at once** and returns the manifest anyway, so the panel
can preview a package it is also complaining about.

### `params` — what adapts a package to one organisation

```json
"params": {
  "env":       { "type": "string",   "label": "osctrl environment", "default": "default", "required": true },
  "fleetTags": { "type": "string[]", "label": "Fleet tags", "default": ["default"], "required": true }
}
```

A spec carries `type`, `label`, `description`, `default`, `required`, `enum` and
**`secret`**. Secret params are stored AES-GCM encrypted and **never returned** by the API.

Substitution into the flow files has one rule worth knowing:

- A string that **is** a `${param.x}` placeholder is replaced by the param's value **with
  its own JSON type** — so an array param can stand in for a `tags` list.
- A placeholder **inside** a longer string is interpolated as text.
- An **unknown** param is left as it is, so the author sees it on the canvas rather than
  getting a silent empty string.

`Effective()` resolves the operator's value over the manifest default; `Missing()` lists
required params that have neither.

### `flows` — the workflows, and their roles

```json
{ "key": "http-audit", "file": "flows/http-audit.flow.json",
  "role": "entry", "step": 1, "schedule": "0 */6 * * *",
  "writes": ["stage"], "doc": "docs/http-audit.md" }
```

| Field | |
| --- | --- |
| `key` | a slug, unique in the package — **this is the binding identity** |
| `file` | the workflow export |
| `role` | **`entry`** (run it, on a schedule or by hand) · **`on-row`** (run it against a pipeline row) · **`helper`** (called by the others) |
| `step` | ordering, for a multi-stage feature |
| `subject` | which pipeline levels an `on-row` flow applies to |
| `schedule` | a cron expression for an `entry` flow |
| `writes` | which tables this flow writes — the package declaring its own effects |
| `doc` | per-flow documentation |

`role` is the part that makes a package usable rather than merely installable: it tells the
panel *how* each flow is meant to be invoked, so an operator gets a Run button in the right
place instead of a list of graphs.

### `requires` — the readiness check

```json
"requires": {
  "osctrl":  { "required": true, "targets": ["linux"], "minNodes": 1 },
  "plugins": [{ "name": "venapce", "actions": ["osquery.query", "osquery.queryByTags", "db.stages.upsert"] }]
}
```

Four kinds of requirement — `osctrl`, `plugins`, `settings`, `operations` (another package)
and `datasets` — and the important one is `plugins.actions`.

**Before installing, Venapce asks the target FloMorphic which plugin actions it can
actually run** (its extension table's action rows), and reports what is missing:

```go
type MissingAction struct {
    Action string   // the action a workflow node calls
    Plugin string   // the plugin that provides it
    Repo, Ref, Subdir string  // where to get it
    Nodes  []string  // which nodes in the file call it
}
```

So a failed install says *"node X calls `jira.issues.create`, which needs the `jira` plugin
from this repo"* — not *"compile error"*. A package that declares its requirements is a
package whose failure is actionable.

---

## Installing

`POST /api/operations/:id/flows/:key/install` lands one flow, via FloMorphic's
`POST /flow/import` — the REST twin of the editor's Import dialog.

```go
type ImportResult struct {
    OK             bool
    Flow           Flow
    Problems       []ImportProblem   // error = node/edge dropped, warn = kept as-is
    MissingActions []MissingAction
    CompileError   string
    DryRun         bool
}
```

Three properties make this safe enough to be a product feature:

- **`dryRun`** — plan the install and report problems without landing anything.
- **FloMorphic re-stamps every plugin node from its own extension table**, so a file may
  come from *any* install. That is what makes packages portable between organisations.
- **Passing an `id` overwrites an existing flow** — which is what a re-install or an
  upgrade is.

The result is recorded in `operations.bindings`:

```json
"bindings": { "http-audit": { "flowId": "flow_…", "flowTitle": "…", "installedAt": "…" } }
```

**Binding by manifest `key`, not by title**, is what lets an upgrade replace the right flow
even after the operator renamed it on the canvas.

## Sources and upgrades

`operations.source` records where a package came from — `url` \| `folder` \| `paste` — so
`POST /api/operations/:id/update` can re-read it and re-install. The panel can read a public
GitHub folder itself (`raw.githubusercontent.com` is CORS-open) and send the bundle; the Go
side is the server-side twin for the URL path.

The whole package — the manifest **and every file it names, as text** — is stored in
`operations.files`. So a flow can be re-installed at any time with the operator's params
substituted, without reaching the network again.

## Packing: the round trip

The reverse direction is the one that closes the loop, and it is why this is a product
feature rather than a deployment script.

`POST /api/operations/pack` takes flows an author built **on the FloMorphic canvas** and
wraps them into a package:

- Their **exports** become `flows/<key>.flow.json`.
- A **manifest is derived from what the exports declare** — their plugin actions become
  `requires.plugins`.
- A **README scaffold** is generated naming the mission and each flow.
- The result installs like any other package, **already bound to the flows it came from**,
  and zips for a catalogue (`GET /api/operations/:id/package.zip`).

So: draw it, pack it, publish it, and someone else installs it with their own fleet tags.
A user-authored feature becomes a distributable artefact without anybody writing code.

---

## Why this generalises

Strip the security vocabulary and the pattern is:

> **When your product's logic is data, your features become packages — and a package
> manager is a better shipping mechanism than a release.**

Everything in the design is domain-neutral:

| The piece | The general problem it solves |
| --- | --- |
| A manifest with `id` + semver | identity and upgrade |
| `params` with types, defaults and secrets | one package, many organisations |
| `role` on each flow | how a user is meant to invoke it |
| `requires` + a readiness probe | failing *before* install, with an actionable message |
| `bindings` by key | upgrading a flow the user has since renamed |
| `writes` | a package declaring its own effects |
| Pack-from-canvas | users becoming authors, and authors becoming publishers |

Any tier-3 product on FloMorphic — or any tier-1 product built on
[`inflow-fusion`](../02-fusion/) whose users author flows — has this problem the moment it
has more than one customer. Venapce's answer is about eight hundred lines of Go and a JSON
Schema.

---

## Next

- **[Collectors](collectors.md)** — where the data an operation processes comes from.

**Source material:** `venapce-api/internal/operations/` (`manifest.go`, `params.go`,
`source.go`, `pack.go`), `internal/flomorphic/install.go`,
`internal/httpapi/operations.go`, `venapce-wapp/public/schemas/venapce-operation.schema.json`,
`venapce-wapp/examples/operations/linux-fleet-http-audit/`.

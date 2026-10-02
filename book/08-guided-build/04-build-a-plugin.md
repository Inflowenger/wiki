# 4 · Filling the gap: build a plugin

> **Forty-five minutes.** The catalog has no ArangoDB plugin. This chapter builds one, runs
> it, and watches it appear on the palette *and in the AI build prompt*. The language is Go;
> Node and Python SDKs are wire-identical and the shape is the same.

## Before you write any code: is a plugin the right answer?

[Chapter 3](03-identify.md#the-gap-arangodb) made the call for Northwind. Here is the test
in general, because building a plugin you did not need is the commonest waste in this
ecosystem:

| Build a plugin when | Use the HTTP node when |
| --- | --- |
| Several flows will reach the same system | One flow makes one call |
| There is a **connection** to hold — a pool, a session, credentials | Each call is independent and stateless |
| A flow author should see a **form**, not a URL and a hand-built JSON body | The URL *is* the interface, and it is simple |
| You want **meta RPCs** — "list the collections", "test the connection" | There is nothing to look up |
| The capability should exist for everyone, including assistants designing flows | It is a one-off |

That last row is the one people underrate, and this chapter ends by demonstrating it.

## 1 · Provision — without touching Infra

A plugin needs an identity in a **space** (a NATS account managed by Infra). That is what
makes its `PLUGIN_ID` reachable and scopes what it may touch.

You could register it with Infra directly. Do not — use **Extensions → New extension →
bring your own**, exactly as in chapter 3. FloMorphic provisions it and hands back the three
values a plugin needs, as a downloadable `.env.inflow`:

```env
# .env.inflow
PLUGIN_ID=aa-bbb-ccc-dddd
INFRA_CRED=LS0tLS1CRUdJTiBOQVRTIFVTRVIgSldULS0t...   # base64 of the .creds blob
INFRA_URL=localhost:4222
```

`INFRA_CRED` is a standard decorated NATS `.creds` blob, base64-encoded. The SDK decodes it,
reads the account from the JWT, and connects with automatic reconnect.

> **Commit `.env.inflow.example`, never `.env.inflow`.** Put the real file in `.gitignore`
> on day one. `INFRA_CRED` is a live credential.

## 2 · Scaffold

```bash
mkdir arangodb-plugin && cd arangodb-plugin
go mod init github.com/northwind/arangodb-plugin
go get github.com/Inflowenger/go-plugin-sdk@latest
```

The module path must match where you will push it; `go get` on a mismatched path fails for
anyone importing your code.

`main.go` — construct, declare, `Start()`, then **block**:

```go
package main

import (
	"log"
	"os"
	"os/signal"
	"syscall"

	"github.com/Inflowenger/go-plugin-sdk/sdkv1"
)

const version = "v0.1.0"

func main() {
	envFile := os.Getenv("INFLOW_ENV_FILE")
	if envFile == "" {
		envFile = ".env.inflow"
	}

	p, err := sdkv1.NewPlugin(sdkv1.WithDotEnv(envFile))
	if err != nil {
		log.Fatalf("cannot connect to infra (%s): %v", envFile, err)
	}

	p.Intro(sdkv1.PluginIntro{
		Name:     "ARANGODB",          // what the palette shows
		Author:   "Northwind Platform",
		Version:  version,
		Settings: &settingsForm,       // declared in step 4
	})

	p.RequiredParams(&sdkv1.Settings{FormBuilder: settingsForm, SubmitHandler: validateConn})

	p.AddAction(sdkv1.Action{
		Method:         "arango.query",
		Title:          "Run AQL Query (read)",
		Description:    "Run a read-only AQL query and return its rows",
		Form:           queryForm,
		RequestHandler: handleQuery,
	})
	p.AddAction(sdkv1.Action{
		Method:         "arango.graph.traverse",
		Title:          "Traverse Graph",
		Description:    "Walk a named graph from a start vertex and return the reached vertices",
		Form:           traverseForm,
		RequestHandler: handleTraverse,
	})

	p.AddMeta(sdkv1.Meta{Method: "arango.meta.ping", RequestHandler: metaPing})
	p.AddMeta(sdkv1.Meta{Method: "arango.meta.graphs", RequestHandler: metaGraphs})

	if err := p.Start(); err != nil {
		log.Fatalf("start: %v", err)
	}

	// Start() only wires subscriptions — the process must stay alive to serve them.
	stop := make(chan os.Signal, 1)
	signal.Notify(stop, syscall.SIGINT, syscall.SIGTERM)
	<-stop
	log.Println("shutting down")
}
```

```bash
go run .
```

On startup the SDK logs **every subject it subscribed to**. That log is your confirmation
the plugin registered with Infra. No log, no registration — this is the same diagnostic from
chapter 3, and it is the one that saves the most time.

## 3 · Name things

Two naming decisions, both visible to users forever:

- **`Intro.Name`** is what the palette shows: `ARANGODB`. Uppercase, product-shaped, no
  version.
- **`Action.Method`** is `<domain>.<noun>.<verb>`: `arango.query`,
  `arango.graph.traverse`, `arango.execute`. Meta methods take `.meta.`:
  `arango.meta.ping`.

Match the catalog's existing shape and nobody has to learn yours — Postgres uses
`postgres.query` / `postgres.execute` / `postgres.table.create`; MySQL adds
`mysql.record.insert`.

## 4 · The connection lives in settings, never in the action

The plugin **declares** what a connection needs; the platform stores the values and ships
the bound profile with every call as `body.settings`.

```go
var settingsForm = sdkv1.FormBuilder{
	Jsonschema: settingsSchema, // endpoint, database, user, password
	Jsonui:     settingsUI,
	SubmitTo:   "arango.meta.ping",
}

func validateConn(r sdkv1.Request) sdkv1.Response {
	cfg, err := sdkv1.CastRequestTo[Conn](r.Data)
	if err != nil {
		return sdkv1.Response{Error: "unreadable settings"}
	}
	if err := ping(cfg.Body); err != nil {
		return sdkv1.Response{Error: "cannot reach ArangoDB: " + err.Error()}
	}
	return sdkv1.Response{Data: map[string]any{"ok": true}}
}
```

> **The submit handler is a validator, not a store.** It checks the submitted values against
> the live service and answers ok/error. The platform does the storing. A plugin stores no
> user credentials — that rule is what lets one process serve many tenants.

Three habits that separate a plugin people can use from one they file bugs against:

**Never put connection fields in an action form.** No tokens, no base URLs. Worth an actual
unit test: walk every action's schema and assert no `settings` property exists.

**Match settings keys leniently.** Profiles are typed by hand as key/value rows. Ignore
case, spaces, dashes and underscores, and accept the obvious synonyms (`endpoint` / `url` /
`host`). Write one `parseSettings(map[string]any) (Conn, error)` and route every action
through it.

**Pool clients per connection, keyed on the resolved settings** — never a package-level
singleton. One process serves many accounts at once, and a global client leaks one tenant's
connection into another's job.

When no usable connection arrives, fail with the *fix*, not the symptom:

```go
job.DoneWithError("no connection: pick a settings profile in the node drawer")
```

## 5 · The forms

Without a form, users hand-write JSON. With one, they get a real drawer. You can write the
JSON Schema and the UI Schema by hand, or generate both from one declaration so they cannot
drift:

```go
import "github.com/Inflowenger/go-plugin-sdk/formkit"

var traverseForm = formkit.New("Traverse Graph").Add(
	formkit.Text("graph", "Graph name").Required(),
	formkit.Text("startVertex", "Start vertex (_id)").Required(),
	formkit.Text("direction", "Direction"),      // outbound | inbound | any
	formkit.Text("maxDepth", "Max depth"),
).Build()
```

`formkit` is adopt-per-form: nothing in `sdkv1` depends on it, its output is ordinary JSON
Schema + UI Schema text, and you can hand-write the next form.

### Make the graph name pickable

`arango.meta.graphs` is the half of the feature that makes the drawer pleasant: a button on
a control that calls the meta RPC and writes the answer back into the form.

```jsonc
{
  "type": "Control",
  "scope": "#/properties/graph",
  "x-inflow-ui": {
    "action": { "name": "pluginFn", "fn": "arango.meta.graphs" },
    "button": { "position": "append", "label": "List" }
  }
}
```

The host sends the whole form plus the bound settings profile, and applies what comes back:
an **object** is a patch of `field → value`; anything else is written to the button's own
control. So the handler returns `map[string]any{"graph": "assets"}` — **not** an
`sdkv1.Response`, whose `{data, error}` envelope would be patched in as two fields called
`data` and `error`.

One trap specific to meta handlers: **do not use `CastRequestTo` in one.** It unmarshals the
action envelope (`{"_registry":…, "body":{…}}`), but a meta call made from a form arrives
**flat** — fields at the top level. You get a zero-valued struct and an empty settings map,
with no error to tell you. Decode tolerantly instead.

## 6 · The handler

```go
func handleTraverse(job sdkv1.Job) {
	req, err := sdkv1.CastRequestTo[TraverseInput](job.Req.Data)
	if err != nil {
		job.DoneWithError(err.Error())
		return
	}

	conn, err := parseSettings(req.Body.Settings)
	if err != nil {
		job.DoneWithError("no connection: pick a settings profile in the node drawer")
		return
	}

	job.Progress(20, sdkv1.Frame{Title: "connecting", Content: conn.Endpoint})

	rows, err := traverse(conn, req.Body)
	if err != nil {
		// Keep the payload and the node's scope alive through the failure.
		job.DoneWithErrorData("traversal failed: "+err.Error(),
			map[string]any{"graph": req.Body.Graph, "start": req.Body.StartVertex})
		return
	}

	job.Progress(80, sdkv1.Frame{Title: "reached", Content: fmt.Sprintf("%d vertices", len(rows))})
	job.Done(map[string]any{"vertices": rows, "count": len(rows)})
}
```

Four things worth noticing, because they are the whole protocol:

| Call | What it means for the flow |
| --- | --- |
| `job.Progress(pct, Frame)` | Streams to the canvas live. Free observability — use it |
| `job.Done(data)` | Progress hits 100 and `data` is committed as the node's output, at the node's `key` |
| `job.DoneWithError(msg)` | Reported and committed as a failure. **The flow continues** unless the run is set to stop on error |
| `job.DoneWithErrorData(msg, data)` | Same, but the payload survives — so a downstream Rule node can route on *why* it failed |

The job also reaches into the running flow: `job.CmdGetScope("$.customer.tier")` reads
context outside the node's own scope, and `job.CmdSetOnPath` writes into it. Use them
sparingly — a plugin that writes all over the context is a plugin nobody can reason about.

### What a plugin may not do

`CmdStopFlow` was **removed** from the protocol. Flow control belongs to the graph and to
the user, not to a plugin. A plugin reports outcomes; it does not decide routing — except
through `next_tags`, which selects among ports the *author* drew.

If your action genuinely has branches — "found" vs "not found" — declare them:

```go
Outbound: []sdkv1.OutboundPort{
	{Title: "found",     Tags: []string{"found"},     Description: "at least one vertex reached"},
	{Title: "not found", Tags: []string{"not-found"}, Description: "the start vertex has no edges"},
},
```

The frontend then renders one output port per entry, and the handler calls
`job.CmdNextFilter([]string{"found"})` to fire the branch it means. That is ordinary
[tag routing](../02-fusion/tag-routing.md) — the same mechanism the LLM node uses to turn a
model's function call into an edge. It is optional; leave it nil for a single-output action.

## 7 · Run it, and watch three things happen

```bash
go run .
```

1. **The subscription log.** `inflow.v1.<PLUGIN_ID>.*` subjects, listed at startup.
2. **The Extensions card goes live.** The card probes the plugin's `@intro` through the
   backend's `inflowv1` proxy — asked live, never stored.
3. **The palette grows.** `ARANGODB` appears with one card per action, each with the form
   you declared.

And then the thing this chapter was really for:

> ### The payoff: the assistant now knows it exists
>
> Open **AI build** on any canvas and look at the generated prompt. It now has a
> **"Plugins available"** section listing `ARANGODB`, its two actions, and **each action's
> parameter schema**. The same list reaches an MCP client through
> `flo_get_design_guide`.
>
> From this moment, an assistant designing a flow in this install can drop a node with
> `kind: "plugin"`, `data: {"pluginId": "<id>", "action": "arango.graph.traverse"}` and
> pre-fill `data.body` from the schema — instead of inventing an integration that is not
> here.
>
> That is the difference between a connection and a capability, and it is why chapter 3
> chose a plugin over an HTTP node.

## 8 · Let an assistant write it with you

The Go SDK ships an **Agent Skill** that teaches Claude Code and similar tools to use the
SDK correctly. Because the SDK is imported as a library, install the skill into *your*
plugin project:

```bash
mkdir -p .claude/skills
cp -r "$(go env GOMODCACHE)"/github.com/\!inflowenger/go-plugin-sdk@*/skills/inflow-plugin .claude/skills/
```

Then the build prompt is ordinary:

```text
Build an inflowv1 plugin for ArangoDB with the go-plugin-sdk, following the
inflow-plugin skill.

- Node name ARANGODB. Actions: arango.query (read-only AQL),
  arango.execute (write AQL), arango.graph.traverse.
- Connection in the SETTINGS form only — endpoint, database, user, password.
  Never in an action form. Add arango.meta.ping as SubmitTo.
- Add arango.meta.graphs so the drawer can list graph names into the form.
- Generate forms with formkit so schema and UI cannot drift.
- Pool the client keyed on resolved settings, not a package singleton.
- Write .env.inflow.example and gitignore .env.inflow.

Then write the README with a `## Run` section showing `go run .` verbatim.
```

The last line matters more than it looks. **If your repo ships a `SKILL.md`, a `MANUAL.md`,
or any file an agent or a person reads to start the plugin, it links to README § Run rather
than restating a command.** One source of truth, so the agent, the human and FloMorphic's
generated one-liner all start the plugin the same way — and the Extensions one-liner works
on your repo at all.

## 9 · Ship checklist

Before this counts as done:

- [ ] `.env.inflow.example` committed at the repo root; `.env.inflow` gitignored.
- [ ] Starts with its language's **standard command** from the repo root — `go run .` —
      with no extra flags or environment beyond the three values.
- [ ] A `## Run` section in the README showing that command verbatim.
- [ ] Subscriptions logged at startup.
- [ ] No action form contains a connection field.
- [ ] Every action fails with a fix, not a symptom.

A repository that follows this is what the Extensions installer expects. One that does not
will fail the one-liner at "detect language", and will fail the catalog's listing bar.

## 10 · Getting it listed (optional)

The catalog only links — **the plugin stays in your account.** Getting listed is an entry
file from the template plus a row in `plugins/index.json` and a PR. A listing is not an
endorsement or a security review, and a consumer should check the repo themselves.

Northwind would not list this one: it is specific to their Arango schema. A generic
ArangoDB plugin would be welcome, and that is the honest line between the two.

## What this costs you

State it plainly, because chapter 3 promised a cost column:

- **A process to run.** The plugin is not inside FloMorphic. It is a service you deploy,
  monitor and restart. `plugin.sh` covers a laptop; production wants systemd, Docker or a
  Kubernetes Deployment, and the script's own command is what you lift into them.
- **A UI that is only as good as your schema.** A builtin node like LLM gets a bespoke Vue
  drawer hand-built in the canvas. Yours renders through the generic form builder. That
  ceiling is real — and it is the *only* asymmetry: the execution path is byte-for-byte the
  protocol every builtin node speaks. → [Builtin nodes are plugins](../05-flomorphic/builtin-nodes.md)
- **A version to keep moving.** The SDK is pre-1.0. Pin it at a released tag in your own
  `go.mod` and update deliberately.

## Checkpoint

`ARANGODB` on the palette, live in Extensions, and listed in the AI build prompt. Northwind
can now reach all four of its sources. Stage 1 is complete.

## Source material

`plugin-catalog/docs/build-a-plugin.md` (the structure this chapter follows),
`docs/run-a-plugin.md` § The rule, `docs/dependent-fields.md`, `docs/publishing.md`,
`CONTRIBUTING.md` § The bar · `go-plugin-sdk/README.md`, `sdkv1/models.go`
(`Action`, `OutboundPort`, `Meta`), `formkit/`, `skills/inflow-plugin/SKILL.md` ·
[Part III — Plugin SDK specification](../03-plugins/plugin-sdk-spec.md),
[Forms: a node with its own UI](../03-plugins/forms-and-ui.md),
[The SDK matrix](../03-plugins/sdk-matrix.md).

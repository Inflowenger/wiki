# Plugin SDK specification

> **Status: outlined.**

## Purpose

`inflowv1` is a plain NATS message protocol, so **an SDK is a convenience, not a
requirement.** This chapter is the specification a new SDK — in Rust, Java, C#, anything —
must satisfy, and the shape the three existing SDKs share.

## The lifecycle every SDK implements

1. **Construct.** Load credentials, open the NATS connection.
2. **Declare identity.** Name, author, version — what the platform shows for this plugin.
3. **Declare requirements (optional).** A settings/onboarding form plus a submit handler.
4. **Declare actions.** One or more methods, each with a form and a request handler.
5. **Start.** Subscribe to every subject: intro, settings, the action list, each action's
   form, each action's executor. Return immediately.
6. **Block and serve.** The process stays alive; from here it is request-driven.

```go
p, _ := sdkv1.NewPlugin(sdkv1.WithDotEnv(".env.inflow"))
p.Intro(sdkv1.PluginIntro{Name: "HTTP.CALL", Author: "inflow Dev. Team", Version: "v0.0.1"})
p.AddAction(sdkv1.Action{Method: "http.call", RequestHandler: handler})
p.Start()
select {}   // Start() only wires subscriptions; the process must stay alive
```

## Sections planned

**1. The conformance checklist.** Exactly what must be implemented: credential decoding
(base64 `.creds`, account read from the JWT), the five subscription families, the
two-phase job handshake, the six job commands, the `{_registry, body}` envelope, the
`Response` shape, request/reply retry on `ErrNoResponders`.

**2. Typed input.** `CastRequestTo[T]` and its equivalents — how each SDK gives a typed
`RequestBody[T]` with `Body` and `Registry`.

**3. Declaring an action.** `Method`, `Title`, `Form`, `RequestHandler`, and how the action
list and per-action forms are served. Plus two optional declarations that make an action
self-describing: **`Outbound`** — statically declared outbound ports, the author-time
counterpart of runtime tag routing, served on `@actions` so a host renders one output port per
entry; and **`Tags`** — an open bag of labels with a reserved `class` key, so one binary can
host several logical products and tell them apart. Neither is required; both keep port
topology and documentation on the action instead of wiring it by hand. →
[Tag routing](../02-fusion/tag-routing.md#declared-outbound-ports--author-time-tag-routing)

**4. Settings profiles.** `RequiredParams` / settings forms, the submit handler, and how
filled values arrive as `body.settings` on every call. The rule that a plugin stores no
user credentials.

**5. Meta functions.** Server-side functions a form may call while open — the mechanism
behind dependent fields. → [Forms](forms-and-ui.md)

**6. Errors and outcomes.** `Done`, `DoneWithError`, `DoneWithErrorData`, what an error means
for the flow (reported and committed; the flow continues) and why `DoneWithErrorData` exists —
to keep a payload and the node's scope alive through a failure. Why `CmdStopFlow` was *removed*:
flow control belongs to the graph and to the user, not to a plugin. A plugin reports outcomes;
it does not decide routing — except through `next_tags`, which selects among ports the author
drew. That includes the reserved `_exception` tag, which lets a node fail *and* route — see
[the exception port](../02-fusion/tag-routing.md#the-exception-port--fail-and-still-route).

**7. Testing without a live platform.** What can be exercised offline.

**8. Packaging and the standard run command.** The catalog rule: every listed plugin starts
with its language's standard command from the repo root
(`go run .` / `npm install && npm run build && npm start` / venv + `python main.py`). This
is what lets one one-liner install any plugin.

## Source material

`inflow-plugin-sdk/README.md`, `cookbook.md`, `docs/architecture.md`, `docs/examples.md`,
`plugin-catalog/docs/build-a-plugin.md`, `plugin-catalog/docs/sdks.md`,
`plugin-catalog/docs/run-a-plugin.md`.

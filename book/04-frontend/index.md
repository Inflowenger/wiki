# Part IV — The Frontend Layer

> **Two npm packages, and nothing else.**
>
> Part II is Go. Part III is a wire protocol. This part is the browser — and it is
> deliberately the smallest part of the book, because the frontend's share of the platform
> is two libraries and a pair of HTTP calls.

Everything so far has been about the parts of Inflowenger that run on a server. A backend
that imports [`inflow-fusion`](../02-fusion/) and answers three questions. A plugin process
that speaks [`inflowv1`](../03-plugins/inflowv1-protocol.md). Both are contracts, and both
are covered.

Something still has to be true for any of it to become a product: **a person has to be able
to draw a process, configure it, run it, and watch it happen.** That is the frontend's job,
and it is the part most teams assume they will be writing from scratch.

They will not. Two of the four jobs are already libraries.

---

## The four jobs, and who does them

| The frontend must… | Who does it |
| --- | --- |
| **1. Author a graph** — a canvas, a form wizard, a YAML editor, an API client | **You.** This is your product, and it is the one thing you should not outsource. |
| **2. Configure a node** — draw a form for an integration you have never seen | **[`@inflowenger/plugin-form-builder`](plugin-form-builder.md)** renders the form the plugin itself declares. |
| **3. Launch a run** | **You** — one `POST` to your own backend, which calls `inflow.NewProcess(...).Exec()`. |
| **4. Watch the run** — movement on the canvas, a log, a status | **[`@inflowenger/flow-trace`](flow-trace.md)** turns the engine's event stream into node and edge state. |

Both libraries live in one repository, [`inflow-js`](https://github.com/Inflowenger/inflow-js),
and they are **independent**. Take one, take both, take neither.

```sh
pnpm add @inflowenger/flow-trace @inflowenger/plugin-form-builder
```

Public packages on npm — no token, no `.npmrc`, nothing to configure.

| Package | What it does | Depends on |
| --- | --- | --- |
| `@inflowenger/flow-trace` | Turns the process event stream into flow movement and completion: where the process is, which edges it took, whether it finished. | **nothing** |
| `@inflowenger/plugin-form-builder` | Renders Inflow's `x-inflow-ui` / `x-inflow-notif` schema extensions as real form widgets, on top of JSON Forms. | Vue 3, JSON Forms |

> **`flow-trace` has no dependencies and no framework.** It runs in a browser, in Node, or
> in a test. If your product is React, Svelte, Angular, a CLI, or has no UI at all, it is
> still yours to use. `plugin-form-builder` is the Vue-only one — see
> [Building on any frontend](build-a-process-product.md#what-each-frontend-gets) for what
> that costs a React app in practice, which is less than it sounds.

---

## What the frontend never does

Worth stating early, because it is the question every architect asks first.

**The browser does not talk to NATS.** It does not hold Infra credentials, it does not know
the engine's address, and it does not subscribe to a subject. The frontend talks to **your
backend**, over ordinary HTTP and an ordinary socket, exactly as it would in any web app
you have built before.

```
        browser                      your backend                   the platform
 ┌──────────────────────┐   ┌───────────────────────────┐   ┌─────────────────────┐
 │                      │   │                           │   │                     │
 │  your authoring UI   │   │                           │   │                     │
 │                      │   │                           │   │                     │
 │  plugin-form-builder ├──►│  POST /plugin/:id/fn ─────┼──►│  plugin process     │
 │                      │   │              (inflowv1)   │   │                     │
 │  "Run"               ├──►│  POST /process            │   │                     │
 │                      │   │    NewProcess().Exec() ───┼──►│  Fractal (engine)   │
 │                      │   │                           │   │         │           │
 │  flow-trace          │◄──┤  WebSocket ◄── relay ◄────┼───┼─────────┘  events   │
 │                      │   │                           │   │                     │
 └──────────────────────┘   └───────────────────────────┘   └─────────────────────┘
                                   inflow-fusion              Infra: NATS, creds,
                                                              spaces, engine registry
```

Your backend is the only thing holding credentials, and that is not an accident of this
design — it is the [backend contract](../02-fusion/the-backend-contract.md). The two
frontend packages consume what your backend forwards: **the event stream**, and **a
plugin's form**.

---

## The claim this part has to earn

The claim is the same one the rest of the book makes, restated for the browser:

> **FloMorphic's frontend is a Vue app that imports these two packages and writes its own
> business logic. Nothing was reserved for it.** Every hook `flomorphic-wapp` uses is a
> published export, at the version on npm.

[How it is wired](build-a-process-product.md) shows that wiring in full — the store that
owns the tracker, the component that wraps the form, the transport function that is the one
thing a host must supply — with pointers into FloMorphic's own source so you can check it
rather than take it.

---

## Chapters

| Chapter | What it establishes |
| --- | --- |
| [Watching a run: `flow-trace`](flow-trace.md) | The tracker, the events it emits, the state it accumulates, and the five rules a consumer gets wrong |
| [The log taxonomy](log-categories.md) | Every kind, level, source and **category** on the wire — the complete set, and what to draw for each |
| [Dynamic forms: `plugin-form-builder`](plugin-form-builder.md) | How a node you have never seen gets a working configuration UI, generated from the plugin's own schema |
| [What a form can say](notifications.md) | `x-inflow-notif` — help, verification results, failures, and the claim protocol |
| [Building a process product on any frontend](build-a-process-product.md) | The chapter this part exists for: giving your users the ability to define their own processes |

---

## Next

- **[Watching a run](flow-trace.md)** — start with the package that works everywhere.
- Or skip to **[Building a process product](build-a-process-product.md)** if you are here to
  decide whether this is worth it before reading the API surface.

**Source material:** [`inflow-js`](https://github.com/Inflowenger/inflow-js) —
`README.md`, `packages/flow-trace`, `packages/plugin-form-builder`, `lab/`;
`flomorphic-wapp/src`.

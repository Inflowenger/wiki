# The `inflowv1` protocol

> **Status: outlined.** The normative reference is
> `inflow-plugin-sdk/docs/protocol-inflowv1.md`; this chapter will be the book's
> self-contained rendering of it.

`inflowv1` is the contract between the runtime (Fractal) and a plugin, carried over NATS
request/reply. Everything is namespaced by the plugin's `PLUGIN_ID`.

## The one idea that makes the subject map readable

Two markers classify every subject, and they carry the meaning:

- **`@`-prefixed segment ⇒ the UI / arguments plane.** `@intro`, `@settings`, `@actions`,
  `@form` are about how the node *presents and configures itself*. Pure metadata. Nothing
  runs.
- **`cpu` ⇒ the execution plane.** `inflow.cpu.<PLUGIN_ID>.*` is the node's **main call**,
  requested by the Fractal at run time, plus the job commands that call back while it runs.

> Read a subject by its markers: a `@` part means *describe/configure me*; `cpu` means
> *run me*.

## Sections planned

**1. The metadata plane — `inflow.v1.*`.** Full table: `@intro`, `@settings`, `@actions`,
`<ACTION>.@form`, and meta functions. What each returns — including how `@actions` carries an
action's **declared outbound ports** (`Action.Outbound`), so a host renders one output port per
branch and stamps each edge with its tags before anything runs.

**2. The execution plane — `inflow.cpu.*`.** `inflow.cpu.<PLUGIN_ID>.<ACTION>` to execute;
`inflow.cpu.<PLUGIN_ID>.<JOB_ID>.<CMD>` for a running job's commands back to the runtime.

**3. The request → job handshake.** Why execution is two-phase: the plugin mints a `jobId`
and replies immediately, then works asynchronously. The immediate ack is how the runtime
correlates every later job command to this execution. Include the full sequence diagram.

**4. The request payload.** The `{_registry, body}` envelope. `body` is the user's form
input; `_registry` is runtime metadata about this node's **previous** run (notably the prior
`jobId` and `doneAt`) — the basis for idempotency, dedup and resume.

**5. Response shapes.** Metadata lookups reply with the marshalled declaration; meta
functions and settings submit reply with `Response{Data, Error}`; job commands reply with
raw command output.

**6. Transport details.** Request/reply with retry — default 5s timeout (`REQ_TIMEOUT`
overrides at deploy time), up to 5 retries on `ErrNoResponders` with backoff, so a job
command issued a moment before the runtime is listening still lands.

**7. Credentials.** A decorated NATS `.creds` blob (JWT + NKey seed), supplied
**base64-encoded** in `INFRA_CRED`. The SDK decodes it, reads the account from the JWT, and
connects with auto-reconnect. The three environment variables a plugin needs:
`PLUGIN_ID`, `INFRA_CRED`, `INFRA_URL`.

**8. Protocol versioning.** The `v1` in both the subject prefix and the package name. A
future revision introduces `inflow.v2.*` and `sdkv2` *alongside* this one, so plugins and
runtimes migrate independently.

**9. Open questions.** Horizontal scale for multiple instances sharing one `PLUGIN_ID`
(NATS queue groups?) — recorded honestly rather than papered over.

## Source material

`inflow-plugin-sdk/docs/protocol-inflowv1.md` (normative),
`inflow-plugin-sdk/docs/architecture.md`.

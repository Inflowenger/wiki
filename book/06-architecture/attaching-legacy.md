# Attaching a system you already run

> **Status: outlined.**
>
> The chapter most likely to decide whether the platform is adoptable in a real
> organisation.

## The claim

**You do not have to move anything.** Most tooling asks you to bring your data and your
logic to it. The graph goes the other way: it reaches into what you already have, over
subjects your own services own.

A system in production for a decade can become AI-native without a rewrite.

## The three doors

| Door | Mechanism | When it fits |
| --- | --- | --- |
| **Extrinsic** | Your Go backend imports `inflow-fusion` and registers handlers with `svcHandler.ImplHandlerOnSubject`. A node publishes to that subject; **your handler's reply becomes the node's output.** | Your backend is Go, or you can put a thin Go service in front of it. The cheapest door — a handful of lines, no new process. |
| **Plugin** | A standalone process speaking `inflowv1` in Go, Node or Python. Appears on the canvas as a node with its own configuration form. | Anything else: a REST API, a database, a message bus, a mainframe gateway, a vendor system. |
| **MCP** | The MCP node, as a client. | The system already speaks MCP, or you can put an MCP server in front of it. |

You can use all three at once.

## Sections planned

**1. The order of operations.** It matters, and it is the opposite of a migration plan:
   1. **Nothing moves.** The database stays. The services stay. The deployment stays.
   2. **Expose what already exists.** A few subject registrations turn existing business
      operations into nodes, without changing what those operations do.
   3. **Draw the new behaviour above them.** Retrieval, model calls, policy checks,
      approvals and scheduling get composed *on top* of capabilities already proven in
      production.
   4. **The domain expert takes the pen.** Because behaviour is a graph, changing it stops
      requiring a release.

> The legacy system does not become AI-native by being rewritten. It becomes AI-native by
> being **reachable**.

**2. One registration, many capabilities.** `svc.add.issue.{TABLE_NAME}` subscribes as a
wildcard; the handler recovers the parameter from `recv_subject`. One handler, one door,
many palette cards.

**3. The thin-Go-service pattern.** For estates that are not Go: what the shim does, how
small it can be, and when a plugin is the better answer instead.

**4. Security posture when reaching inward.** Origin tagging on plugin-initiated calls;
what a handler should verify; why "the graph can call it" is an authorisation decision, not
a wiring decision.

**5. A staged adoption plan.** One flow, one team, one process — and the honest
prerequisites for it to succeed.

**6. When *not* to do this.** Logic that is genuinely better as code; processes with no
operator who wants the pen; systems where the constraint is organisational rather than
technical. Naming these protects the cases where it does work.

## Source material

`FloMorphic/getting-started/docs/architecture.md` (Meeting the system you already run),
`inflow-fusion/docs/nodes/extrinsic.md`, `inflow-fusion/docs/plugin-svc-calls.md`.

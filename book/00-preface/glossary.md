# Glossary

> **Status: outlined.** Terms and one-line definitions are committed; full entries pending.

| Term | One line |
| --- | --- |
| **Inflowenger** | The platform: a runtime for context processing. Infra + Fractal, plus the SDKs around them. |
| **Infra** | The control plane. Embedded NATS, accounts, credentials, engine registry. Closed source; contract documented. |
| **Fractal** | The execution engine. Walks a compiled node map. Closed source; contract documented. Several may run at once. |
| **Context** | The memory. A durable JSON document, JSONPath-addressable, that a run iterates over. |
| **Context document** | `{Data, Header}` — `Data` is yours; `Header` is engine-managed memory surviving across runs. |
| **Flow** | A compiled graph: a flat map of primitive nodes. What the engine executes. |
| **Process / run** | One execution of a flow, identified by a `pid`. |
| **Node** | One step. Always one of six primitives. |
| **Primitive** | Void, Code, Contract, Extrinsic, Plugin, GoTo. The only things the engine can execute. |
| **`Next`** | An outgoing transition, carrying the tags that select it. |
| **`Depends`** | Inbound nodes that must all finish first. The join mechanism. |
| **Tag routing** | Emit a tag list; the engine follows only transitions whose tags match. |
| **Scope** | The JSONPath slice of context a node reads and writes under. |
| **Key** | Where a node's output is written into the context. |
| **Compiler** | Turns *your* authoring format into the engine's node map. |
| **Hook** | The per-node function inside a compiler. The only place your vocabulary lives. |
| **`inflow-fusion`** | The Go SDK binding your backend into the platform. |
| **`IInflowService`** | The three-method interface your backend implements. |
| **Extrinsic service** | Domain logic you expose on a NATS subject, callable from a node. |
| **Plugin** | A live external process speaking `inflowv1`. The one primitive that never compiles away. |
| **`inflowv1`** | The NATS protocol between the runtime and a plugin. |
| **Job** | One plugin execution, identified by a `jobId`, able to report progress and read/write context. |
| **Space** | A NATS account — the unit of authentication, authorisation and isolation. |
| **Portal / resource** | A registered engine instance in Infra's registry. |
| **FloMorphic** | The AI harness built end to end on the runtime. The runtime's first product. |
| **Venapce** | A security-governance product whose business logic lives in FloMorphic workflows. |
| **`x-inflow-ui`** | The JSON Schema extension letting a plugin's form call back into the plugin. |
| **Process event stream** | The `v:1` event contract an engine publishes while executing. |

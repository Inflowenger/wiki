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
| **Process / run** | One execution of a flow, identified by a `pid`. A fractal instance forgets a `pid` the moment its run ends. |
| **`instanceId`** | Correlation id shared by every run of one *logical* workflow instance. A park→resume chain is several runs, several pids, one `instanceId`. Your convention, not the engine's. |
| **Process row** | The backend's own record of a run — status, request, snapshot, error ledger. The engine keeps no run history, so this is the only durable account of what happened. |
| **Park** | A node's handler answers `{"_cmd":"stop"}`; the branch ends and the process finishes normally. Nothing stays open, which is why a three-day wait is free. |
| **`_sched`** | Context-header slot holding a run's completed node generations and join watermarks (`ResumeState`). Seeded into a continuation so a join past the resume point does not lock; gated on a structural flow signature. |
| **`_errors`** | Context-header slot holding one run's error ledger (`RunErrors`): a true `count`, capped `items`, each tagged `kind: node` (the flow author's) or `kind: system` (the platform's). A run that hit errors can still finish `completed`. |
| **Node** | One step. Always one of six primitives. |
| **Primitive** | Void, Code, Contract, Extrinsic, Plugin, GoTo. The only things the engine can execute. |
| **`Next`** | An outgoing transition, carrying the tags that select it. |
| **`Depends`** | Inbound nodes that must all finish first. The join mechanism. |
| **Tag routing** | Emit a tag list; the engine follows only transitions whose tags match. |
| **Outbound port** | A statically declared output branch on a plugin action (`Action.Outbound`), served on `@actions` so the host renders one port per branch and stamps its tags before anything runs. |
| **Exception port** | The reserved `_exception` tag: a node routes it *and then* fails, so the branch continues to a handler the author drew while the node is recorded as an error. |
| **Decider** | A node that picks among a closed set of declared answers (e.g. Jev), as opposed to a *reasoner* (e.g. LLM) that generates open text. Both route through tag routing. |
| **Scope** | The JSONPath slice of context a node reads and writes under. |
| **Key** | Where a node's output is written into the context. |
| **Settings profile** | Named configuration (provider credentials, base URLs) bound to a node kind or plugin, referenced by id from the graph — so secrets never live on a canvas node. |
| **Compiler** | Turns *your* authoring format into the engine's node map. |
| **Hook** | The per-node function inside a compiler. The only place your vocabulary lives. |
| **`inflow-fusion`** | The Go SDK binding your backend into the platform. |
| **`IInflowService`** | The three-method interface your backend implements. |
| **Extrinsic service** | Domain logic you expose on a NATS subject, callable from a node. |
| **Plugin** | A live external process speaking `inflowv1`. The one primitive that never compiles away. |
| **`inflowv1`** | The NATS protocol between the runtime and a plugin. |
| **Job** | One plugin execution, identified by a `jobId`, able to report progress and read/write context. |
| **`DoneWithErrorData`** | A job outcome that ends failed but still commits a payload — keeping the node's scope and partial results alive through the failure, and the companion of the exception port. |
| **Space** | A NATS account — the unit of authentication, authorisation and isolation. |
| **Portal / resource** | A registered engine instance in Infra's registry. |
| **FloMorphic** | The AI harness built end to end on the runtime. The runtime's first product. |
| **Venapce** | A security-governance product whose business logic lives in FloMorphic workflows. Tier 3 for its logic, and tier 2 for its reach — it ships an `inflowv1` plugin in-process. |
| **Operation** | *(Venapce)* A feature shipped as an installable package: a manifest plus workflow exports, landed into FloMorphic with the operator's params substituted in. |
| **Activity** | *(Venapce)* A row's history entry. A `run` activity is one flow executed on one pipeline row, with the flow's conclusion lifted back into typed columns. |
| **`x-inflow-ui`** | The JSON Schema extension letting a plugin's form call back into the plugin. |
| **Process event stream** | The `v:1` event contract an engine publishes while executing. |

# Jobs and commands

> **Status: outlined.**

The execution register — where a plugin's power actually lives. Metadata lookups are
inert; a **Job** is where progress, context injection and routing happen.

## The six job commands

Every one is a NATS request on `inflow.cpu.<PLUGIN_ID>.<JOB_ID>.<CMD>`:

| `<CMD>` | Sent by | Meaning |
| --- | --- | --- |
| `progress` | `job.Progress` / `job.Done` / `job.DoneWithError` | report progress `0–100` (100 = finished) |
| `context/current` | `job.CmdGetCurrentScope` | read the current context scope |
| `context/path` | `job.CmdGetScope` | read context by JSON path; `$this` is rewritten to the node's location |
| `commit` | `job.CmdSetOnPath` | write data into context at a JSON path (`commit_on`, `$this` allowed) |
| `next_tags` | `job.CmdNextFilter` | route outbound ports: keep only the named tags |
| `request/svc.<ACTION>` | `job.CmdSvcCall` | call a backend service through the runtime |

## Sections planned

**1. Progress, and what the canvas does with it.** `job.Progress(20, Frame{Title, Content})`
— the `Frame` is what streams into the node on the canvas. Designing useful progress for a
long job.

**2. Reading context.** Current scope vs. an explicit JSON path. The `$this` rewrite and why
it matters inside an iteration.

**3. Writing context.** `commit`, `commit_on`, and the open question of merge-vs-replace
semantics at a path that does not yet exist. Flagged honestly as *to verify*.

**4. Routing from inside a job.** `CmdNextFilter` — the mechanism that lets a model node
choose among the ports the author drew. Cross-reference
[Tag routing](../02-fusion/tag-routing.md).

**5. Calling back into the backend — `CmdSvcCall`.** The subtle one. The action rides in the
subject (`request/svc.log`, `request/svc.add.db.record`); the runtime cuts the prefix and
re-issues the request to the bare action **on the plugin space**. The payload is a
`{data, op}` envelope, forwarded with an `origin: plugin:<node title>` header — so a backend
can refuse ungranted plugin-originated calls. Note these do **not** arrive on the infra
connection; a backend subscribes on the plugin space deliberately.

**6. Finishing.** `Done` vs `DoneWithError`. An error is reported and committed, and the
flow continues — because routing belongs to the graph.

**7. Long-running and reconnecting jobs.** How `_registry` carries the previous `jobId`, and
what it takes for a job that outlived its process to be reconnected.

**8. Idempotency patterns.** Using `_registry.jobId` and `doneAt` for dedup and resume.

## Source material

`inflow-plugin-sdk/docs/jobs-and-commands.md` (normative),
`inflow-plugin-sdk/docs/protocol-inflowv1.md`, `inflow-fusion/docs/plugin-svc-calls.md`.

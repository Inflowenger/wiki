# 6 · Stage 3 — Decision

> **Sixty minutes.** A real request arrives. The brain has to answer it against a real
> contract, decide whether a human must sign, respond, and leave behind something you could
> defend in a dispute. This is the stage the previous two existed for.

## The request

> *Arcadia Mills, customer since 2019, reports that press line 3 is down. They want an
> emergency engineer on site at 02:00 Saturday. Are they entitled to it, what will it cost
> them, and who has to approve it?*

A sentence a duty manager answers in twenty minutes with four browser tabs. Decompose it and
there are four distinct questions, and **they want four different mechanisms** — which is the
whole lesson of this chapter:

| The question | What answers it | Why not a model |
| --- | --- | --- |
| *Who is this, and what did we sell them?* | **Retrieval** — contract row, prior incidents, asset graph | It is a lookup. A model would guess |
| *Is this covered?* | **A decider** — closed set, calibrated, fast | It is a choice among known answers, not an essay |
| *Does policy allow it?* | **A policy engine** — Rego, outside the model | It must be auditable and it must not be persuadable |
| *Who signs?* | **A human** — the run parks | Because it is their call, not the system's |
| *What do we tell them?* | **A model** — grounded on the above | This is the one genuinely generative step |

Most systems that get this wrong use a model for all five. The architecture's position is
that only the last one is a model's job.

## The flow

```
POST /hooks/northwind-request
        │
   ┌────▼──────────────┐
   │ normalize request │ js · $
   └────┬──────────────┘
        ├──────────────────────┬────────────────────────┐   parallel: no mutual dependency
   ┌────▼────────┐   ┌─────────▼────────┐   ┌───────────▼────────────┐
   │ contract    │   │ prior incidents  │   │ asset dependency graph │
   │ POSTGRES    │   │ vecstore search  │   │ ARANGODB plugin        │
   └────┬────────┘   └─────────┬────────┘   └───────────┬────────────┘
        └──────────────────────┴────────────────────────┘
                     ┌─────────▼──────────┐
                     │   Wait for All     │  promissall — merges ONCE
                     └─────────┬──────────┘
                     ┌─────────▼──────────┐
                     │  Jev · entitlement │  decider · ports per answer
                     └──┬────────┬────────┴──────┐
         covered ───────┘        │ needs_approval │ _exception
          ┌─────────────┐  ┌─────▼────────────┐  └──▶ exception path
          │             │  │ OPA policy check │  rule · lang opa
          │             │  └──┬────────┬──────┘
          │             │     │approve │escalate
          │       ┌─────▼─────▼──┐  ┌──▼─────────────────┐
          └──────▶│ compose reply│  │ Human in the Loop  │ parks the run
                  │ llm          │  └──┬─────────────────┘
                  └─────┬────────┘     │ resumes on answer
                        ├──────────────┘
                  ┌─────▼────────┐
                  │ POST to portal│ http
                  └─────┬─────────┘
                  ┌─────▼──────────────┐
                  │ write decision row │ docstore · THE AUDIT TRAIL
                  └────────────────────┘
```

## Build it

```text
Use the flomorphic MCP server.

1. flo_get_design_guide first. Follow it exactly.
2. Build a workflow "answer-request".

   a. Start → a JS node at scope "$", key "req", normalizing the delivered webhook
      payload into { customerId, assetId, requestedAt, summary, kind }.

   b. THREE parallel branches off that node — they do not read each other:
      - The POSTGRES plugin action postgres.query against the LIVE CRM:
          SELECT * FROM contracts WHERE customer_id = '{{$.req.customerId}}'
            AND status = 'active'
        → key "contract", scope "$". (Settings profile northwind-crm.)
      - Vector Store READ (search) on org_knowledge for prior incidents, query text
        built from {{$.req.summary}} → key "priors", scope "$".
      - ARANGODB plugin action arango.graph.traverse from the asset
        {{$.req.assetId}} on graph "assets", direction outbound, maxDepth 2
        → key "impact", scope "$".

   c. All three into a Wait for All.

   d. Then a Jev node at scope "$", key "verdict", with the state template built from
      {{$.contract}}, {{$.req}} and {{$.impact}}, and TWO questions:
      - id "entitlement", type choice, min_confidence 0.75, options:
          covered, needs_approval, out_of_scope, other
      - id "urgency", type score, route false, options: low, normal, high
   e. Wire the entitlement ports:
      - entitlement.covered        → the compose node (h)
      - entitlement.needs_approval → the policy node (f)
      - entitlement.out_of_scope   → the compose node (h)
      - _exception                 → the human node (g)

   f. A Rule node at scope "$", lang opa, key "policy", with handlers "approve"
      and "escalate" — approve only when the contract tier allows out-of-hours work
      AND the estimated cost is under the contract's ceiling.

   g. A Human in the Loop node, mode park, channel direct, whose prompt explains the
      request, the contract terms found, and Jev's scores, and asks the duty manager
      what to do.

   h. An LLM node at scope "$", key "reply", composing the customer-facing answer
      GROUNDED ONLY on $.contract, $.priors and $.policy — with an explicit
      instruction to say "I don't have that" rather than infer.

   i. An HTTP node POSTing $.reply back to the staff portal.

   j. A Doc Store WRITE on decisions_store with the audit row (see below).

   Ids you need and I have: <paste the store ids, pluginId, settings profile ids>

3. flo_plan_patch, NOT apply. Show me `problems`.
```

### Read the `problems` list before you apply

This flow is exactly the shape that trips the three checks the planner runs:

| Problem it may report | What it means here |
| --- | --- |
| *a routing node on a many-valued scope* | The Jev or Rule node got a wildcard scope. Both must be `$` — they route for the whole node |
| *branches converging without a join* | Something after the Jev node has two inbound edges and will run twice. The `compose` node is the likely culprit — see below |
| *a no-op wait for all* | A `promissall` with one inbound branch, which waits for nothing |

## The three mechanisms, and why each one

### Why the contract is read live, and the vector store is not

Chapter 5 indexed contracts into `org_knowledge`, so it is tempting to answer *"what did we
sell them?"* with a vector search. Don't. The two reads are doing different jobs:

| | Live `postgres.query` | `org_knowledge` search |
| --- | --- | --- |
| Answers | *What are the terms, exactly, right now?* | *What is relevant to this situation?* |
| Guarantees | The authoritative row, at this instant | Nearest neighbours, as of the last sweep |
| Used for | The entitlement decision and the audit row's `contractId` | Prior incidents, context, precedent |

A decision you have to defend cites the authoritative row, not a shadow copy that is up to
24 hours stale. The vector store is for *finding* things; the source of record is for
*applying* them. This is also the clearest instance of the platform's central claim in
operation: the decade-old CRM was not migrated, replaced or wrapped — the graph just calls
it.

### Why Jev and not an LLM here

Both the LLM node and the Jev node bind their result to ports the author drew. The
difference is what gets bound, and it decides which one you reach for:

| | **LLM** | **Jev** |
| --- | --- | --- |
| Decides with | generated text | a calibrated template over typed questions |
| Bound to ports | the model's callable **functions** | the template's declared **answers** (`<question>.<option>`) |
| Can it refuse | yes — hallucinate, waffle, call nothing | **no** — no confident match answers `other`, which is a port you drew |
| Cost | one model turn, streamed | one 70–500 ms round-trip, no free text |

"Is this covered: yes / needs approval / out of scope" is a **closed set of four answers**.
Paying a model turn to choose among four known options is the symptom of a decider that got
implemented as a reasoner. Move it to a decider and it becomes one edge on a graph you can
inspect.

A Jev question is `{id, type, instructions, options: [{name, description}], min_confidence}`
where `type` is `choice`, `score` or `noul`. Its ports are `<question id>.<option name>` —
`entitlement.covered`, `entitlement.needs_approval`. Set `route: false` on a question you
want *scored but not routed*, which is what `urgency` is here: it lands in the context for
the reply and the audit row without adding four more edges to the canvas.

### `min_confidence` is policy, not a prompt instruction

`min_confidence: 0.75` is not "be careful". It is a **threshold the node enforces**. Below
it, the node routes the reserved `_exception` port and reports failure *with its scores
attached*, rather than quietly returning its best bad guess.

The threshold is a field on the node. The fallback is an edge you drew. That is the whole
difference between a guardrail and a hope — and in this flow `_exception` goes to the human,
which is the correct answer to "the system is not confident about a contract question".

### Why the policy is Rego, outside the model

Northwind's out-of-hours rule is a real contractual term with a number in it. Three
properties you want from the thing that evaluates it, none of which a model has:

- **Deterministic.** The same inputs give the same answer, today and in the dispute.
- **Unpersuadable.** No phrasing of the request changes the cost ceiling.
- **Readable by a non-engineer.** The duty manager's manager can read the Rego.

In an `opa` Rule node, `input` is the scoped slice and `data` holds the Conditions
key/values; the node outputs the variable named by `opa_result`. In a `js` Rule node the
`logic_rule` must return **one string equal to a handler's name** — that string is the port
that fires. Return an array to fire several at once.

The failure mode to know: **a Rule that fires no tag prunes every outgoing edge** and the
branch ends silently, with no error and no log line. So make the returned value exhaustive
over the declared handlers.

### The human is a node, and the hard part is resuming

The Human in the Loop node lowers to an `Extrinsic` call to `svc.hitl.add`. It poses the
situation, records a **Human Task**, and the run **parks**. When the answers arrive the run
resumes from its captured next nodes.

| Field | Values | Note |
| --- | --- | --- |
| `mode` | `park` · `continue` | `park` stops the run here. `continue` records the task and carries on — a notification, not a gate |
| `channel` | `direct` (in-app) · `telegram` · `whatsapp` | `direct` for this session |
| `prompt` | text, with `{{$.path}}` tokens | Shipped as authored; the runtime resolves every JSONPath in the payload before delivering it, so the person sees finished text |

> **A HITL node carries no question list.** What has to be asked is worked out in the
> session, not on the canvas — which is why the `prompt` is written as *"explain the point
> the flow could not settle, and ask what you need"*, not as a form. The questions are
> whatever the session produces, and `flo_answer_human_task` answers them one at a time.

Pausing is easy. Resuming is the hard part, and it is the runtime's problem rather than
yours — the captured traversal, the park shape and the reconciliation are walked in
[The wire](../02-fusion/the-wire.md).

### Grounding the one generative step

The LLM node is last for a reason: by the time it runs, every fact it needs is already in
the context, written there by nodes that cannot make things up. Its prompt should reference
`{{$.contract}}`, `{{$.priors}}` and `{{$.policy}}` explicitly and instruct it to say *"I
don't have that"* rather than infer.

That is all RAG is here, and it is deliberately anticlimactic: **a vector-store search and
a model call.** Two nodes. No framework.

## The audit row is the deliverable

If you take one thing from this chapter into your own build, take this. The flow's last
node writes a row to `decisions_store`, and what goes in it is the difference between a
system you can defend and a system you cannot:

| Field | Why it has to be there |
| --- | --- |
| `requestId`, `receivedAt` | Identity and ordering |
| `inputs` | The normalized request, verbatim as decided on — not as later edited |
| `contractId`, `contractVersion` | *Which* terms were applied |
| `retrievedIds` | The vector ids and document ids the answer was grounded on. Without this, "the AI said so" is the whole explanation |
| `verdict`, `scores` | Jev's answer **and its confidence per question** |
| `policyResult`, `policyVersion` | What the Rego returned, and which Rego |
| `humanTaskId`, `answeredBy`, `answeredAt` | Who signed, if anyone |
| `reply` | What was actually sent to the customer |

Create the store the same way as chapter 5's:

```text
flo_create_document_store
  name: "decisions_store", table: "decisions"
  columns: [{name:"request_id",type:"TEXT",primary:true}, {name:"received_at",type:"TEXT"},
            {name:"inputs",type:"TEXT"}, {name:"contract_id",type:"TEXT"},
            {name:"retrieved_ids",type:"TEXT"}, {name:"verdict",type:"TEXT"},
            {name:"scores",type:"TEXT"}, {name:"policy_result",type:"TEXT"},
            {name:"human_task_id",type:"TEXT"}, {name:"reply",type:"TEXT"}]
```

> **Observability is a precondition, not a feature.** Everything above is visible *while the
> run happens* — progress frames stream to the canvas, the context document is readable at
> every step, the Processes list holds status, error and timing. The audit row exists because
> a run's live trace is not a record you can query in six months. Both, not either.

## Run it as a test

Do not fire the webhook. Craft the context:

```text
flo_upsert_context with exactly:

{
  "payload": {
    "customerId": "arcadia-mills",
    "assetId": "assets/press-line-3",
    "requestedAt": "2026-10-10T02:00:00Z",
    "kind": "emergency_restore",
    "summary": "Press line 3 down, no output. Requesting engineer on site 02:00 Saturday."
  }
}

Then flo_start_process on answer-request with that contextId.
Poll flo_get_process until it leaves `running`.
Then flo_get_context and show me, in order: req, contract, priors, impact, verdict,
policy, reply.
```

The run will almost certainly end in `waiting`, not `finished` — because it parked at the
human node. That is the system working. Resume it:

```text
flo_list_human_tasks with status open — show me the task and its questions.
flo_answer_human_task for each question.
Then flo_get_process again: the run should have resumed and reached the reply.
```

That round trip — park, answer, resume — is the thing most agent frameworks do not have, and
running it once by hand is worth more than reading about it.

## The three amendments you will actually make

This is where the session's thesis cashes out: **the brain changes by conversation, not by
redeploy.**

### 1 · The cost ceiling changed

```text
Amend workflow answer-request: the out-of-hours cost ceiling moved from 3000 to 5000.

Read it with flo_get_workflow first. Change ONLY the opa Rule node's logic_rule.
Keep every node id. Show me the before and after of the policy text, then
flo_plan_patch, then apply once I confirm.
```

No redeploy, no release, no engineer. The business logic is a workflow graph over living
context, and this is what that sentence means in practice.

### 2 · Jev routes too many requests to `_exception`

The debugging prompt, and note that it reads *evidence* before proposing anything:

```text
Several runs of answer-request ended at the human node via _exception.

- flo_list_processes for this flow, status waiting.
- For three of them, flo_get_context and show me verdict — including the per-option
  scores for the "entitlement" question.
- Tell me which it is:
  (a) the top score is below min_confidence 0.75 → the threshold is too high, or the
      state template is not giving Jev enough to decide on;
  (b) the top answer is "other" → our four options do not cover the real cases;
  (c) an API error → it is not a calibration problem at all.

Name which one before proposing a change.
```

Three different causes, three different fixes, and they are distinguishable **only because
the scores are in the context**. A decider that reported just its answer would leave you
guessing. (a) is a threshold or a template fix; (b) is a fifth option plus the edge to go
with it; (c) is an operational problem.

### 3 · Bronze-tier customers must never auto-approve

```text
Amend answer-request: a customer on contract tier "bronze" must never reach the
compose node directly — every bronze request goes to the duty manager.

Read the flow first. Add a handler "bronze_review" to the opa Rule node, make the
policy return it whenever input.contract.tier == "bronze", and wire that port to
the existing Human in the Loop node. Do not touch any other edge.

Plan, show me the diff in words, apply.
```

Three sentences, one new port, one new edge. The important part is what did *not* change:
the retrieval branches, the decider, the reply, the audit row.

> And every one of these amendments is also a thing a person can do on the canvas, by hand,
> with no assistant involved. The assistant is faster. It is not privileged.

## What this stage costs, stated plainly

- **A decider needs calibration, and calibration needs examples.** `min_confidence: 0.75` is
  a guess until you have run a few hundred real requests and looked at where it lands. Plan
  to tune it.
- **Retrieval quality caps everything downstream.** If chapter 5 indexed badly scoped text,
  this flow produces a confident answer grounded in the wrong paragraph. The audit row's
  `retrievedIds` is how you catch it.
- **The human is a bottleneck by design.** Every `_exception`, every bronze request, every
  escalation lands in one queue. Measure its depth before you widen the net of what escalates.
- **An `out_of_scope` answer is still an answer that goes to a customer.** Decide whether
  that path needs its own human check. For Northwind it does not; for a regulated business
  it would.
- **This flow writes to a staff portal over HTTP.** It is one node and it will be the thing
  that fails most often. It has no retry in this design — a real one wants a Rule on the HTTP
  node's result and a `Continue After` to come back in ten minutes.

## Checkpoint

One flow, run end to end, that parked for a human and resumed. One audit row holding the
inputs, the retrieved ids, the scores, the policy result and the signature. Three amendments
made by conversation.

That is the brain answering a question. Stage 3 is complete.

## Source material

`flomorphic-api/inflow/node_builders.go` (the Jev question wire shape, the HITL payload and
its `mode` / `channel` narrowing) and `inflow/jev_body_test.go` ·
`flomorphic-api/designer/assets/preamble.md` (branching, joining, Rule semantics, opa) ·
`flomorphic-api/mcpserver/tools_hitl.go`, `tools_process.go`, `tools_context.go` ·
`FloMorphic/getting-started/docs/ai-harness.md` and `docs/nodes.md` ·
[Part V — The AI harness](../05-flomorphic/ai-harness.md),
[Tag routing § the exception port](../02-fusion/tag-routing.md),
[Part II — The backend contract § resuming a run](../02-fusion/the-backend-contract.md),
[Observing a run](../02-fusion/observing-a-run.md).

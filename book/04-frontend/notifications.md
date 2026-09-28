# What a form can say — `x-inflow-notif`

> A form has things to say that are not field values: *this is what the token is for*,
> *the site accepted it*, *no assignable user matches "mehdi"*, *re-authorise before this
> runs at 3am*.

Before this key existed, a plugin author's only way to say any of those was to add a
readonly `lookupStatus` string property to the schema and patch text into it — **the form
pretending to have a field it does not have**, and the host with no way to know that this
one is a message and the others are data.

`x-inflow-notif` is that vocabulary, and it is the second reason a third-party form can
feel native in your product rather than merely functional.

---

## Declared per field

```jsonc
{
  "type": "Control",
  "scope": "#/properties/apiToken",
  "x-inflow-notif": [
    { "severity": "help", "message": "Create one at id.atlassian.com → Security → API tokens." },
    { "display": "dialog" }
  ]
}
```

Two kinds of entry, distinguished by whether they carry a `message`:

- An entry **with** a `message` **is** a message — declared help, a standing warning — and
  renders as soon as the field does.
- An entry **without** one declares the **channel**: how messages that arrive at this field
  *later* want to be shown.

---

## The same key raises one at runtime

A meta function returns it beside the rest of its answer, and it is lifted out before the
patch is applied:

```json
{
  "issueKey": "OPS-42",
  "x-inflow-notif": { "severity": "success", "message": "Issue: OPS-42 — Rotate the staging certs" }
}
```

Or, from an action:

```ts
ctx.notify({ severity: 'error', display: 'dialog', message: '401 — the token was rejected.' })
ctx.notify('Connected as mehdi@acme.io')   // a bare string is the common case
```

| Field | |
| --- | --- |
| `message` | The text. **Empty or absent clears the channel** rather than raising a blank one |
| `severity` | `help` · `info` · `success` · `warning` · `error`. Defaults to `info` |
| `display` | `inline` · `banner` · `toast` · `dialog`. A **request** — see below. Defaults to the field's declared channel, else `inline` |
| `field` | Which field it is about. Defaults to the one the action drives (`action.target`, else its own control) |
| `id` | Which channel on that field it occupies |
| `title`, `timeout`, `dismissible` | Optional. `timeout` is ms until it clears itself |

> **Messages replace by `(field, id)`.** Press a lookup twice and the second answer
> supersedes the first instead of stacking under it — the behaviour every plugin author
> hand-rolls with a status property today. Give a field two ids when it genuinely has two
> live messages (a validity line *and* a quota warning).

---

## The host decides where it appears

```ts
onNotify(event) {
  if (event.severity === 'error') { myToasts.danger(event.message); return true }
  if (event.display === 'dialog') { myDialog.alert(event); return true }
  // returning nothing = "seen, not handled"
}
```

**Return `true` to claim a message. That is the whole protocol.** A claimed message is
yours to show — as a toast, a modal, a desktop notification, a log line, whatever this
platform has — and the package renders nothing for it. Return anything else and you have
merely *observed* it: the message still appears inline at its field.

That asymmetry is deliberate. `display` is a **request, not an instruction**, because only
the host knows what its platform can do — and a message that no host got round to wiring
**must not vanish**.

> **Nothing is ever lost for want of a toast.** Implementing `onNotify` for errors only, or
> not at all, is a safe place to stop. A listener that *throws* is treated as one that did
> not claim.

Configure it app-wide in `createInflowUi`, per form with `:on-notify`, or observe the
traffic without taking responsibility for it via `@notification` — an event handler cannot
return a value, so it cannot claim.

### A worked policy

FloMorphic claims **only what asked to interrupt** (`flomorphic-wapp/src/main.ts`):

```ts
function onNotify(event: InflowNotificationEvent): boolean | void {
  if (event.display !== 'toast' && event.display !== 'dialog') return

  useNotificationsStore(pinia).notify({
    level: TOAST_LEVEL[event.severity] ?? 'info',
    title: event.title,
    message: event.message,
  })
  return true
}
```

The reasoning is worth copying: a verification result belongs *against the field it is
about*, where the user is already looking, and a toast for every ↻ press would be noise —
so `inline` and `banner` are left to render in the drawer. `toast` and `dialog` are the
plugin saying *this cannot wait for the user to look down*, and both are served by the one
interruption surface that app has.

---

## Where an unclaimed message renders

| | |
| --- | --- |
| **At its field** | When that field's control carries either Inflow key — so it has a renderer to put it under |
| **Under the form** | Everything else: messages addressed to no field, and ones whose field is a plain control or **was dropped by a re-rendered schema** |

That second row matters if you use the "answer with the next form" pattern from
[the forms chapter](plugin-form-builder.md#the-first-row-is-the-interesting-one): a message
about a field the new schema no longer has is not allowed to vanish with it, so it collects
at the form level.

Both surfaces are slottable, and both are plain by design:

```vue
<InflowForm ... :on-notify="onNotify" @notification="log">
  <template #notifications="{ notifications, dismiss }">
    <Alert v-for="n in notifications" :key="n.key" :tone="n.severity" @close="dismiss(n.key)">
      {{ n.message }}
    </Alert>
  </template>

  <template #status="{ busy, status, error }">
    <Spinner v-if="busy" />
    <Alert v-else-if="error" tone="danger">{{ error }}</Alert>
    <p v-else-if="status">{{ status }}</p>
  </template>
</InflowForm>
```

---

## `notify` vs `setStatus` / `setError`

They are different channels and both stay.

| | |
| --- | --- |
| `setStatus` / `setError` | The **form's own line** — "working…", and the transport failures the package raises itself (`fn` unnamed, no plugin bound, a 502). Not about any field, not routed to the host. |
| `notify` | **About a field** — addressable, replaceable, and offered to the host first. |

A plugin's *own* failure is not an error in the first sense. A meta function that answers
`{"error": "..."}` is **patching a field called `error` into the form**. Answer with
`x-inflow-notif` instead — it is the difference between *"the button does nothing"* and
*"no assignable user matches 'mehdi' — is the project key right?"*.

Buttons disable themselves while their own action runs, and `busy` is **counted**, so two
in flight at once do not have the first to finish clear the second's spinner.

---

## Next

- **[Building a process product on any frontend](build-a-process-product.md)** — both
  packages, in place, in a real product.

**Source material:** `inflow-js/packages/plugin-form-builder/README.md`;
`flomorphic-wapp/src/main.ts`, `src/components/plugin/PluginForm.vue`.

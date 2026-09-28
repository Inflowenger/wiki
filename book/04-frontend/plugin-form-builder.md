# Dynamic forms — `@inflowenger/plugin-form-builder`

> A node you have never seen, from a plugin you did not write, gets a working
> configuration UI in your product — buttons, lookups, cascades and all — without a line
> of code per integration.

This is the problem that quietly kills workflow products. The canvas is a weekend. The
engine you can bind to. But **every integration needs a configuration form**, and a
hand-written form per integration is a maintenance cost that grows linearly with your
catalogue and is paid forever.

The answer is that the plugin already knows what its form looks like, and says so.

```sh
pnpm add @inflowenger/plugin-form-builder
pnpm add vue @jsonforms/core @jsonforms/vue   # peers — bring your own
```

---

## Where the form comes from

An `inflowv1` plugin describes every dialog it wants drawn as a **`FormBuilder`**: a JSON
Schema plus a [JSON Forms](https://jsonforms.io) UI schema. It serves them on two reserved
descriptors:

| Descriptor | The form |
| --- | --- |
| `@intro` / `@settings` | The **settings profile** a plugin needs before any action runs — its onboarding form (host, token, workspace). |
| `<action>.@form` | The **parameters of one action** — the form that action's canvas node renders in its drawer. |

Both are carried on the wire as **strings**, because that is the SDK's format. Both are
rendered by the same component. **A plugin only ever has to speak this one protocol to get
a UI.**

So the sequence, in full, is: your backend asks the plugin for a descriptor → the plugin
answers with a schema pair → your frontend hands that pair to `<InflowForm>` → the user
sees a form. Nothing in that chain knows what Jira is.

> **Parse defensively.** A plugin is a third party; a malformed form is *its* bug and must
> degrade to "no form" rather than break your drawer. FloMorphic's boundary for this is
> `flomorphic-wapp/src/lib/pluginForm.ts`, which tolerates both wire shapes (JSON-in-a-string
> and an already-decoded object) and never throws.

---

## Two schema extensions

JSON Forms renders a form from a schema. Inflow's schemas carry two extra keys describing
what a plain form has no vocabulary for:

| Key | |
| --- | --- |
| **`x-inflow-ui`** | What this field can **do** — *put a button here, and run this action when it is clicked.* |
| **`x-inflow-notif`** | What can be **said** about it — help, a verification result, a warning, a failure — and how this platform should show it. See [What a form can say](notifications.md). |

This package is the renderer that understands both. It is **strictly opt-in**: fields
carrying neither key fall through untouched to whatever renderer set you already use
(vanilla, Vuetify, your own). **Installing it changes nothing about your existing forms.**

---

## The whole integration

```vue
<script setup lang="ts">
import { vanillaRenderers } from '@jsonforms/vue-vanilla'
import { InflowForm } from '@inflowenger/plugin-form-builder'
import '@inflowenger/plugin-form-builder/style.css'
</script>

<template>
  <InflowForm
    :schema="form.schema"
    :uischema="form.uischema"
    v-model:data="body"
    :renderers="vanillaRenderers"
    :plugin-id="pluginId"
    :settings="settingsProfile"
    :call="({ pluginId, fn, body }) => api.pluginFn(pluginId, fn, body)"
  />
</template>
```

**`call` is the only thing that is yours** — the transport, a function that gets a request
to a plugin's meta function and returns the answer. What goes over it, and what happens to
the answer, is the package's job.

Every plugin's UI schema names the same action — the literal string **`pluginFn`** — and
that action **ships built in**. Give the package a way to reach a plugin and a plugin's
form works, buttons and all.

---

## `pluginFn`, the platform's one action

A control's button calls one of the plugin's own **meta functions**, so a form can consult
the live service *while it is being filled in*: turn a typed name into the `accountId` an
API needs, load the labels a project actually has, check a token before 3am does.

**It sends the form flat.** Every field at the top level, plus the action's static `body`,
plus `settings` and `value` (the control the button sits on) — deliberately *not* the
`{_registry, body}` envelope an action execution uses.

**It applies the answer by shape:**

| The plugin returns | The form does |
| --- | --- |
| An object carrying `schema` and/or `uischema` | **Re-renders as that form**, data included |
| Any other object | Patches it in — every key an absolute **leaf** data path |
| Anything else (array, string, number) | Writes it to `action.target`, defaulting to the button's own control |
| `null` / nothing | **Nothing** — a meta function that already wrote a status field does not clear the field it was fired from |

A reserved key, `x-inflow-notif`, is lifted out of any of these *first* and raised as a
message. It rides **alongside** the answer rather than instead of it: a lookup that
resolved a key patches the key **and** says what it found.

### The first row is the interesting one

**A plugin can answer a call with *the next form*, and the dialog becomes it.**

That is also the **only** way to change a `<select>`'s options at runtime, since options
live in the schema, not the data. A cascading picker — pick an account, now pick from
*that account's* projects — is one meta function returning a new schema.

The Go SDK's `jsonschema` / `jsonui` spelling is accepted alongside `schema` / `uischema`,
and schemas arriving as JSON text are parsed, so a handler can return its own `FormBuilder`
verbatim.

> **Telling a re-render from a patch is deliberately conservative.** An object counts as a
> form only when *every* key it has is one of `schema`, `jsonschema`, `uischema`,
> `uiSchema`, `jsonui`, `data`, `submit_to`, `submitTo`. A patch that happens to include a
> field called `schema` also includes `projectKey`, and stays a patch — mistaking one for
> the other would discard the user's work.

---

## Marking up a field

```jsonc
{
  "type": "Control",
  "scope": "#/properties/projectKey",
  "x-inflow-ui": {
    "action": { "name": "pluginFn", "fn": "jira.meta.projects.resolve" },
    "button": { "position": "append", "label": "Find", "icon": "↻" }
  },
  "x-inflow-notif": { "severity": "help", "message": "The key, not the name — OPS, not Operations." }
}
```

The two keys are siblings, and either works alone: **a field may have something to say
without having anything to do.**

| | |
| --- | --- |
| `action.name` | `pluginFn` for a plugin call, or any name you registered |
| `action.fn` | The meta method, exactly as passed to the SDK's `AddMeta` |
| `action.target` | Where a non-object answer lands. Defaults to this control |
| `action.body` | A static object merged into the request — for telling one shared meta function which caller it serves |
| `button.position` | `append` \| `prepend` \| `above` \| `below`, relative to the control that would have rendered anyway |
| `button.icon` | Rendered as **literal text**, not looked up in an icon font. Use a character (`↻`), not `mdi-refresh` |
| `button.label` | Should say what the button will do (`Find user`) — **nothing fires automatically.** There is no on-change hook and no type-ahead |

---

## Registering it app-wide

App-wide defaults go through Vue's provide/inject: the **transport**, your extra
**actions**, and a **theme**.

```ts
import { InflowUiPlugin, createInflowUi } from '@inflowenger/plugin-form-builder'
import '@inflowenger/plugin-form-builder/style.css'

app.use(InflowUiPlugin, createInflowUi({
  call: ({ pluginId, fn, body }) => api.pluginFn(pluginId, fn, body),
  onNotify: (event) => { /* see the notifications chapter */ },
  theme: {
    containerClass: 'my-field',
    buttonClass: 'my-btn',
    iconButtonClass: 'my-btn-icon',
    notifClass: 'my-alert',
  },
  actions: {
    async loadLabels(ctx) {
      ctx.updateData({ labels: await api.labelsFor(ctx.data) })
    },
  },
}))
```

Configure it once and every `<InflowForm>` inherits it; each form still supplies the
`pluginId` and settings profile it is bound to.

> **Your actions layer over the built-ins, and a name collision replaces one.** Registering
> your own `pluginFn` — app-wide or on one `<InflowForm>` — is how you take the whole thing
> over. **Defaults are a floor, not a lock.**

---

## Customising one action without rewriting it

Every action is handed a context that includes the **default handling as a step**, so
overriding part of the behaviour does not mean reimplementing the rest:

```ts
actions: {
  async pluginFn(ctx) {
    const result = await myTransport(ctx.config.fn, {
      ...ctx.rootData,            // the whole form, not just this field
      settings: ctx.host.settings,
    })

    if (result.needsConfirmation && !confirm(result.prompt)) return

    ctx.applyResult(result)       // ← the default handling, as its own call
    ctx.setStatus('Loaded.')
  },
}
```

| `ctx` | |
| --- | --- |
| `data` | The value at the renderer's scope — the field the button sits on |
| `rootData`, `rootSchema`, `rootUiSchema` | The whole form |
| `schema`, `uischema`, `path` | Where the action was fired from |
| `config` | This button's `x-inflow-ui.action` — `fn`, `target`, `body` |
| `host` | `pluginId`, `settings`, `call` |
| `updateData(patch)` | Apply changes. Keys are absolute **leaf** data paths |
| `updateSchema`, `updateUiSchema`, `updateForm(envelope)` | Replace the document and re-render |
| `applyResult(result, { target? })` | The default handling, as its own step |
| `setStatus`, `setError` | Report on the form's own line, about nothing in particular |
| `notify(message)`, `clearNotifications(field?)` | Say something **about a field** — routed to the host |

### The two things that bite people

**1. Effects go through the context, not the return value.**

```ts
type InflowAction = (ctx: InflowActionContext) => void | Promise<void>
```

The signature is `void` **on purpose**; there is no return channel back into the form. An
action that computes a value and returns it silently does nothing.

**2. Patch leaf paths, not objects.**

```ts
ctx.updateData({ 'connection.host': 'api.example.com' })    // sets host
ctx.updateData({ connection: { host: 'api.example.com' } }) // REPLACES connection,
                                                            // dropping port, secure, ...
```

Both are easier to *see* than to read about. The `inflow-js` lab's **Form Preview** page
logs every action call alongside what it saw and the patch it wrote, and its Jira sample's
Reload button patches leaves under `connection` while port and TLS survive.

> Actions resolve **by name from the registry**, so a schema can never invoke anything the
> app did not provide. An unregistered name is **inert rather than fatal** — the button
> reports itself and does nothing. That is the security property that makes rendering a
> third party's schema acceptable at all.

---

## Using your own `<JsonForms>`

`<InflowForm>` is a wrapper, not a requirement. The renderers work under a bare
`<JsonForms>`:

```ts
import { markRaw } from 'vue'
import {
  InflowControlRenderer, InflowLayoutRenderer,
  inflowControlTester, inflowLayoutTester,
} from '@inflowenger/plugin-form-builder'

const renderers = [
  { tester: inflowControlTester, renderer: markRaw(InflowControlRenderer) },
  { tester: inflowLayoutTester, renderer: markRaw(InflowLayoutRenderer) },
  ...vanillaRenderers.map((r) => ({ tester: r.tester, renderer: markRaw(r.renderer) })),
]
```

Both testers rank at **1000**, above the defaults, but only match elements that actually
carry an Inflow key — so array order does not matter and unmarked fields never reach them.
`markRaw` matters: without it Vue makes the component definitions reactive and JSON Forms
gets slower for nothing. The layout tester covers `VerticalLayout`, `HorizontalLayout`,
`Group`, `Category` and `Categorization`.

**Two things do not work this way**, both because they need state that outlives a re-seeded
core — which is exactly what `<InflowForm>` owns:

- **A replaced schema does not last.** `<JsonForms>` re-seeds its core whenever `data`
  changes, so in a host that two-way binds data, a schema an action pushed in is reverted
  by the next keystroke. This is why FloMorphic's `PluginForm.vue` notes that using
  `<InflowForm>` rather than a bare `<JsonForms>` is **load-bearing**.
- **Raised messages have nowhere to live.** Declared `x-inflow-notif` help still renders,
  but a message an action raises goes to your `onNotify` if you configured one, and to the
  console if you did not.

Everything else — buttons, patches, status — behaves identically.

---

## Styling: four class names

`style.css` is **geometry, not a design system**. It carries what a decorated field needs
to hold together at all — the parts every host was otherwise re-deriving:

- **The button sits on its input's row.** A decorated control is a flex row of
  `[prepend] [control] [append]`, but the control is itself a column of
  label / input / error / description whose height changes as validation comes and goes.
  A centred button drifts down the moment an error appears. The row is a grid, the
  control's rows are promoted into it, and the button takes the input's row and stays there.
- **A checkbox stays a checkbox** rather than being stretched across the field.
- **`above` / `below` buttons do not stretch** to full width and read as banners.
- **Notifications are laid out, not decorated** — the one exception in the whole stylesheet
  being severity colour, because telling an error from a help line is the entire point.

Everything visible is still yours:

```ts
theme: {
  containerClass: 'my-field',
  buttonClass: 'my-btn',
  iconButtonClass: 'my-btn-icon',
  notifClass: 'my-alert',
}
```

Those land alongside the package's own classes, and every rule is scoped inside the
`.inflow-*` elements this package renders — **none of them reach markup it does not own.**

This is how FloMorphic makes a third-party form look like FloMorphic in both light and dark
mode while the package stays design-system-agnostic: the four theme classes point at the
same design tokens as the rest of the app (`flomorphic-wapp/src/main.ts`), and JSON Forms'
own `styles` injection is merged with `mergeStyles` so vanilla's structural classes survive.

### The optional vanilla fixes

`@jsonforms/vue-vanilla` has a few layout bugs of its own that every host ends up
re-deriving. They ship as a second, **opt-in** stylesheet:

```ts
import '@jsonforms/vue-vanilla/vanilla.css'
import '@inflowenger/plugin-form-builder/style.css'
import '@inflowenger/plugin-form-builder/vanilla-fixes.css'
```

| | |
| --- | --- |
| Empty error / description rows | Rendered with a `min-height` whether or not they say anything — ~3em of dead space per control. Collapsed until they have text. |
| Checkboxes | `flex: 1` zeroes a checkbox's basis and smears it across the field. Reset, and re-placed as "box then label". |
| Arrays | A `<fieldset>` styled `display: flex` straddles its `<legend>` across the border. Put back as a block. |
| Arrays of plain values | Every item an accordion headed by its index — a list of bare numbers, one click per field. Collapsed onto one row. |

It is opt-in because it targets vanilla's generic class names (`.control`, `.wrapper`,
`.array-list`); importing it is how you say you use that renderer set.

---

## What is deliberately left to you

**Enum and `oneOf` controls** fall through to the host's own renderer set. The package
stays loosely coupled and compatible across JSON Forms versions, and you keep the widgets
your design system already has.

---

## Exports

Component `InflowForm`; renderers `InflowControlRenderer`, `InflowLayoutRenderer`,
`InflowActionButton`, `InflowNotifications`; testers `inflowControlTester`,
`inflowLayoutTester`, `inflowTesters`; plugin `InflowUiPlugin`, `createInflowUi`; the
built-in `pluginFn` with its name as `PLUGIN_FN`, plus `builtinActions`,
`builtinActionNames`, `withBuiltinActions`; shape helpers `isFormEnvelope`,
`normalizeFormEnvelope`, `patchFor`, `applyActionResult`; message helpers `NOTIF_KEY`,
`buildNotification`, `splitNotifications`, `readNotifExtension`, `hasNotifExtension`,
`notifDefaults`, `notificationKey`, `reportToConsole`; registry helpers
`createInflowActionRegistry`, `resolveAction`; and `createInflowFormController` /
`useInflowActionContext` for building on top — plus the injection keys and all types.

---

## Next

- **[What a form can say](notifications.md)** — `x-inflow-notif`, the other extension key.

**Source material:** `inflow-js/packages/plugin-form-builder/README.md`;
`inflow-plugin-sdk/docs/form-builder.md`; `plugin-catalog/docs/dependent-fields.md`;
`flomorphic-wapp/src/lib/pluginForm.ts`, `src/components/plugin/PluginForm.vue`,
`src/main.ts`.

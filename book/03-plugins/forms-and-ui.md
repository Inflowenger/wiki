# Forms — a node with its own UI

> **Status: outlined.**

The property that most separates a plugin node from a compiled one: **it ships its own
configuration UI**, and that UI can talk back to the plugin while it is open.

## The stack

- Forms are **JSON Schema + UI Schema**, rendered by **[JSON Forms](https://jsonforms.io)**.
- Inflowenger extends them with the **`x-inflow-ui`** key — behaviour a plain form has no
  vocabulary for: *put a button on this field, and run this action when it is clicked.*
- **`@inflowenger/plugin-form-builder`** (Vue 3 + TS) is the renderer that understands that
  key. It is one of the two packages in
  [`inflow-js`](https://github.com/Inflowenger/inflow-js); the other is
  [`flow-trace`](../02-fusion/observing-a-run.md#consuming-it-inflowengerflow-trace).

```sh
pnpm add @inflowenger/plugin-form-builder
pnpm add vue @jsonforms/core @jsonforms/vue   # peer dependencies — bring your own
```

The design is **strictly opt-in**: fields without `x-inflow-ui` fall through untouched to
whatever renderer set you already use (vanilla, Vuetify, your own). Installing it changes
nothing about your existing forms.

## Sections planned

**1. Declaring a form on an action.** The schema/UI-schema pair, and how the host fetches it
over `<ACTION>.@form`.

**2. The `x-inflow-ui` contract.** Control and layout renderers that attach an action
button; the theme config (three class names — `containerClass`, `buttonClass`,
`iconButtonClass` — the entire styling contract, so the package ships no opinions about
your design system); and the action registry.

**3. The host registers the actions a form may call, by name.** The production host
registers exactly **one**: `pluginFn`, which calls a plugin **meta function** and patches
its answer into the form. That single action is the mechanism behind every dependent field.

**4. Dependent fields in practice.** Lookups, cascades and connection tests — a picker that
shows what *this* account can actually see, because the form asked the plugin while it was
open.

**5. The two edges that bite people.** Effects travel through `updateData`, not through a
return value; and a patch touches leaf paths while leaving the rest of the object alone.
Both are easier to *see* than to read about — the `inflow-js` `lab/` app has a **Form
Preview** page that logs every action call alongside what it saw and the patch it wrote.

**6. The known limitation, stated plainly.** `x-inflow-ui` answers are patched into form
*data* only, so a `<select>` cannot currently have its `enum` populated live. The two
candidate fixes (implementing `updateSchema` in the renderer, or an `x-inflow-ui.options`
source read from form data) are recorded as open.

**7. What is deliberately left to the host.** Enum and `oneOf` controls fall through to the
host's own renderer set.

**8. Settings profiles vs. action forms.** Which form is which, and the open question of
whether `PluginIntro.Settings` and `RequiredParams` are meant to coexist or are two
iterations of one idea.

## Source material

`inflow-plugin-sdk/docs/form-builder.md`, `plugin-catalog/docs/dependent-fields.md`,
`inflow-js/README.md` and `packages/plugin-form-builder`, `inflow-js/lab/`.

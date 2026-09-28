# Spaces and isolation

> **Status: outlined.**

## The central idea

**Infra models tenants as NATS accounts, not database rows.** A *space* **is** an account —
the unit of authentication, authorisation and isolation. Isolation is therefore enforced by
the message substrate itself, not by application code remembering to filter.

Three built-in accounts exist: `sys`, `inflow`, `plugins`. Your backend authenticates as
`inflow`. Plugins get credentials scoped to their own account and, within it, to a subject
prefix unique to that plugin instance — so **one plugin instance cannot observe another's
traffic even though they share NATS infrastructure.**

## Sections planned

**1. Why substrate-level isolation is different in kind.** A forgotten `WHERE tenant_id = ?`
is a data breach. A forgotten subject scope is a connection that will not authorise. The
failure mode moves from silent to loud.

**2. Credentials.** Decorated NATS `.creds` (JWT + NKey seed), base64-encoded into
`INFRA_CRED`. The account is read from the JWT. What a plugin can and cannot do with one.

**3. Strict plugin permissions.** `PluginCredentialStrictPermission` and
`GetInboxConfigWithPluginId` — how a plugin's publish/subscribe surface is narrowed to its
own namespace.

**4. The custom inbox prefix.** A JWT may carry a custom inbox prefix (the SDK honours an
`_INBOX*` tag). Its purpose is **hardening accessibility when many plugins share one
account** — scoping each plugin's private reply inboxes so co-tenants cannot reach them. A
security knob, not a conceptual layer.

**5. Single-tenant vs. multi-tenant.** The built-in plugins space for single-tenant use;
**custom accounts** for multi-tenant and enterprise setups, isolating accessibility and
domain scope per tenant or trust boundary. A plugin must be **defined in a space** before it
can connect — that registration is what makes its `PLUGIN_ID` reachable.

**6. The never-dial-NATS-yourself rule, as a security property.** The SDK is the single
choke point that mints the right credentials and builds the right subjects. Bypassing it is
not merely untidy — it is how a plugin ends up reached on the wrong account.

**7. Origin tagging.** Plugin-originated service calls carry `origin: plugin:<node title>`,
so a backend can refuse ungranted plugin-initiated calls. Authorisation at the application
boundary, on top of substrate isolation.

**8. What is not yet settled.** The exact Infra API flow to define a plugin in a space and
mint its credential, and how domain scopes map to NATS subject permissions, are recorded as
*to verify* in the SDK's own notes. Flagged rather than invented.

## Source material

`inflow-fusion/docs/architecture.md` (Multi-tenancy / isolation), `spaces/` package docs,
`inflow-plugin-sdk/docs/inflow-ecosystem.md` (NATS as the platform bus).

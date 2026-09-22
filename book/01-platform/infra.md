# Infra — the control plane

> **Status: outlined.**
>
> **Infra is closed source.** This chapter documents what it owns, what it guarantees, and
> the contract you build against — not its internals.

## What Infra is

The service that everything starts from. It bootstraps the ecosystem: runs an **embedded
NATS server**, mints accounts and credentials, hosts the onboarding portal, and keeps the
registry of live execution engines.

It is headless. It has no opinion about your domain, no UI beyond onboarding, and no data
of its own beyond identity and topology.

## What it owns

| Responsibility | What it means in practice |
| --- | --- |
| **The message substrate** | An embedded NATS server. Every other component is a client. |
| **Identity** | Accounts are **NATS accounts**, not database rows. Three built-ins: `sys`, `inflow`, `plugins`. |
| **Credentials** | Mints decorated NATS `.creds` (JWT + NKey seed), scoped per account and per plugin. |
| **Spaces** | A space *is* an account — the unit of auth, authz and isolation. |
| **The engine registry** | Which Fractal instances exist and are alive. This is what round-robin dispatch reads. |
| **The API Secret Key** | The shared HMAC secret. Possession of it is the credential; there is no login flow. |

## Sections planned

**1. The bootstrap sequence.** Operator key → sys account → builtin accounts → resolver
service → API server → NATS server → listen for Fractal registrations. Why "everything
starts here" is literal.

**2. The REST contract.** The three endpoints `inflow-fusion` calls, their envelopes, and
the bearer-JWT scheme. (Detailed in
[Part II — The backend contract](../02-fusion/the-backend-contract.md#the-rest-surface-for-reference).)

**3. Accounts, spaces and scoped credentials.** How a plugin credential is narrowed to a
subject prefix unique to that plugin instance, so co-tenants of one account cannot reach
each other's inboxes. The custom inbox prefix as an access-hardening knob.
→ [Part VI — Spaces and isolation](../06-architecture/spaces-and-isolation.md)

**4. The resource registry and dispatch.** How a Fractal registers, what a *portal* record
carries (including `subscribe_prefix`, the event-log subject, and an optional per-instance
`jwt_secret`), and how pinning overrides round-robin.

**5. Clustering.** Enterprise deployments may run Infra as a cluster with multiple instance
endpoints — which is why `INFRA_URL` is always explicitly required and never assumed.

**6. The licence dimension.** `INFRA_CLUSTER` and the licence-manager component; what is
gated and what is not.

**7. Failure modes.** Infra down: what still runs (in-flight processes) and what stops
(new credentials, new registrations, new dispatch). What a backend should do about it.

## Source material

`Inflowenger/getting-started` README, `inflow-fusion/docs/infra.md`,
`inflow-fusion/docs/architecture.md`, `inflow-plugin-sdk/docs/inflow-ecosystem.md`,
`infra/Readme.md` (NATS resolver configuration).

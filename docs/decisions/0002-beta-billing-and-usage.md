# ADR 0002: Social Hub beta billing, entitlements, usage events, and abuse controls

- **Status:** Accepted
- **Date:** 2026-11-14
- **Scope:** Social Hub SaaS operating mode on the `development` branch
- **Supersedes:** No upstream OpenPost billing behavior; this decision defines the Social Hub product layer.

## Context

The upstream Cloud Edition currently treats Paddle as a mandatory part of cloud runtime configuration. The existing entitlement service resolves cloud usage from active billing subscriptions and falls back to a restrictive bootstrap snapshot. The self-hosted entitlement service treats unspecified limits as unlimited, so changing a public SaaS deployment to self-hosted mode is not an acceptable way to bypass Paddle.

The repository already has:
- Workspace-level monthly usage counters and transactional quota checks.
- A separate provider-cost event/reservation subsystem, including X API spend guardrails.
- A shared database-backed rate limiter and authentication rate-limit storage.
- Paddle-specific checkout, portal, webhook, and browser configuration paths.

These foundations are useful but do not yet constitute the Social Hub beta billing/usage design.

## Decision

Social Hub will launch its initial beta in **Cloud Edition with Paddle disabled by an explicit provider flag, a first-party `beta_free` plan, an auditable product-usage event ledger, and independent abuse controls**.

The following are mandatory invariants.

### 1. Paddle is an optional provider, explicitly gated

- Introduce a server-side provider setting such as `OPENPOST_BILLING_PROVIDER=disabled|paddle`. The final implementation may choose a clearer Social Hub-prefixed name, but it must be explicit and validated.
- Social Hub beta deployment uses `disabled`. Missing Paddle credentials must not accidentally activate Paddle or prevent Cloud Edition from starting.
- `paddle` mode must fail startup validation unless every required Paddle credential, environment, webhook secret, and price ID is valid.
- Paddle SDK initialization, Paddle webhook registration, checkout/portal/purchase-choice APIs, and Paddle-specific background work must be unreachable when the provider is disabled. Hiding UI alone is insufficient.
- Do not silently switch to self-hosted edition. Cloud database, object storage, security, legal, email, telemetry, provider approval, and operational requirements remain in force.
- When disabled, the API must return a deliberate stable unavailable/not-enabled response for direct requests to Paddle-only endpoints. No half-configured checkout state is allowed.

### 2. Beta is a first-party plan, not a fake Paddle subscription

- Define a versioned plan with stable ID `beta_free` and an explicit entitlement snapshot.
- Plan limits must be configured centrally and enforced by the existing entitlement gates. No absent/unknown limit may accidentally become unlimited for this plan.
- Beta grants are first-party entitlements, not Paddle subscriptions. They must not require Paddle customer IDs, Paddle subscription IDs, checkout attempts, or webhook events.
- Implement a dedicated entitlement-grant record/resolver (or an equivalently explicit first-party entitlement domain) rather than overloading Paddle subscription rows with fabricated provider data.
- Grants are organization/workspace scoped, auditable, idempotent, and have explicit active/revoked/effective states. The resolver must enforce tenant ownership and define precedence between a beta grant and any future paid subscription.
- Keep plan definitions and limit snapshots versioned so plan changes do not retroactively rewrite historical usage or silently alter existing grants.
- The actual numeric beta quotas are a separate product configuration decision. They must be approved before public registration is enabled; do not invent or silently default them to unlimited.

### 3. Product usage has an append-only event ledger

- Add a first-party `usage_events` ledger for billable/quota-relevant product actions. It is distinct from Paddle webhook events, provider-cost events, PostHog telemetry, and operational/security logs.
- Each event must have a stable event ID, tenant scope (organization/workspace), optional actor, metric/action, positive quantity, source, occurrence time, and an idempotency key. Store only allowlisted non-sensitive metadata.
- Never store post text, message bodies, media bytes, OAuth tokens, secrets, full provider payloads, or unnecessary personal data in usage events.
- Event insertion and the corresponding quota counter/projection update must occur atomically with the domain mutation wherever possible. Retries must not double count; idempotency must be enforced in the database.
- Keep `usage_counters` as an efficient projection for quota checks and UI summaries, not as the sole audit record.
- Define retention, export, deletion, and aggregate-preservation behavior before launch. Detailed event retention must not be conflated with account/security audit retention.
- Existing provider-cost events/reservations remain a separate ledger because estimated third-party financial exposure has different semantics from product usage.

### 4. Quotas and abuse prevention are independent control planes

**Entitlement quotas** answer: “How much of this product feature may this tenant consume under its plan?” They are tenant/plan scoped and measured over defined periods.

**Abuse/security controls** answer: “Is this actor, credential, IP/network, or operation behaving in a way that threatens service integrity?” They include rate limits, authentication throttles, request/concurrency limits, bot/automation controls, and suspicious-activity handling.

- Abuse controls must apply regardless of plan and must never be disabled or increased merely because a tenant has a higher quota.
- A tenant reaching a plan quota must not be treated automatically as malicious.
- A rate-limit or abuse-control denial must not be recorded as successful product usage.
- Provider-specific cost/spend budgets (including X API safeguards) remain an additional independent operational guardrail, not a substitute for either plan quotas or abuse prevention.
- Use shared/distributed enforcement for SaaS replicas; do not rely on process-local memory as the only production protection.

### 5. Payment UI and checkout routes disappear in beta mode

- The backend is the source of truth for billing capabilities. The frontend must consume an explicit capability/config response rather than infer mode from missing Paddle data.
- When Paddle is disabled, hide payment plan comparison, price cards, checkout buttons, payment methods, billing portal actions, and Paddle-specific notices/navigation.
- The beta account/workspace settings page should show the active beta plan and its usage/limits, not a broken subscription screen or an empty payment form.
- Direct navigation to checkout routes must redirect to a clear beta/account page or show a neutral unavailable page. It must never load Paddle.js or expose Paddle configuration.
- Re-enabling Paddle later is an explicit operator action and a tested product-mode transition, not a UI-only toggle.

## Target architecture

1. **Billing provider selector:** explicit `disabled|paddle` mode, validated at startup.
2. **Entitlement resolver:** first-party beta grants plus optional Paddle-backed paid entitlements, with deterministic precedence and workspace/organization ownership checks.
3. **Plan catalog:** versioned `beta_free` definition and future paid plan definitions, independent of provider-specific price IDs.
4. **Usage ledger:** immutable `usage_events` plus idempotent monthly counter projections.
5. **Abuse controls:** independent shared rate limiting and security/automation policies.
6. **Provider cost controls:** existing provider-cost events/reservations remain separate.
7. **Capability-driven UI:** server-reported billing mode and capabilities determine whether billing/payment surfaces exist.

## Required implementation sequence

1. Add configuration/provider-mode tests and make Cloud Edition validation conditional on the explicit provider mode.
2. Add the first-party entitlement-grant model, migration, resolver, tenant-isolation checks, and grant lifecycle.
3. Define beta quotas and enforce them on all corresponding mutation paths; add boundary and concurrency tests.
4. Add the usage event ledger, idempotency constraints, transactional writes, projection/reconciliation, retention, and privacy tests.
5. Audit rate limiting and abuse controls separately from entitlement gates; verify shared enforcement across replicas.
6. Gate Paddle APIs, webhooks, background jobs, SDK setup, frontend queries, navigation, and checkout routes behind the provider capability.
7. Add end-to-end tests for beta mode (no Paddle secrets, no payment UI, working beta entitlements and usage) and Paddle mode (explicit enablement, strict validation, checkout/webhook regression).
8. Run Go, web, database integration, migration, and browser tests in CI before enabling public beta registration.

## Explicit non-goals

- Do not remove Paddle integration or paid plan catalogue in this phase.
- Do not create fake Paddle subscriptions for free beta users.
- Do not treat usage counters as a sufficient audit ledger.
- Do not merge abuse/rate limits into plan quota checks.
- Do not change `main`; all work stays on `development`.

## Open product configuration

The following values remain intentionally unset until approved:
- Numeric beta quotas for workspaces, connected social accounts, scheduled/published posts, media storage and uploads, provider write calls, team members, and feature flags.
- Detailed usage-event retention period and deletion/export policy.
- Abuse-control thresholds and escalation/appeal policy.

These are launch gates, not permission to run with unlimited or unmetered defaults.

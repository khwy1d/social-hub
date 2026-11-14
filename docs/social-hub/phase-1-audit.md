# Social Hub — Phase 1 Repository Audit

Date: 2026-11-14
Repository: https://github.com/khwy1d/social-hub
Upstream: https://github.com/getopenpost/openpost
Working branch: `development`

## 1. Repository provenance and baseline

- Confirmed that `khwy1d/social-hub` is accessible and the connected GitHub identity has push/admin permissions.
- The repository root listing and the Git blob SHAs checked for `README.md`, `LICENSE`, `docker-compose.yml`, `package.json`, `apps/web/package.json`, and `apps/server/go.mod` match the upstream repository. This is consistent with a clean fork at the inspected revision.
- Created the `development` branch from `main`. No upstream files have been changed in this audit.
- License is AGPL-3.0-only. Any modified network-accessible derivative must be handled in accordance with AGPL-3.0, including its source-offer obligations. Obtain legal review before deciding which parts of a commercial service can remain proprietary.

## 2. Existing architecture

- Web application: SvelteKit, TypeScript, Tailwind CSS, Bun workspaces.
- Backend: Go, Echo, Huma, Bun ORM.
- Native application: Expo / React Native under `apps/mobile`; it consumes the same versioned API contract as the web app.
- Persistence: SQLite is the self-hosted default; PostgreSQL support is present.
- Media: local filesystem by default; S3-compatible object storage is supported.
- Background work: durable database-backed jobs are used for publishing and other work that must survive restarts.
- Deployment: the web build is embedded in the Go server binary. The default Docker Compose file runs one application container with a persistent data volume.

## 3. SaaS foundations already present

The codebase already contains substantial foundations for a multi-user product:

- Organizations, workspaces, memberships, invitations, and role-based access.
- Workspace-scoped authorization, including credential workspace boundaries and organization identity policy checks.
- Plan catalog, entitlements, usage limits, and billing integration.
- Cloud edition configuration exists. Its documented production configuration requires PostgreSQL, S3-compatible media storage, and Paddle billing settings.
- The existing Hosted billing flow is coupled to Paddle and the Hosted plan catalogue. This needs an explicit product decision before reuse: Social Hub should not require payment-provider credentials merely to run an initial free beta unless that is intentionally part of the business model.
- The self-hosted edition has permissive defaults unless an operator configures limits. Those defaults are not, by themselves, a safe public-SaaS policy.

## 4. Social platform coverage

The current capability catalogue and platform package include adapters for the following relevant platforms:

- Facebook
- Instagram
- X
- TikTok
- YouTube
- LinkedIn

The codebase also includes Threads, Bluesky, Mastodon, Pinterest, Discord, Telegram, and other providers.

**Snapchat is not present in the inspected platform constants/adapter inventory.** It remains a separate integration project and must be based on Snap's approved API access and documented capabilities. Do not emulate private APIs or automate the consumer website/app to bypass API restrictions.

Inbox, comments, analytics, and engagement are capability-specific rather than universal. A provider's publishing adapter does not prove that its messaging, comments, or analytics are supported. Build a verified provider-by-feature matrix and live-test each supported account type before making product claims.

## 5. Important integration and operational constraints

- X has provider-side usage costs. Existing code contains Hosted X budget/cost guardrails; preserve and review them rather than bypassing them.
- Provider OAuth requires a stable public HTTPS application URL and correct callback registration.
- Some providers require publicly fetchable HTTPS media URLs.
- Email delivery must be configured before enabling mandatory email verification, password recovery, or invitations.
- Production operations require backups of the database, media, deployment configuration, and secrets as one restore set.
- Use pinned release versions for deployment; do not rely on a floating `latest` image for production.

## 6. Decisions required before implementation

1. Separate Social Hub's SaaS operating mode from the upstream Hosted product's Paddle-specific billing assumptions.
2. Define initial free-beta quotas for users, workspaces, connected accounts, posts, media storage, scheduled jobs, and provider API spend.
3. Decide whether PostgreSQL and object storage will be provided by one free-tier host initially or split across services. Keep backups independent of the primary host.
4. Define the tenant isolation test plan: cross-workspace reads/writes, media access, OAuth credentials, jobs, webhooks, API tokens, invitations, and deletion.
5. Define the supported provider-by-feature matrix, with Snapchat explicitly marked unavailable until approved API access is obtained.
6. Define abuse prevention, rate limits, audit logs, data deletion/retention, privacy policy, terms, and support contact before public registration.

## 7. Verification status

- GitHub repository access and write permissions: verified.
- Fork baseline: verified through matching repository contents and blob SHAs for the files listed above.
- Development branch: created.
- Source inspection: completed for the repository guide, architecture, production-readiness notes, workspace authorization, billing service, platform registry/capability catalogue, and mobile application documentation.
- Local dependency installation, application startup, automated tests, and production build: **not run**. The available execution environment could not resolve `github.com`, so a local clone could not be fetched. No claim of a successful runtime test is made.

## 8. Next implementation sequence

1. Complete a security-focused tenant-isolation review of workspace-scoped handlers, background jobs, media, tokens, and webhooks.
2. Trace the current Hosted/cloud edition checks and decouple billing from the initial beta operating mode if required.
3. Define and implement Social Hub-specific quotas and abuse controls with tests.
4. Add a provider capability matrix and decide the Snapchat adapter boundary.
5. Prepare local development and CI checks, then run backend/frontend/mobile test gates before any public deployment.


## 9. Tenant-isolation review — 2026-11-14

### 9.1 Central workspace authorization

Reviewed:
- `apps/server/internal/api/handlers/workspace_access.go`
- `apps/server/internal/services/workspaceaccess/access.go`
- `apps/server/internal/services/workspaceaccess/context.go`
- `apps/server/internal/services/workspaceaccess/access_test.go`
- `apps/server/internal/services/workspaceaccess/role_matrix_test.go`

Findings:
- The backend has a centralized authorizer rather than relying only on UI filtering. It checks the authenticated user, optional credential-bound workspace, active workspace membership, role capability, and organization identity/SSO policy.
- The role matrix is explicit: Admin can administer; Editor can edit; Client can connect accounts and approve but cannot author/administer; Viewer is read-only.
- Organization membership alone does not grant access to workspace content. Inactive workspace members are denied.
- A credential bound to one workspace is denied when it requests another workspace.
- Stored workflow authority is a separate path. It checks the persisted user/workspace/organization relationship and current active membership/role. Its comments explicitly limit it to previously authorized native workflows; HTTP authentication does not populate this context. Identity assurance is treated as evidence captured by the initiating workflow rather than a live session.
- Tests cover role levels, inactive members, credential scope, and organization SSO assurance.

Assessment: **Strong existing foundation.** Keep all state-changing handlers on the central authorizer. Do not use role-only helpers such as `RoleMeetsLevel` as a substitute for a full authorization decision.

### 9.2 API tokens and credential boundaries

Reviewed:
- `apps/server/internal/api/handlers/api_tokens.go`
- `apps/server/internal/services/apitokens/service.go`
- `apps/server/internal/api/middleware/auth.go`
- `apps/server/internal/api/handlers/workspaces_credential_access_test.go`

Findings:
- Workspace-scoped token creation resolves the requested/default workspace and verifies read access before issuing the token.
- Token creation also evaluates organization token policy, SSO assurance, reauthentication, and policy-controlled expiry.
- Tokens are stored as hashes; the raw token is returned once.
- Token principals carry workspace scope into middleware, and API operation scopes are separately checked.
- Credential-access tests exercise workspace-bound and unbound credentials, including organization listing and workspace team/invitation operations.

Assessment: **Good baseline.** For Social Hub, default newly created integration tokens to the narrowest practical scope and workspace binding. Keep unbound credentials restricted to operations that independently authorize each workspace.

### 9.3 Media isolation

Reviewed:
- `apps/server/internal/api/handlers/media_credential_access_test.go`
- `apps/server/internal/api/handlers/media_security_test.go`
- `apps/server/internal/api/handlers/public_media.go`
- Relevant media handler implementation and media signer/storage services.

Findings:
- The media credential tests create two workspaces and verify that a credential for workspace 1 cannot read private media belonging to workspace 2, even when using either Authorization headers or query-token transport.
- Anonymous access and insufficient scopes are denied without returning original or preview bytes.
- A valid workspace-bound token and an authorized browser session can access the media.
- Time-limited signed media URLs are a separate intentional public-delivery mechanism; they are bounded by expiry/signature rather than ordinary user-session authorization.
- Upload tests reject active HTML/XML content and enforce safe response headers for stored files (attachment disposition, octet-stream fallback, nosniff, sandbox CSP).

Assessment: **Good baseline with an explicit public-link boundary.** Social Hub must keep signed URLs short-lived, avoid exposing them in logs, and ensure every media derivative/reference remains attached to the same workspace. Review any new export, thumbnail, CDN, or object-store route against the same isolation tests.

### 9.4 OAuth credentials and background jobs

Reviewed:
- `apps/server/internal/api/handlers/oauth_grants_test.go`
- `apps/server/internal/services/tokenmanager/jobs.go`
- OAuth grant/account models and workspace authorization call sites.

Findings:
- Social accounts and OAuth grants are workspace-associated; grant refresh scheduling resolves the target workspace from the grant/account before enqueueing and locks the workspace through the organization guard.
- New refresh jobs are grant-scoped; the account-scoped path remains for compatibility with older jobs/fixtures.
- OAuth tests cover shared grants, disconnect-versus-revoke behavior, and preserving a grant when provider-side revocation fails so it can be retried.
- The inspected refresh-job path is system work: it is scoped to a persisted grant/account, not to an HTTP user session. This is appropriate only while the worker re-resolves the grant/account and its workspace at execution time.

Assessment: **Promising, but needs a broader worker audit before public launch.** Audit every job type—not only token refresh—for payload-to-record resolution, workspace ownership checks, revocation/deletion behavior, retry semantics, and stale-job handling. Do not trust a workspace ID supplied only in a job payload when it can be derived from the persisted target record.

### 9.5 External webhooks

Reviewed:
- `apps/server/internal/api/handlers/external_webhooks.go`
- `apps/server/internal/services/externalwebhooks/service.go`

Findings:
- Creating a webhook requires an active external-app installation grant for the requested workspace and the `events:subscribe` delegated scope.
- Listing subscriptions and deliveries filters them through the current installation/workspace grant.
- The delivery worker rechecks the installation grant and subscription revocation immediately before sending; revoked authorization cancels pending delivery.
- Secrets are generated randomly, encrypted at rest, and used to sign timestamped payloads with HMAC.
- Webhook destinations are HTTPS-only and pass through URL/network validation, reducing SSRF exposure.

Assessment: **Good baseline.** Preserve the grant recheck at delivery time. Add tenant-isolation tests for revoked grants, deleted workspaces, and cross-workspace delivery attempts in Social Hub's own CI.

### 9.6 Workspace deletion and lifecycle

Reviewed:
- `apps/server/internal/api/handlers/workspaces_delete_test.go`
- Workspace deletion service and lifecycle audit paths.

Findings:
- Tests verify deletion of workspace content, media records, notifications, and workspace-owned jobs while retaining another workspace's data and jobs.
- Workspace deletion requires owner-level authorization, canonical name confirmation, and recent authentication.
- Lifecycle audit records retain the organization, actor, workspace name, and action.

Assessment: **Good baseline.** Before launch, confirm object-store bytes, cached derivatives, OAuth grants, queued provider actions, webhook deliveries, and backups follow a documented deletion/retention policy—not just database-row cleanup.

### 9.7 Findings, severity, and remaining work

No confirmed cross-tenant data disclosure or authorization bypass was identified in the specific paths inspected in this pass. This is a source review, not a penetration test, and does not establish that every handler or worker is safe.

| Priority | Finding / required action | Status |
|---|---|---|
| High | Complete route-by-route audit of every handler that accepts `workspace_id`, record ID, account ID, media ID, or organization ID; verify authorization is based on the persisted record's workspace. | Open |
| High | Inventory every background job and verify it re-resolves tenant ownership at execution time, including stale/retried jobs after membership revocation or workspace deletion. | Open |
| High | Add automated cross-tenant negative tests for publication, comments/inbox, analytics, media derivatives, OAuth accounts/grants, exports, webhooks, API/MCP, and deletion. | Open |
| High | Verify S3 object keys and signed URL generation cannot be used to address another tenant's object; ensure signed URLs are intentionally public, expiring, and not logged. | Open |
| Medium | Verify all external webhook delivery paths recheck active grant and workspace ownership, including retries and deletion races. | Partially reviewed |
| Medium | Document and test the trusted stored-authority boundary; only internal/native workflows may attach this evidence, never client-controlled HTTP input. | Open |
| Medium | Verify audit logs, retention, account export/deletion, and backups preserve tenant boundaries and privacy requirements. | Open |

### 9.8 Recommended next action

Do not rewrite the existing authorization framework. It is a substantial part of the upstream project's security model. The next pass should build a complete route/job inventory and add Social Hub-specific cross-tenant regression tests around any custom code. Only after that should we change billing/edition behavior or introduce new tenant-facing features.

## 10. Verification status update

- Read the tenant authorization service and its role/access tests on branch `development`.
- Read representative API-token, media-access, OAuth-grant, webhook, and workspace-deletion implementations/tests on branch `development`.
- This pass was a GitHub source review only. No local Go tests, integration tests, fuzzing, browser tests, or live multi-tenant penetration tests were run.


## 11. Cloud Edition and Paddle dependency trace — 2026-11-14

### 11.1 Confirmed startup and configuration behavior

Reviewed on `development`:
- `apps/server/.env.example`
- `.env.example`
- `apps/server/internal/config/config.go`
- `apps/server/cmd/openpost/main.go`
- `apps/server/internal/services/billing/service.go`
- `apps/server/internal/services/entitlements/service.go`
- `apps/server/internal/services/entitlements/subscription.go`
- `apps/server/internal/api/handlers/billing.go`

Confirmed:
- `OPENPOST_EDITION=cloud` causes `Config.ValidateRuntime()` to run cloud-specific validation. The validator combines data-plane, billing, account, AI, CORS, and telemetry requirements. Cloud mode is therefore not merely a billing toggle.
- The cloud data-plane validator requires PostgreSQL and S3-compatible storage, including the configured public media base URL and S3 credentials/settings.
- The cloud billing validator explicitly requires all of the following: Paddle API key, environment, client token, webhook secret, and six configured price IDs (Founder/Team/Agency, monthly and annual). It also validates sandbox/production credential prefixes.
- The root and backend `.env.example` files describe Paddle values as cloud-only runtime configuration. The backend reads `OPENPOST_PADDLE_*` from its own process environment; bare `PADDLE_*` values are ignored and warned about.
- The main process constructs the billing service in both editions. The service initializes a Paddle SDK client only when an API key is present; its constructor does not itself perform a Paddle network request. The strict requirement comes from cloud runtime validation.
- In `main.go`, the entitlement service differs by edition: non-cloud uses `NewSelfHostedService()`; cloud uses `NewSubscriptionService(db, NewCloudBootstrapService())`.
- `NewSelfHostedService()` creates a static snapshot with no configured limits. In this implementation, an absent limit is interpreted as unlimited. Switching a public SaaS deployment to `selfhost` merely to bypass Paddle would therefore remove the intended default quota enforcement.
- `NewCloudBootstrapService()` is intentionally restrictive: one workspace is allowed, while connected accounts, publishing, media, team members, and some features are initially set to zero. It is a bootstrap state, not a usable free-beta plan.
- `NewSubscriptionService` resolves an active subscription and applies its stored entitlement snapshot. Without an eligible active/trialing subscription record, usage is denied. This is a useful basis for a free plan, but the free entitlement grant and its lifecycle are not implemented by simply omitting Paddle credentials.
- Billing handler code contains Paddle checkout, portal, purchase-choice, and webhook behavior. A Paddle-free beta must explicitly define which billing endpoints and UI surfaces are disabled or replaced; returning incidental configuration errors is not a product-ready billing-off mode.

### 11.2 Assessment

**Confirmed: Paddle is a hard configuration gate for the existing Cloud Edition, but Paddle SDK construction itself is conditional/lazy.** The current code does not provide a ready-made “Cloud Edition with billing disabled” switch.

**Do not set Social Hub to `selfhost` as a workaround.** That would bypass cloud-only validation and select unlimited self-hosted entitlements, while also changing other cloud-specific defaults and policies. It is not an acceptable SaaS beta configuration.

The safer architectural direction is to keep the cloud operating controls and make the billing provider independently configurable. For the initial beta, a first-party/free entitlement grant should feed the existing subscription/entitlement enforcement path (or a separately tested equivalent), while Paddle checkout, portal, and webhook integrations remain explicitly disabled. The plan catalogue and limit snapshots should remain data-driven so paid plans can be introduced later without rewriting usage enforcement.

### 11.3 Required design and implementation work — not yet performed

1. Introduce an explicit billing mode/provider setting, e.g. `disabled` or `paddle`, independent of `OPENPOST_EDITION`. Do not silently infer “free beta” from missing secrets.
2. Define a Social Hub free-beta plan and its quotas before enabling registrations. The quotas must cover workspaces, connected accounts, monthly scheduled/published posts, stored/uploaded media, provider write calls, team members, and feature flags.
3. Define how a free entitlement grant is created, renewed (if applicable), revoked, and migrated. It must be organization/workspace-scoped, auditable, idempotent, and independent of Paddle customer/subscription IDs.
4. Make Paddle-only endpoints and UI controls explicitly unavailable in billing-disabled mode, with stable API responses and tests. Do not leave checkout or webhook routes in a partially configured state.
5. Preserve cloud data-plane, account, security, email, legal, telemetry, X cost guardrail, and provider approval requirements unless a separate documented product decision changes them.
6. Add tests proving that the free plan enforces limits, that exceeding each quota is denied, that a user cannot transfer or reuse another tenant's free grant, and that Paddle credentials are not required in billing-disabled cloud mode.
7. Add regression tests proving that Paddle mode still validates credentials and continues to support checkout/webhooks when deliberately enabled.

### 11.4 Decision gate

No source code or runtime configuration was changed in this pass. Before implementing the separation, Social Hub's intended initial billing mode must be recorded as a product decision. The recommended default for the first beta is **Cloud Edition + Paddle disabled + an explicit free plan with enforced quotas**; this is a proposal, not a user-approved final decision.

### 11.5 Verification status

This was a GitHub source inspection on branch `development`. No local startup, Go tests, database integration tests, or live Paddle tests were run. The inspected code paths and configuration rules are confirmed from source; runtime behavior still requires CI or a working development environment.


## 12. Accepted Social Hub beta billing and usage architecture — 2026-11-14

Decision record: `docs/decisions/0002-beta-billing-and-usage.md`
Status: **Accepted**

The product decision is now explicit:
- Cloud Edition remains the SaaS operating mode.
- Paddle is optional and explicitly gated by a provider-mode flag; beta runs with Paddle disabled. Paddle credentials are validated only when Paddle is deliberately enabled.
- A first-party, versioned `beta_free` entitlement grant replaces the need for fabricated Paddle subscription records.
- Product usage must be recorded in an append-only, idempotent event ledger, with `usage_counters` retained as the fast quota projection.
- Entitlement quotas, abuse/rate-limit controls, and provider-cost budgets are separate control planes.
- Payment UI, checkout routes, Paddle.js configuration, webhook routes, and Paddle-only jobs must be absent/inert in disabled mode. The beta UI shows plan and usage instead.

### Source facts informing the decision

- `Config.ValidateRuntime()` currently invokes `missingCloudBillingConfig()` unconditionally for Cloud Edition; that function requires Paddle credentials and all six plan price IDs.
- The main process constructs the billing service and chooses `NewSubscriptionService(db, NewCloudBootstrapService())` for Cloud Edition. The self-hosted fallback has no explicit limits and must not be used as a SaaS bypass.
- `usage_counters` currently stores monthly aggregate totals by workspace/metric/period. It is a projection, not a general immutable product usage event ledger.
- The repository already has a shared database-backed rate limiter (`services/ratelimit`) and authentication rate-limit storage (`auth_rate_limits`).
- The repository also has a separate provider-cost event/reservation subsystem. Keep its X API spend controls distinct from beta plan quotas and generic usage events.
- The settings UI has a Paddle-oriented `BillingSettingsTab`, and checkout routes exist. Both frontend and backend gating are required; hiding the tab alone is not sufficient.

### Implementation status

This decision has been documented only. No runtime code, migrations, quota values, Paddle configuration, or UI behavior has been changed yet.

The following are implementation gates:
1. Explicit provider-mode configuration and startup validation tests.
2. First-party beta entitlement-grant schema/resolver/lifecycle with tenant isolation.
3. Approved numeric beta quota values before public registration.
4. Append-only usage events with database idempotency, transactional counter projection, retention and privacy rules.
5. Separate abuse-control audit and shared multi-replica enforcement.
6. Backend and frontend Paddle gating, including direct-route/API denial and no Paddle.js loading.
7. Beta-mode and Paddle-mode end-to-end regression tests.

The beta quota numbers, usage-event retention duration, and abuse-control thresholds remain product configuration decisions; they must not default to unlimited behavior.

### Verification

The decision record and this report update were committed to `development`. This was source inspection and documentation only. No code tests, migration tests, browser tests, or deployment checks were run.

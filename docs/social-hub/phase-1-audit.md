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

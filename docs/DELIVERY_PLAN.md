# AccessFlow delivery plan

Status: planning complete, awaiting approval to begin implementation  
Delivery style: vertical slices with a deployable `main` branch  
Estimated focused effort: 12–14 weeks for one developer; longer if performed part-time or if cloud/account prerequisites are delayed

## 1. Preconditions and decisions

### Already decided

- Product: outpatient scheduling, non-attendance intervention, and waitlist recovery.
- Public portfolio release uses synthetic data only.
- Stack: .NET 10, ASP.NET Core, React/TypeScript, PostgreSQL, EF Core, ML.NET, containers.
- Browser auth: Identity cookies, sessions, forgot/reset password, 2FA, and antiforgery.
- Architecture: modular monolith, inference sidecar, scheduled worker job.
- Free demo: Azure Container Apps, Neon, public GHCR, Brevo, Turnstile.
- GitHub: public repository is recommended to keep standard Actions and public-code security features free.

### Confirm at implementation kickoff

- Final repository name (`accessflow-clinic` is the default).
- GitHub owner/account or organization under which the public repository will be created.
- Permission to make the repository public.
- Azure subscription to use and agreement that free grants are allowances, not a hard spending cap.
- Neon, Brevo, and Cloudflare accounts or approval for the user to create/connect them when provisioning requires interactive login.
- Whether a custom domain will be supplied later; it is not required for launch.

### Local prerequisite gap discovered during planning

As of 2026-11-24, the workspace exposes Git and GitHub CLI, but does not expose `dotnet`, `npm`, Docker CLI, or Azure CLI on `PATH`. Bootstrap will require installation or discovery of bundled runtimes. Any installation/download requiring elevated permission or network access will be requested at implementation time.

## 2. Delivery gates

| Gate | Required outcome |
|---|---|
| G0 — Plan approval | User accepts product boundary, public/synthetic posture, and free-tier tradeoffs. |
| G1 — Repository ready | Separate public repository, branch rules, CI skeleton, local prerequisites, and zero secrets committed. |
| G2 — Secure foundation | Registration/login/reset/2FA/session/tenant flows pass integration and security tests. |
| G3 — Scheduling integrity | Provider/patient conflicts and concurrent booking demonstrate exactly-one correctness. |
| G4 — Recovery workflow | Cancellation-to-waitlist-offer-to-booking works end to end, including retries/expiry/races. |
| G5 — ML validity | Baseline/candidate, leakage audit, chronological evaluation, model card, fallbacks, and version traceability. |
| G6 — Release candidate | Accessibility, security, load, migration, backup/restore, and E2E critical flows pass. |
| G7 — Free deployment | IaC provisions approved free-profile resources, smoke tests pass, usage guard is active. |
| G8 — Portfolio release | Live synthetic demo, documentation, video/screenshots, tagged release, and resume-ready evidence. |

Implementation must not skip a failed gate to preserve a target date.

## 3. Milestones

### Milestone 0 — Validation assets and bootstrap readiness

Target: days 1–3.

Deliverables:

- Confirm name, public repository destination, and free-tier accounts.
- Install/locate .NET 10 SDK, Node package manager, Docker, Azure CLI, and required browser tooling.
- Create a short clinic-interview guide, workflow diagram, and product assumptions log.
- Establish no-real-data policy and synthetic demo disclaimer.
- Create implementation issues from the approved plan.

Exit criteria:

- Toolchain versions recorded.
- `docker compose` and minimal .NET/React smoke projects run locally.
- No code has been added to the unrelated parent repository.

### Milestone 1 — Repository and engineering foundation

Target: days 4–7.

Deliverables:

- Create separate GitHub repository and initial protected `main` branch.
- Solution/project structure, centralized NuGet packages, frontend lock file, editor settings, license, security/contribution files.
- Local Compose stack with PostgreSQL and Mailpit.
- CI for format, build, unit tests, architecture tests, dependency restore, and container build.
- Configuration validation, Problem Details, OpenAPI, correlation ids, structured logging, health endpoints.
- Initial Bicep skeleton and environment configuration contract.

Exit criteria:

- Clean clone can build and test from documented commands.
- CI passes on Windows/Linux assumptions as selected.
- No secret-like values in history.

### Milestone 2 — Identity, sessions, and tenancy

Target: week 2.

Vertical slices:

- Patient registration, bot validation abstraction, email verification, login/logout.
- Forgot/reset/change password.
- Secure cookie and antiforgery configuration.
- Staff invitation and TOTP enrollment/recovery.
- Server-side session registry, device list, revoke one/all.
- Clinic creation, membership, roles, switch context, last-owner protection.
- Data Protection key persistence and restart test.
- Authentication/authorization audit events.

Tests:

- Token expiry/replay, non-enumeration, lockout, CSRF, open redirect, cookie flags.
- Role downgrade/session invalidation.
- Cross-tenant and stale-membership attempts.
- Container restart retains valid protected state according to policy.

Exit: G2.

### Milestone 3 — Clinic directory and availability

Target: week 3.

Vertical slices:

- Locations/timezones/hours/closures.
- Providers and visit types.
- Recurring schedule templates and exceptions.
- Slot-generation/query service.
- Configuration impact preview and archive behavior.
- Patient scheduling profile and communication consent.

Tests:

- DST gaps/overlaps, leap day, location timezone changes.
- Provider/visit eligibility and buffer calculations.
- Archive/reference-history behavior.

### Milestone 4 — Appointment lifecycle and concurrency

Target: weeks 4–5.

Vertical slices:

- Slot search and accessible calendar UI.
- Slot hold and expiry.
- Patient/staff booking.
- View, confirm, cancel, and atomic reschedule.
- Check-in, complete, and no-show recording/correction.
- Status history, audit, outbox, idempotency, and ETags.
- Provider/patient overlap constraints.

Tests:

- Fifty concurrent booking attempts for the same slot yield one success.
- Double-click/network retry creates one appointment.
- Atomic reschedule failure preserves original.
- Cancellation/check-in/completion races converge safely.
- Playwright covers the primary patient and staff paths.

Exit: G3.

### Milestone 5 — Waitlist and recovery engine

Target: weeks 6–7.

Vertical slices:

- Waitlist join/edit/pause/withdraw.
- Eligibility policy and explainable exclusions.
- Recovery workflow after confirmed cancellation.
- Sequential offer creation, expiring token, accept/decline/expiry.
- Exactly-one acceptance and supersession.
- Staff manual assignment and override audit.
- Failed recovery task and operational dashboard slice.

Tests:

- Accept/expire/manual-assignment races.
- Existing-appointment and preference conflicts.
- Replayed/shared offer link.
- Frequency/quiet-hour suppression.
- End-to-end cancellation to recovered slot.

Exit: G4.

### Milestone 6 — Communications and worker reliability

Target: week 8.

Vertical slices:

- Versioned email templates and Mailpit local delivery.
- Brevo provider adapter for demo.
- Simulated SMS outbox.
- PostgreSQL outbox/inbox, worker leases, retries, jitter, dead letter.
- Delivery webhook validation/deduplication.
- Bounce/complaint suppression.
- Notification quota and abuse controls.
- Scheduled offer expiry and reminder processing.

Tests:

- Crash after provider send/before acknowledgement does not create a duplicate logical email.
- Duplicate/out-of-order webhooks converge.
- Opt-out after queue but before send is honored.
- Quota exhaustion creates visible fallback task.

### Milestone 7 — ML training, inference, and MLOps

Target: weeks 9–10.

Vertical slices:

- Versioned Synthea/scheduling-data generator.
- Shared feature schema and leakage audit.
- Clinic-rate and logistic baselines.
- LightGBM candidate and chronological evaluation.
- Calibration and subgroup/drift report.
- Model card and artifact metadata/checksum.
- Strict inference service and app client with timeout/fallback.
- Immutable prediction history, current prediction, intervention/outcome records.
- GHCR inference image and manual model promotion workflow.

Tests:

- Feature parity between training and inference.
- No future information in sampled historical features.
- Invalid schema/NaN/corrupt model fails closed to standard policy.
- App remains usable when inference is down.
- Model version is traceable from prediction to image/artifact.

Exit: G5.

### Milestone 8 — Reporting, CSV, and mock FHIR

Target: week 11.

Vertical slices:

- Operational dashboard with metric definitions/completeness.
- Model-monitoring dashboard safe for synthetic data.
- Audit viewer and dead-letter operations.
- CSV upload, preview, mapping, validation, resumable apply, report/export.
- Mock FHIR adapter for supported resources and sync conflict workflow.
- Synthetic tenant reset and seed restore.

Tests:

- CSV formula injection, malformed encoding, duplicate ids, ambiguous time.
- Partial import retry and cross-tenant crafted identifiers.
- Duplicate/out-of-order external events and reconciliation.

### Milestone 9 — Professional hardening

Target: week 12.

Workstreams:

- OWASP ASVS Level 2 checklist and threat-model review.
- CSP/security headers, rate limits, upload quotas, log redaction.
- WCAG 2.2 AA audit with automated and keyboard/screen-reader spot checks.
- Responsive UI, empty/loading/error/offline/degraded states.
- PostgreSQL index/query review and load tests.
- Migration from prior snapshot and clean database.
- Backup/export and documented restore exercise.
- Container hardening, SBOM, image scan, action pinning.
- Runbooks and incident/failure drills.

Exit: G6.

### Milestone 10 — Free-tier infrastructure and deployment

Target: week 13.

Deliverables:

- Neon demo database.
- Brevo sender/API configuration and webhook secret.
- Turnstile widget and server validation.
- Public GHCR images.
- Azure Container Apps environment, one app with API/web + inference sidecar, and scheduled worker job.
- Scale 0..1, resource limits, generated HTTPS hostname, secrets, sampled telemetry.
- GitHub OIDC deployment, guarded migrations, smoke tests, rollback.
- Usage dashboard/alerts, application quotas, registration/showcase switch, kill-switch runbook.

Exit: G7.

### Milestone 11 — Portfolio release

Target: week 14.

Deliverables:

- Seeded synthetic clinic and safe demo accounts.
- Polished README with live URL, screenshots, architecture, local run, limitations, model results, and test/deployment badges.
- Architecture diagram, ERD, threat model, model card, OpenAPI, and runbooks.
- Two-to-four-minute demonstration video.
- GitHub release with immutable image digests and changelog.
- Resume bullets and interview talking points based on measured results.
- Interview/pilot outreach package, without claiming product-market fit.

Exit: G8.

## 4. Backlog structure

Create GitHub milestones matching the milestones above and label issues with:

- `area:identity`
- `area:tenancy`
- `area:scheduling`
- `area:waitlist`
- `area:communications`
- `area:ml`
- `area:integration`
- `area:reporting`
- `area:security`
- `area:accessibility`
- `area:infrastructure`
- `type:feature`
- `type:bug`
- `type:test`
- `type:documentation`
- `priority:p0` through `priority:p3`
- `risk:security`, `risk:data`, `risk:cost`, or `risk:ml`

Each feature issue must include:

- User/actor and desired outcome.
- In-scope and out-of-scope behavior.
- Authorization policy.
- State transition and idempotency requirement.
- Audit and outbox effects.
- Happy-path and edge-case acceptance criteria.
- Unit/integration/E2E/accessibility/security test expectations.
- Observability and degraded behavior.

## 5. Testing strategy

### Test pyramid and suites

| Suite | Purpose | Tooling direction |
|---|---|---|
| Domain unit | State machines, policies, scheduling math | xUnit + assertion library |
| Application unit | Commands, authorization decisions, fallbacks | xUnit + fakes only at boundaries |
| Architecture | Module/project/reference rules | NetArchTest or equivalent |
| Integration | Real PostgreSQL, EF mappings, transactions, auth | Testcontainers + WebApplicationFactory |
| Contract | API schemas, inference contract, provider webhooks | OpenAPI snapshot/compatibility + fixtures |
| Frontend unit | Components, forms, accessibility behavior | Vitest + Testing Library + axe |
| End to end | Critical patient/staff journeys | Playwright |
| Concurrency | Booking/offer/status races | Integration harness against PostgreSQL |
| Load | Slot search, login protection, booking contention | k6 or NBomber |
| Security | ASVS cases, dependency/code/container scan | CodeQL, Dependabot, Trivy, targeted tests |
| ML | Leakage, parity, metrics, artifact/schema | xUnit plus trainer evaluation harness |
| Infrastructure | Bicep lint/what-if and smoke | Bicep/Azure CLI + scripted probes |

### Mandatory critical E2E journeys

1. Register, verify, log in, log out, reset password, and revoke another session.
2. Admin invites staff; staff enrolls 2FA; role downgrade takes effect immediately.
3. Admin configures provider/visit type/schedule; patient books and confirms.
4. Patient atomically reschedules; old link no longer mutates state.
5. Patient cancels; waitlist offer is generated; eligible patient accepts; other offer is superseded.
6. Inference outage still permits booking and standard confirmation.
7. Cross-tenant crafted request is denied and audited.
8. CSV import previews, rejects errors, applies valid rows idempotently, and exports report.

### Browser and accessibility matrix

- Current Chrome/Edge and Firefox desktop.
- Mobile viewport and touch interaction.
- Keyboard-only complete critical flows.
- Screen-reader spot checks on login, calendar, booking, waitlist offer, and error summary.
- Reduced motion, 200% zoom, high contrast, and focus visibility.

## 6. Definition of done

A feature is done only when:

- Acceptance criteria and negative paths pass.
- Authorization and tenant isolation are explicit and tested.
- State changes are idempotent/concurrency-safe where applicable.
- Audit and outbox effects are correct.
- API/OpenAPI and UI behavior are documented.
- Unit and integration tests exist; E2E is added for critical paths.
- Accessibility states are implemented.
- Logs/metrics contain useful, non-sensitive signals.
- Migrations are reviewed and reversible through forward-fix/restore strategy.
- No new high/critical dependency or container vulnerability is accepted without documented disposition.
- CI is green and branch review requirements are satisfied.

The application release is done only when:

- All G0–G8 gates pass.
- Live deployment and rollback have been exercised.
- Free-tier assumptions have been reverified and no unintended paid resource exists.
- Public source/history contains no secret or real patient information.
- Demo limitations and synthetic nature are prominent.
- Model card distinguishes synthetic validation from real-world effectiveness.
- The final demo can be reproduced from the tagged source.

## 7. Risk register

| Risk | Likelihood/impact | Mitigation |
|---|---|---|
| Scope expands into an EHR | High/high | Enforce explicit out-of-scope list and vertical milestone gates. |
| Synthetic model looks impressive but has no real validity | High/high | Prominent model card, baseline comparison, no clinical claims, pursue controlled validation separately. |
| Prediction does not improve outcomes | Medium/high | Measure recovered capacity and intervention response; model never presented as the whole product. |
| Cross-tenant data leak | Low/critical | Defense in depth, resource authorization, database constraints/optional RLS, broad IDOR suite. |
| Double booking under race | Medium/high | Database exclusion/unique constraints, locks, idempotency, concurrency tests. |
| Free-tier costs or policy changes | Medium/medium | Recheck before deploy, scale limits, quotas, no paid dependencies, usage kill switch, local fallback. |
| Public registration abuses email/compute | Medium/high | Turnstile, rate limits, quotas, invite/showcase mode, bounce suppression. |
| Cold starts harm demo | High/medium | Loading state, health warm-up before interviews, minimal images, optional showcase data cached in UI. |
| No custom email domain reduces deliverability | Medium/medium | Brevo replacement sender for synthetic demo; in-app status; add owned domain only if supplied. |
| External EHR integration becomes unbounded | High/high | Mock adapter first, FHIR resource allowlist, source-of-truth policy, no vendor-specific production integration in MVP. |
| ML/app feature schema drifts | Medium/high | Shared contracts, schema version, contract tests, incompatible-model fallback. |
| Protected proxies create unfair contact | Medium/high | Feature allowlist, fairness audit, contact caps, no access-denying actions, human policy control. |
| Missing local tools delays start | High/medium | Bootstrap milestone and approval-based installation before app coding. |
| Public repository leaks secrets | Medium/critical | OIDC, secret scanning/push protection, ignored config, history review before first push. |

## 8. Professional portfolio evidence

The final repository should make these claims demonstrable rather than merely stated:

- Secure account lifecycle with cookie/session management and staff 2FA.
- Multi-tenant authorization with adversarial tests.
- Correct transactional booking and waitlist recovery under concurrency.
- Outbox/inbox reliability and provider webhook handling.
- FHIR-shaped interoperability boundary without claiming to be an EHR.
- Reproducible ML training, calibration, model card, deployment, fallback, and monitoring.
- Public cloud deployment through IaC and OIDC-based CI/CD.
- Security, accessibility, observability, and runbooks.
- Measured synthetic results clearly separated from real-world claims.

Suggested eventual resume bullet, to be rewritten with actual measured values:

> Built and deployed a multi-tenant clinic-access platform with ASP.NET Core 10, React, PostgreSQL, and ML.NET; implemented secure Identity sessions/2FA, concurrency-safe scheduling and waitlist recovery, versioned no-show inference, FHIR-shaped integration, containerized Azure deployment, and automated security/test pipelines.

## 9. Immediate next action after plan approval

Do not begin feature code first. The implementation kickoff sequence is:

1. Confirm repository visibility/name and GitHub owner.
2. Resolve the missing local toolchain.
3. Create the separate repository with planning documents as the first commit.
4. Enable repository security/rules and baseline CI.
5. Provision no cloud resources until local Identity and booking foundations pass their gates.


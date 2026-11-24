# AccessFlow architecture, ML, GitHub, and deployment plan

Status: proposed  
Primary constraint: full public demonstration at $0 within current free-tier allowances  
Secondary constraint: retain a credible migration path to a paid real-world deployment

## 1. Architectural decisions

### ADR-001 — Modern .NET, not legacy .NET Framework

Use .NET 10 LTS and ASP.NET Core. .NET 10 is an LTS release supported through November 2028. [Microsoft support policy](https://learn.microsoft.com/en-us/dotnet/core/releases-and-support)

### ADR-002 — Modular monolith first

The business backend is one deployable ASP.NET Core application with enforced module boundaries. A portfolio project does not benefit from manufacturing many network services. Separate processes exist only where there is a lifecycle reason:

- ML inference has an independently versioned container/artifact.
- Finite background work uses a worker image and scheduled/on-demand job.

### ADR-003 — Same-origin browser application

The ASP.NET Core container serves the compiled React application and `/api/v1/*` from the same HTTPS origin. This simplifies secure cookie authentication, CSRF protection, CSP, CORS, and free hosting. The API remains cleanly separated in source and can later support another client.

### ADR-004 — Cookie authentication for the browser

Use ASP.NET Core Identity with encrypted, HTTP-only, secure cookies and antiforgery protection. Do not store bearer tokens in browser local storage. Session registry data is server-side in PostgreSQL; the Identity cookie remains the authentication ticket.

### ADR-005 — PostgreSQL as the system of record

Use PostgreSQL and EF Core for transactional data, jobs/outbox, session registry, audit metadata, idempotency, and data-protection key records. Do not add Redis to the $0 profile merely to make the architecture look larger. Add it later only for measured caching or distributed-rate-limit needs.

### ADR-006 — Transactional outbox, no paid message broker in demo

Domain changes and outbox messages commit in one PostgreSQL transaction. A worker claims jobs with `FOR UPDATE SKIP LOCKED`, uses leases, and records deterministic idempotency keys. The interface permits Azure Service Bus or another broker in a paid profile without rewriting domain code.

### ADR-007 — Model sidecar in the free deployment

The inference process is a separate OCI image but is deployed beside the API container in one Azure Container App. The containers share lifecycle and local networking. Azure Container Apps supports multiple containers/sidecars in one app. [Microsoft documentation](https://learn.microsoft.com/en-us/azure/container-apps/containers)

This is a free-tier optimization, not the production reference. The paid profile deploys inference as an internal container app with independent scaling and rollout.

### ADR-008 — Synthetic-only public environment

The public deployment contains no real patient data, no real clinical integration, and no HIPAA-compliance claim. Production safeguards are designed and demonstrated where possible, but legal/organizational compliance and appropriate vendors are prerequisites for real data.

## 2. System context

```text
Patient / Clinic Staff
          |
          v
Azure Container App (public HTTPS, scale 0..1)
  +-----------------------------------------------+
  | accessflow-app                                |
  | React static UI + ASP.NET Core API            |
  | Identity + modules + OpenAPI + health         |
  |                                               |
  | localhost HTTP                                |
  |       v                                       |
  | accessflow-inference sidecar                  |
  | ML.NET model and strict prediction contract   |
  +-----------------------------------------------+
          |
          +------ Neon PostgreSQL (TLS)
          +------ Brevo transactional email
          +------ Cloudflare Turnstile verification

Azure Container Apps Job (scheduled/on-demand)
  accessflow-worker
    - outbox dispatch
    - offer expiry
    - scheduled rescoring
    - retention/demo reset

GitHub
  - source and documentation
  - Actions CI/CD and model pipeline
  - public GHCR application, worker, and inference images
```

## 3. Planned repository structure

```text
accessflow-clinic/
  .github/
    workflows/
      ci.yml
      codeql.yml
      container-scan.yml
      deploy-demo.yml
      train-model.yml
      smoke-test.yml
    ISSUE_TEMPLATE/
    pull_request_template.md
    dependabot.yml
  docs/
    PRODUCT_PLAN.md
    USE_CASES_AND_EDGE_CASES.md
    ARCHITECTURE_ML_DEPLOYMENT.md
    DELIVERY_PLAN.md
    threat-model.md
    model-card.md
    api-guidelines.md
    runbooks/
  src/
    AccessFlow.Domain/
    AccessFlow.Application/
    AccessFlow.Infrastructure/
    AccessFlow.Contracts/
    AccessFlow.Api/
    AccessFlow.Worker/
    AccessFlow.ML.Contracts/
    AccessFlow.ML.Inference/
    AccessFlow.ML.Training/
    accessflow-web/
  tests/
    AccessFlow.Domain.Tests/
    AccessFlow.Application.Tests/
    AccessFlow.IntegrationTests/
    AccessFlow.ArchitectureTests/
    AccessFlow.ML.Tests/
    accessflow-web-tests/
    AccessFlow.E2E/
    AccessFlow.LoadTests/
  tools/
    AccessFlow.SyntheticData/
  infra/
    bicep/
    docker/
  model/
    schemas/
    baselines/
  .editorconfig
  .gitattributes
  .gitignore
  AccessFlow.slnx
  Directory.Build.props
  Directory.Packages.props
  global.json
  compose.yaml
  LICENSE
  README.md
  SECURITY.md
  CONTRIBUTING.md
```

The implementation should be a fresh repository named `accessflow-clinic`, not committed into the unrelated parent repository currently present in the workspace.

## 4. Backend modules

| Module | Owns | May publish |
|---|---|---|
| Identity | Application users, credentials, sessions, 2FA, security events | UserVerified, SessionRevoked |
| Tenancy | Clinics, memberships, invitations, roles | MembershipChanged, ClinicConfigured |
| Directory | Locations, providers, visit types, patient scheduling profiles | ProviderChanged, VisitTypeChanged |
| Scheduling | Availability, holds, slots, appointments, status history | AppointmentBooked/Changed/Cancelled/Completed |
| Waitlist | Entries, matching policies, offers, recovery workflows | OfferCreated/Accepted/Expired |
| Communications | Consent, templates, notifications, delivery state | NotificationDelivered/Failed, PatientResponded |
| Risk | Features, predictions, interventions, model registry metadata | RiskScored, InterventionRecorded |
| Integrations | CSV/FHIR imports, inbox events, reconciliation | ExternalChangeReceived, SyncConflictDetected |
| Reporting | Read models and metric definitions | No business writes |
| Audit | Append-only audit events and review queries | No outward domain behavior |

Rules:

- Each module owns its EF mappings and write paths.
- Cross-module behavior uses application commands/events, not direct table mutation.
- Reads may use purpose-built projections while honoring tenant filters.
- Architecture tests prevent forbidden project/module references.
- No business rule exists only in React or a controller.

## 5. Key data model

All tenant-owned tables contain `clinic_id`. Public identifiers are UUIDv7/opaque values; database surrogate choices must not leak counts.

### Identity and tenancy

- `application_user`
- `user_session`
- `security_event`
- `clinic`
- `clinic_membership`
- `clinic_invitation`
- `role_assignment` or policy-backed membership role
- `data_protection_key`

### Directory and scheduling

- `location`
- `provider`
- `visit_type`
- `provider_visit_type`
- `schedule_template`
- `schedule_exception`
- `slot_hold`
- `appointment`
- `appointment_status_history`
- `patient_scheduling_profile`
- `communication_consent`

### Waitlist and communication

- `waitlist_entry`
- `recovery_workflow`
- `slot_offer`
- `notification`
- `notification_delivery_event`
- `template_version`
- `outbox_message`
- `dead_letter`

### ML, integrations, and audit

- `model_version`
- `feature_schema_version`
- `appointment_prediction`
- `intervention`
- `outcome_label_history`
- `import_job`
- `import_row_issue`
- `external_connection`
- `external_entity_map`
- `inbox_message`
- `sync_conflict`
- `idempotency_record`
- `audit_event`

### Critical database constraints

- Unique normalized user email according to Identity policy.
- Unique active membership per user/clinic.
- Last-owner protection enforced transactionally.
- Unique source/external-id mapping per clinic.
- Unique active appointment for a source external id.
- PostgreSQL exclusion constraint preventing overlapping provider intervals for active appointments, including buffers.
- Patient-overlap rule enforced transactionally.
- Unique accepted offer per slot/recovery workflow.
- Unique idempotency key within actor/endpoint scope.
- Unique inbox/outbox event id.
- Check constraints for interval ordering, probability ranges, and allowed terminal transitions where feasible.

Soft deletion is not a default. Use explicit active/archived states for reference data and retention-aware pseudonymization for accounts. Audit and status history remain immutable through normal APIs.

## 6. API and application conventions

- REST under `/api/v1` with OpenAPI generation.
- `application/problem+json` using current Problem Details conventions.
- Command endpoints accept `Idempotency-Key` when network retry could duplicate a business action.
- Optimistic concurrency uses ETags/version tokens; conflicts return `409` or `412` with refresh guidance.
- Cursor pagination for changing operational collections; bounded offset paging only for stable admin lists.
- UTC timestamps in API using ISO 8601 with offset; IANA timezone identifiers for scheduling rules.
- Validation returns stable machine-readable error codes plus accessible user messages.
- Correlation/trace id returned in response headers and problems.
- No tenant id is trusted without current server-side membership resolution.
- Cookie-authenticated commands require antiforgery validation.
- External webhook routes use provider signature validation, timestamp tolerance, inbox deduplication, and no browser cookies.
- OpenAPI hides internal worker/inference routes from the public contract.

## 7. Security architecture

Use OWASP ASVS 5.0 Level 2 as the public-web verification baseline. [OWASP ASVS](https://asvs.owasp.org/)

### Authentication and sessions

- ASP.NET Core Identity password hashing and token providers.
- Secure, HTTP-only, host-only cookie with an application-specific name.
- `SameSite=Lax` by default, strict CORS, HTTPS only, short idle timeout, bounded absolute lifetime.
- Antiforgery header/token on mutations; no state-changing GET routes.
- Staff TOTP 2FA required; recovery codes hashed and one-time.
- Session registry stores session hash, user, creation, last seen, expiry, revoked time/reason, security-stamp version, and coarse client metadata.
- Password, email, privilege, and 2FA recovery events rotate the security stamp and sessions.
- Turnstile plus server-side rate limiting on registration, login after threshold, reset, invitation acceptance, and anonymous token forms. Turnstile’s free plan allows unlimited challenges, but server-side token validation is mandatory. [Plan](https://developers.cloudflare.com/turnstile/plans/) [Validation](https://developers.cloudflare.com/turnstile/get-started/server-side-validation/)

### Authorization and isolation

- Policy-based authorization: `ClinicOwner`, `ClinicAdmin`, `Scheduler`, `Clinician`, `Auditor`, `PatientSelf`.
- Resource-based checks for appointment/patient ownership.
- Tenant context resolved after authentication from active membership.
- EF global filters are a convenience, not the only control; commands also require tenant-scoped keys.
- Integration tests attempt IDOR and cross-tenant access for every resource family.
- Optional PostgreSQL RLS becomes a defense-in-depth milestone after connection-pool context behavior is tested.

### Keys and secrets

- Local secrets use .NET user-secrets or ignored `.env` files with documented placeholders.
- GitHub Actions uses Azure workload identity/OIDC, not a long-lived publish profile.
- Azure secrets reside in Container Apps secret configuration for the demo.
- ASP.NET Data Protection keys persist through EF Core and are protected by an X.509 certificate supplied as a deployment secret. Losing container-local keys would invalidate cookies, so ephemeral key storage is forbidden. Microsoft recommends persistent or external key storage for containers. [Data Protection guidance](https://learn.microsoft.com/en-us/aspnet/core/security/data-protection/configuration/overview)
- Production profile moves secrets/certificates to Key Vault and uses managed identity.

### Web hardening

- Strict Content Security Policy with nonces/hashes and no unsafe inline script.
- HSTS, `X-Content-Type-Options`, appropriate referrer and permissions policies, frame denial, and HTTPS redirects.
- Output encoding, parameterized EF queries, upload size/type controls, and CSV formula-injection escaping.
- Rate limits partitioned by authenticated user or anonymous network/account key. ASP.NET Core provides fixed, sliding, token-bucket, and concurrency limiters. [Microsoft rate-limiting guidance](https://learn.microsoft.com/aspnet/core/performance/rate-limit?view=aspnetcore-10.0)
- Dependency pinning, Dependabot, CodeQL, secret scanning, push protection, and container vulnerability scanning.
- Logs use allowlisted structured fields and a redaction processor. No raw email tokens, cookies, credentials, full message bodies, or clinical text.

### Threat-model priorities

- Cross-tenant data access.
- Appointment/offer race leading to double booking.
- Account takeover and session replay.
- Forged or replayed public action links.
- Webhook spoofing/replay.
- Notification abuse and quota exhaustion.
- Training/feature leakage and malicious model artifact.
- Public-demo PHI entry.
- Supply-chain compromise in public CI/CD.

## 8. ML design

### 8.1 Model inventory

| Model | Initial implementation | Promotion requirement | Output/use |
|---|---|---|---|
| Appointment outcome baseline | Multinomial logistic regression or calibrated one-vs-rest logistic models in ML.NET | Always trained and reported | Baseline and fallback probabilities |
| Appointment outcome candidate | ML.NET LightGBM multiclass, or two calibrated binary LightGBM models if calibration is materially better | Beat baseline on chronological data and pass calibration/fairness gates | `attend`, `late_cancel`, `no_show` probabilities |
| Intervention response | Logistic regression | Deferred until enough actual outreach labels exist | Probability of response per permitted channel/action |
| Waitlist ranking | Deterministic policy initially; LightGBM ranking later | Deferred until sufficient offer impressions/acceptances | Rank already eligible candidates |
| Time-to-cancellation | Survival/time-to-event research model | Post-MVP only | Probability of cancellation within windows |

No LLM is required for core behavior. If reminder drafting is explored later, output is constrained to approved templates and can never contain medical advice or determine scheduling priority.

### 8.2 Feature policy

Feature groups known at scoring time:

- Appointment lead time, local day/time, visit type, location, new/follow-up indicator.
- Prior attended/no-show/late-cancel counts and recency, computed only from outcomes available before score time.
- Confirmation and reschedule history.
- Number of existing future appointments and recent schedule changes.
- Clinic/provider operational aggregates with minimum-support thresholds.
- Reminder delivery/response state available at rescore time.

Prohibited from decision features in MVP:

- Race, ethnicity, sex, religion, disability, diagnosis, insurance status, income proxies, clinical notes, or protected demographic attributes.
- Any future event or outcome unavailable at the prediction timestamp.
- Raw patient identity or high-cardinality identifier that memorizes individuals.

Where legally and ethically supportable in a controlled evaluation, protected attributes may be isolated from training and used only to audit group performance. They are never exposed to the inference endpoint or UI.

### 8.3 Synthetic data

- Generate patient-like records with Synthea FHIR/CSV data; Synthea states its synthetic data can be used without real-patient privacy restrictions. [Synthea](https://synthetichealth.github.io/synthea/)
- Generate scheduling events with a versioned simulator whose parameters create clinic, seasonality, lead-time, confirmation, and attendance patterns.
- Mark every record with `is_synthetic=true` and a generator version.
- Include deliberate cold-start clinics, sparse patients, drift periods, class imbalance, timezone changes, and label corrections.
- Synthetic performance demonstrates engineering only, not clinical validity or real-world effectiveness.

### 8.4 Training and evaluation

1. Validate schema and dataset provenance.
2. Build features in a shared, versioned contract used by training and online inference.
3. Run leakage tests that reconstruct what was known at each prediction timestamp.
4. Split chronologically by appointment time, with a final untouched holdout period.
5. Train clinic-rate and logistic baselines.
6. Train candidate LightGBM configurations with bounded tuning.
7. Evaluate class prevalence, confusion matrices, PR-AUC, ROC-AUC as secondary, log loss/Brier score, calibration curves/error, and top-risk recall/precision.
8. Evaluate clinic/time slices and ethically approved demographic audit slices.
9. Evaluate action-oriented metrics through simulation: calls per recovered appointment and recoverable high-risk recall.
10. Produce a model card and signed/attested artifact metadata.

Promotion requires documented thresholds relative to baseline, not an arbitrary universal accuracy target. Prospective results are reported separately from retrospective results.

### 8.5 Inference contract

Request contains:

- Schema version, clinic-scoped non-identifying appointment key, score timestamp, and allowlisted numeric/categorical features.

Response contains:

- Model name/version, feature-schema version, `attend`, `lateCancel`, and `noShow` probabilities, confidence status, model-generated-at timestamp, and allowlisted descriptive signals.

The API validates finite probabilities, range, class sum tolerance, version compatibility, and latency. Invalid responses are discarded and standard policy is used.

### 8.6 Model packaging and deployment

- Training output includes ML.NET artifact, schema JSON, metrics JSON, model card, checksum, code commit, and generator/dataset version.
- The artifact is copied into an immutable `accessflow-inference:<model-version>` image during CI.
- Image is pushed to public GHCR and deployed by digest.
- API performs inference contract/health checks before receiving traffic.
- Previous Container Apps revision/image digest remains available for rollback.
- App-to-model calls are local to the sidecar in free mode; no inference endpoint is public.
- A model deployment cannot migrate the business database.

### 8.7 Monitoring

- Inference count, latency, error/fallback rate, active model version, and schema mismatch.
- Prediction distributions and feature missingness without patient identity.
- Delayed outcome metrics by prediction date, clinic, and model version.
- Calibration and drift reports only after sufficient labels.
- Contact/recovery outcomes to detect a model that predicts risk but does not improve operations.

## 9. Background processing

`AccessFlow.Worker` supports modes selected by command:

- `dispatch-outbox`
- `expire-offers`
- `rescore-upcoming`
- `import-batch`
- `compute-metrics`
- `reset-demo`
- `retention`

Free deployment uses a scheduled Azure Container Apps Job at a conservative interval (initially 15 minutes) plus on-demand executions for imports. Container Apps Jobs are finite tasks that can run on a schedule and are billed only while executing within the same consumption allowance. [Microsoft jobs documentation](https://learn.microsoft.com/en-us/azure/container-apps/jobs)

Workers are at-least-once. Every handler must be idempotent, use leases, and tolerate termination between external effect and local acknowledgement.

## 10. Free-tier deployment profile

Free tiers change. Limits below are planning assumptions verified on 2026-11-24 and must be rechecked immediately before provisioning.

| Capability | Service | Current free assumption | Guardrail |
|---|---|---|---|
| Source/CI | Public GitHub repository | Standard GitHub-hosted Actions are free for public repositories. [GitHub](https://docs.github.com/en/actions/concepts/billing-and-usage) | Public synthetic code/data only; pin actions and minimize artifact retention. |
| Images | Public GitHub Container Registry | Public packages are free; container image storage/bandwidth is currently free subject to policy change. [GitHub](https://docs.github.com/en/enterprise-cloud%40latest/billing/concepts/product-billing/github-packages) | Public images, multi-stage builds, prune tags, deploy by digest. |
| App/model/worker compute | Azure Container Apps Consumption | Monthly free grant: first 180,000 vCPU-seconds, 360,000 GiB-seconds, and 2 million requests per subscription. [Microsoft](https://learn.microsoft.com/azure/container-apps/billing) | Scale 0..1, minimal CPU/RAM, no VNet/private endpoint, conservative jobs, usage alerts/kill switch. |
| PostgreSQL | Neon Free | As of Nov. 2026, 1 GB storage and 100 CU-hours per project are stated for the free plan. [Neon](https://neon.com/blog/how-to-make-the-most-of-neons-free-plan) | One demo project, pooled connections, retention/reset, synthetic data only. |
| Transactional email | Brevo Free | 300 emails/day and no credit card/time limit; when no custom domain is authenticated, Brevo may replace a free sender address with its compliant sender domain. [Free plan](https://help.brevo.com/hc/en-us/articles/208589409-About-Brevo-s-pricing-plans) [Sender behavior](https://help.brevo.com/hc/en-us/articles/14925263522578-Comply-with-Gmail-Yahoo-and-Microsoft-s-requirements-for-email-senders) | Synthetic demo only, Brevo branding/replacement sender accepted, per-user quotas, Turnstile, bounce suppression, no SMS credits. |
| Bot defense | Cloudflare Turnstile Free | Up to 20 widgets and unlimited challenges. [Cloudflare](https://developers.cloudflare.com/turnstile/plans/) | Separate dev/prod widgets; mandatory server verification and hostname checks. |
| Telemetry | Azure Monitor/Application Insights | Default pay-as-you-go Log Analytics currently includes a 5 GB monthly free allowance per billing account. [Microsoft](https://learn.microsoft.com/da-dk/azure/azure-monitor/app/statsbeat) | Aggressive sampling, low daily cap, no bodies/PHI, exportable console logs. |
| SMS | Demo outbox | No honest free production SMS dependency | Never call a carrier; show simulated messages in authorized UI. |
| Domain/TLS | Azure-generated hostname | Managed platform hostname/certificate | Custom domain is optional and outside the $0 promise. |

### Important cost truth

Azure’s monthly free grant is an allowance, not a hard global spending cap. Budget alerts can lag. “Deploy for free” therefore means the normal portfolio workload is engineered to stay within the allowance, paid features are not provisioned, maximum scaling is bounded, and a usage kill switch exists. It cannot be represented as mathematically impossible to incur a charge if credentials are abused or limits/pricing change.

### Resource sizing target

- Container App: min replicas `0`, max replicas `1`.
- Main app: start at 0.25 vCPU / 0.5 GiB if platform/runtime behavior is acceptable.
- Inference sidecar: start at 0.25 vCPU / 0.5 GiB.
- Worker job: 0.25 vCPU / 0.5 GiB, parallelism 1, strict timeout/retry limit.
- Database pool: small fixed maximum with Npgsql multiplexing/pooling evaluated safely.
- Application Insights sampling and daily ingestion guard.

Combined app+sidecar resources mean the free grant is shared and cold starts are expected. If profiling shows memory pressure, the first free-mode simplification is to load inference in the main process while retaining the independently built model artifact/image for the production reference; correctness and cost control take precedence over a decorative process boundary.

### Demo abuse controls

- Turnstile and rate limits on anonymous/account flows.
- Per-user, per-clinic, and global daily quotas for tenants, appointments, imports, and messages.
- Max upload and row counts.
- Registration can switch to invite-only/showcase mode.
- Synthetic-only terms and free-text minimization.
- Scheduled reset of shared demo tenant.
- Read-only kill switch.
- Alert at 50%, 75%, and 90% of internally measured quotas; disable nonessential jobs at threshold.

## 11. Production reference profile

The paid/real-data design differs intentionally:

- Separate public web/API and internal inference services with independent scaling.
- Managed PostgreSQL appropriate to contractual requirements, point-in-time recovery, HA, and tested restoration.
- Key Vault/managed identity and persistent Data Protection keys protected by HSM/key service as appropriate.
- Durable broker such as Azure Service Bus.
- Dedicated worker services/jobs.
- Contractually appropriate email/SMS provider and BAAs where required.
- WAF/API gateway decision based on threat model.
- Multiple replicas, deployment slots/revisions, private networking only when justified.
- Formal logging, SIEM, incident response, vulnerability management, DR, vendor risk, retention, and access review.
- Real EHR integration only through approved sandbox and then controlled production onboarding.

No real-data deployment can reuse the public free environment.

## 12. GitHub and delivery workflow

### Repository policy

- Public repository to keep CI/security features and standard Actions cost-free.
- MIT license unless the owner chooses otherwise before publication.
- `main` is deployable; short-lived `codex/*` or feature branches.
- Pull request required for meaningful changes after bootstrap.
- Conventional commits and release tags.
- Branch protection/ruleset: required CI, CodeQL, no force pushes, linear history preferred, signed commits optional.
- No generated datasets, secrets, production exports, or large model binaries committed directly.

### CI on every pull request

1. Restore with locked dependencies.
2. `dotnet format --verify-no-changes` and frontend lint/format.
3. Build with warnings as errors.
4. Unit, architecture, frontend, ML-contract, and integration tests.
5. EF migration consistency check.
6. OpenAPI compatibility check.
7. Playwright critical-flow tests against ephemeral local stack where time permits.
8. CodeQL and dependency review.
9. Build containers and scan with Trivy or equivalent.
10. Upload small test/coverage artifacts with short retention.

### Deployment workflow

1. Merge to `main` after required checks.
2. Build reproducible images with source/version labels and SBOM.
3. Push to GHCR and capture immutable digests.
4. Deploy Bicep changes using GitHub-to-Azure OIDC.
5. Apply database migration as a one-off guarded job before new revision traffic.
6. Deploy app/sidecar and worker definitions.
7. Run health, auth, booking, and prediction smoke tests using synthetic account.
8. Automatically roll back/deactivate new revision if smoke tests fail.
9. Create GitHub deployment record and release notes.

### Model workflow

- Manual dispatch initially; optional scheduled evaluation only after stable pipeline.
- Generate/load synthetic training data, run leakage tests, train baselines/candidate, evaluate gates, and produce model card.
- A candidate failure ends without artifact promotion.
- Successful candidate creates a versioned inference image, but production/demo promotion requires explicit GitHub environment approval.
- Deploy and run contract/calibration smoke tests; rollback by image digest.

### Security features

Enable Dependabot alerts/updates, secret scanning, push protection, CodeQL, and public security reporting. GitHub documents that public repositories receive a number of these security capabilities for free. [GitHub security features](https://docs.github.com/en/code-security/getting-started/github-security-features)

## 13. Observability and runbooks

### Signals

- HTTP rate, latency, status, cold-start duration, database pool, and retry count.
- Job lease, duration, processed/failed/dead-letter counts.
- Notification quota, delivery, bounce, and suppression.
- Slot conflict and idempotent replay counts.
- Inference latency, fallback, invalid response, active version, and drift/outcome reports.
- Security events: login failures, lockouts, session revocation, role changes, suspicious token replay.

### Health endpoints

- `/health/live`: process only.
- `/health/ready`: database, migration version, and essential configuration.
- `/health/dependencies`: authorized detail for inference/email/integration degradation.

### Required runbooks

- Failed deployment/rollback.
- Broken migration.
- Database unavailable or pool exhausted.
- Model unavailable/schema mismatch/rollback.
- Email quota exhausted or provider failure.
- Suspected secret leak/account takeover.
- Cross-tenant authorization incident.
- Free-tier quota approaching limit.
- Demo reset and seed restoration.

## 14. Local development profile

Docker Compose services:

- PostgreSQL.
- Mailpit for email capture.
- API/web.
- Inference.
- Worker invoked by profile or command.
- Optional mock FHIR server/adapter fixture.

Developer conveniences:

- One command to restore/build/start after prerequisites.
- Deterministic synthetic seed.
- Test Turnstile bypass only through Cloudflare-provided test keys/configuration, never a production secret flag.
- HTTPS development certificate or documented localhost exception.
- `.env.example` contains names only, no usable secrets.

Current workspace readiness check on 2026-11-24 found Git and GitHub CLI, but no `dotnet`, `npm`, Docker CLI, or Azure CLI available on `PATH`. Installation/bootstrap of those prerequisites is a gated first implementation task, not part of planning.

## 15. Architectural quality gates

Implementation is not ready to deploy until:

- Same-origin authentication, antiforgery, session revocation, and key persistence survive container restart tests.
- Cross-tenant test suite covers all resource families.
- Concurrent booking and offer-acceptance tests demonstrate exactly-one success.
- Outbox/provider retry tests demonstrate no duplicate logical message.
- Model contract rejects corrupt/incompatible output and scheduling continues.
- Migrations run from a clean database and upgrade a prior release snapshot.
- Containers run unprivileged, use read-only filesystem where feasible, expose only required ports, and have health checks/resource limits.
- Bicep what-if contains no paid resource outside the approved free profile.
- Public demo displays synthetic-data notice and SMS simulation status.
- README contains live demo, architecture, local run, model card, security limitations, and verified screenshots/video.


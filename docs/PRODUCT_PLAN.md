# AccessFlow product plan

Status: proposed for approval before implementation  
Planning date: 2026-11-24  
Initial market: small US outpatient specialty practices with 3–30 providers  
Deployment profile: public synthetic-data portfolio demo

## 1. Executive summary

AccessFlow helps outpatient clinics recover appointment capacity without making clinical decisions. It predicts whether an appointment is at risk of a no-show or late cancellation, chooses a policy-approved confirmation or rescheduling workflow, and fills confirmed openings from an opt-in waitlist.

The product must optimize an operational outcome, not a model metric. Its north-star outcome is **eligible appointments recovered and offered to waiting patients**. Model accuracy is useful only when it improves that outcome without creating unfair or excessive patient contact.

The first market wedge is a small outpatient specialty clinic whose existing scheduler can export appointments but whose staff still manage confirmations, cancellations, and waitlists manually. AccessFlow is not a replacement EHR. CSV import and an internal demo scheduler come first; a FHIR adapter is added after the workflow is correct.

## 2. Evidence and product rationale

- A 2025 primary-care study covering 1,118,236 appointments reported 6.9% no-shows and 6.8% late cancellations; gradient boosting performed best among the evaluated models. [Study](https://pubmed.ncbi.nlm.nih.gov/40721333/)
- A randomized quality-improvement initiative found model-directed outreach reduced no-shows in its setting, including a larger improvement for Black patients. [Study](https://pubmed.ncbi.nlm.nih.gov/37126125/)
- A recent prospective MRI study showed the necessary caution: prediction alone did not produce an overall attendance improvement when the intervention failed to reach or change enough patients, and performance drifted prospectively. [Study](https://pubmed.ncbi.nlm.nih.gov/42553932/)
- Existing scheduling products sell waitlist recovery and no-show functionality, which validates the category but also means AccessFlow needs a focused wedge rather than a generic scheduling clone.

The resulting product thesis is:

> A no-show score is not the product. A safe, measurable path from risk to confirmation, rescheduling, and recovered access is the product.

Desk research does not establish product-market fit. Before a real-data pilot, the project needs at least ten interviews with clinic schedulers or practice managers, two workflow-validation partners, and one partner willing to test an anonymized export under an appropriate agreement.

## 3. Target users

### Initial customer profile

- US outpatient specialty practice.
- 3–30 providers and one or more locations.
- Meaningful wait time for appointments.
- Staff currently call or message patients manually.
- Existing EHR/scheduler can export CSV, or exposes a FHIR/API sandbox.
- No requirement for AccessFlow to store clinical notes, diagnoses, medications, or imaging.

### Product actors

| Actor | Primary goal |
|---|---|
| Visitor | Understand the product and create an account safely. |
| Patient | Book, confirm, cancel, reschedule, and request an earlier eligible slot. |
| Clinic owner/admin | Configure the organization, locations, policies, staff, integrations, and reporting. |
| Scheduler/front desk | Manage calendars, exceptions, waitlists, outreach, and conflicts. |
| Clinician | View schedules, set non-diagnostic eligibility rules, record outcomes, and assign clinical priority manually. |
| Read-only auditor | Review configurations, outcomes, model versions, and audit events without changing data. |
| System worker | Process outbox messages, offers, expirations, imports, and scheduled scoring idempotently. |
| External scheduling system | Exchange appointments, slots, patients, and status changes through an adapter. |

### Role policy

Patients may access only their own portal data. Staff access is tenant-scoped and least-privilege. A user may belong to multiple clinics, but must explicitly select a clinic context. No role may silently cross tenant boundaries.

## 4. User jobs

### Patient jobs

- See accurate appointment details without calling the clinic.
- Confirm attendance quickly.
- Cancel or reschedule early without losing access to care.
- Join a waitlist with precise availability preferences.
- Accept or decline an earlier appointment without double booking.
- Control communication channels, consent, and quiet hours.

### Clinic jobs

- Identify appointments that deserve attention while there is still time to act.
- Avoid indiscriminate reminders and unnecessary staff calls.
- Open a cancelled slot safely and fill it with an eligible waiting patient.
- Prevent double bookings, stale offers, and schedule divergence.
- Understand recovered capacity, waiting-time reduction, and intervention outcomes.
- Audit who or what changed an appointment.

### Engineering and ML jobs

- Deploy and roll back app and model versions independently.
- Trace a prediction to its model, features, and subsequent outcome.
- Detect calibration degradation, drift, and group-level performance differences.
- Reproduce builds, migrations, training, evaluation, and deployments from source control.

## 5. Product principles

1. **Access before utilization:** success includes patients seen sooner, not only fuller calendars.
2. **Human clinical authority:** staff supply clinical urgency and eligibility; ML never invents either.
3. **No punitive automation:** a high-risk prediction cannot cancel, deny, deprioritize, charge, or shame a patient.
4. **Actionability over sophistication:** a calibrated simple model that enables a useful workflow beats a complex model with no operational effect.
5. **Consent and restraint:** outreach respects opt-in state, channel preferences, quiet hours, frequency limits, and opt-outs.
6. **Synthetic by default:** the public demo rejects real health data and is visibly labeled as synthetic.
7. **Traceability:** every material state change and prediction has an actor, timestamp, tenant, and correlation identifier.
8. **Fail safely:** when ML or an integration is unavailable, core scheduling continues with clear degraded behavior.

## 6. Scope

### MVP / portfolio release

- Marketing and status pages.
- Patient and staff registration paths.
- Email confirmation, login, logout, remember-me, forgot/reset password, and optional patient 2FA.
- Required staff TOTP 2FA, recovery codes, session/device listing, and remote revocation.
- Multi-tenant clinics, locations, memberships, invitations, and roles.
- Provider, visit-type, calendar, hours, holidays, and schedule-exception management.
- Appointment search, booking, confirmation, cancellation, and rescheduling.
- Staff check-in, completion, and no-show recording.
- Patient waitlist preferences and withdrawal.
- Deterministic waitlist eligibility and transactional offer acceptance.
- Email notifications and a simulated SMS outbox.
- Appointment risk model with confidence, model version, and non-causal risk signals.
- Operations dashboard and audit viewer.
- CSV import/export and a mock FHIR adapter.
- Fully synthetic seeded demonstration tenant.
- Public GitHub repository, CI/CD, containers, infrastructure as code, and live free-tier deployment.

### Version 1 after validated workflow

- Real FHIR sandbox integration for `Patient`, `Practitioner`, `Schedule`, `Slot`, and `Appointment` resources.
- Intervention-response model after enough labeled outreach data exists.
- Learned waitlist ranking after enough offer data exists.
- Configurable sequential versus small-batch offers.
- Web push notifications.
- Clinic-level model calibration and champion/challenger evaluation.
- Data export and account-deletion workflow suitable for a controlled pilot.

### Explicitly out of scope for the public release

- Clinical notes, diagnoses, medications, test results, images, billing, claims, or insurance adjudication.
- Medical advice, symptom interpretation, diagnosis, treatment, clinical triage, or emergency guidance.
- Automatic overbooking.
- Automatic cancellation based on a prediction.
- ML-generated clinical priority.
- Penalties or differential fees based on risk.
- Real patient data, real production EHR connections, or claims of HIPAA compliance.
- Real SMS unless a funded/approved provider and compliance review are added.
- Minors, guardianship, proxy access, emergency scheduling, and inpatient scheduling in the first release.
- Native mobile applications.
- A full EHR or practice-management replacement.

## 7. Core end-to-end workflow

1. An admin configures a clinic, location, providers, visit types, opening hours, scheduling rules, and outreach policy.
2. A patient registers, verifies email, accepts the synthetic-demo notice, and books an eligible slot.
3. The API commits the appointment and an outbox event atomically.
4. The inference service scores the appointment. The prediction is stored with model and feature-schema versions.
5. Policy determines whether and when to send a normal confirmation, rescheduling link, or staff task. Prediction never directly changes the appointment.
6. The patient confirms, cancels, reschedules, or does not respond.
7. A confirmed cancellation makes the slot available after external-sync and eligibility checks.
8. The waitlist engine finds candidates using hard clinic rules, staff-supplied priority, preferences, and waiting time.
9. A time-limited offer is sent. Acceptance acquires a database lock, revalidates eligibility and slot availability, books exactly once, and supersedes other offers.
10. Appointment outcomes eventually become training/evaluation labels. Dashboards report capacity recovered, waiting days reduced, outreach burden, and model performance.

## 8. Functional requirements

### Identity and sessions

- Browser authentication uses ASP.NET Core Identity cookies, never access tokens in local storage.
- Registration requires a valid email, accepted terms, and bot validation in public-demo mode.
- Email verification and password-reset tokens are expiring, single-purpose, and single-use at the application layer.
- Successful login rotates the authentication session identifier.
- Staff must enroll TOTP before accessing tenant data; recovery codes are shown once and stored hashed.
- Users can see approximate device, created time, last activity, and revoke individual or all sessions.
- Password and privilege changes revoke other sessions through the Identity security stamp.
- Authentication failures do not reveal whether an email exists.

### Clinic configuration

- A clinic has a name, default timezone, locations, communication policy, retention policy, and demo/production mode.
- Each location has an IANA timezone, address label, hours, closures, and supported visit types.
- Provider schedules support recurring availability plus dated exceptions.
- Visit types define duration, buffers, location/provider eligibility, booking horizon, cancellation cutoff, and whether waitlist recovery is allowed.
- Configuration changes affecting future bookings show an impact preview before commit.

### Scheduling

- Search returns only slots that remain valid under provider, location, visit-type, booking-horizon, and blackout rules.
- A short-lived hold prevents a selected slot from being taken during confirmation.
- Booking is idempotent and conflict-safe under concurrent requests.
- Rescheduling creates an auditable link from old to new appointment and releases the old slot only after the new booking succeeds.
- Cancellation records actor, timestamp, reason category, optional patient-safe note, and source.
- Staff can mark checked-in, completed, or no-show according to a validated state machine.
- All stored instants are UTC; user-facing rendering uses the location timezone and displays timezone ambiguity when relevant.

### Waitlist and offers

- Preferences include visit type, acceptable providers/locations, date range, days of week, time windows, minimum notice, and accessibility constraints represented as non-clinical scheduling rules.
- Clinic-defined eligibility and staff-entered clinical priority are evaluated before any predictive score.
- Offers contain opaque, expiring, single-use tokens and reveal the minimum necessary information.
- Accepting an offer rechecks the slot, patient, appointment conflicts, eligibility, and offer status within one serializable transaction.
- Decline, expiry, cancellation, or supersession advances the next candidate according to policy.
- Frequency limits prevent repeatedly contacting the same patient.

### Communications

- Templates are clinic-approved and contain no diagnosis or sensitive reason in subjects, previews, or URLs.
- Notifications use an outbox and deterministic idempotency key.
- Quiet hours are evaluated in the patient's configured timezone.
- Opt-outs are channel-specific and immediately honored except for narrowly defined transactional security messages.
- Delivery, bounce, complaint, open, click, response, and failure events are deduplicated.
- Public-demo SMS is captured in an in-app outbox and never sent externally.

### Risk and recommendations

- Appointments receive `attend`, `late-cancel`, and `no-show` probabilities when a compatible model is available.
- The UI shows uncertainty and descriptive risk signals, not causal claims.
- A missing, stale, incompatible, or unavailable model produces an `unavailable` status and policy falls back to standard reminders.
- Predictions are immutable records. Rescoring creates a new record and marks the currently active prediction.
- Staff can record whether an intervention was performed and its outcome.

### Reporting and audit

- Operational metrics: utilization, no-show rate, late-cancellation rate, confirmation rate, open slots, recovered slots, offer acceptance, median time-to-fill, waiting days reduced, and staff tasks.
- ML metrics: class distribution, precision/recall, PR-AUC, calibration error, top-risk recall, drift, and group slices where ethically and legally supportable.
- Audit events record actor, action, object type/id, tenant, source, result, timestamp, correlation id, and redacted change summary.
- Audit data is append-only through the application.

## 9. Success metrics and guardrails

### North-star and outcome metrics

- Eligible cancelled slots recovered.
- Median time from cancellation to accepted replacement.
- Total and median waiting days saved.
- Confirmation or early-reschedule rate after outreach.
- Front-desk manual touches per 100 appointments.

### Model quality gates

- Beat a clinic-rate and logistic-regression baseline on a chronological holdout.
- Report PR-AUC and calibration, not accuracy alone.
- Report top-10% and top-20% risk recall.
- No automatic promotion if calibration or a defined fairness slice materially regresses.
- Prospective performance is measured separately from retrospective validation.

### Safety and experience guardrails

- Zero automatic cancellations or clinical-priority changes.
- Zero cross-tenant access in automated authorization tests.
- Zero double bookings under concurrency tests.
- Contact-frequency and quiet-hour violations must be blocked.
- Accessibility target: WCAG 2.2 AA for patient and staff web experiences.
- Patient-facing language target: plain language, no blame, and no revelation of risk scores to patients.

## 10. Non-functional requirements

| Area | Requirement |
|---|---|
| Availability | Demo may scale to zero and cold-start; core operations must fail clearly and recover. Production reference target is 99.9% after paid infrastructure. |
| Performance | Warm p95 read API under 400 ms and write API under 800 ms excluding external providers; slot search under 1 s for the seeded demo. |
| Concurrency | Booking and offer acceptance remain correct with at least 50 concurrent attempts for the same slot. |
| Security | OWASP ASVS 5.0 Level 2 used as the verification baseline for internet-facing functionality. |
| Privacy | No PHI in logs, metrics, URLs, exception messages, model artifacts, public fixtures, or source control. |
| Reliability | All external webhooks, imports, outbox events, and commands are idempotent and retry-safe. |
| Observability | Correlation IDs, structured logs, traces, health/readiness probes, audit records, and model-version metrics. |
| Maintainability | Modular-monolith boundaries are enforced by architecture tests; no direct cross-module database writes. |
| Portability | Application and model run locally through containers and deploy through provider-neutral OCI images. |
| Reproducibility | Lock files, pinned actions, deterministic seed generation, migration checks, and versioned model metadata. |
| Accessibility | Keyboard navigation, visible focus, semantic labels, screen-reader status messages, contrast, reduced motion, and accessible time/date inputs. |

## 11. Data handling posture

### Public demo

- Synthetic names, contact details, appointments, and outcomes only.
- Persistent banner and seed metadata identify synthetic records.
- Registration terms prohibit entering real patient or health information.
- Free-text fields are minimized and automatically stripped from exported demo datasets.
- Demo tenant can be reset without affecting personal accounts.

### Real pilot prerequisite

Before processing real patient data, the project requires legal/compliance ownership, vendor BAAs where applicable, a formal risk analysis, data-flow review, incident-response process, retention/deletion policy, workforce access policy, verified backups, disaster recovery, vendor due diligence, and paid infrastructure appropriate to the required obligations. Code alone cannot establish HIPAA compliance.

## 12. Market-validation gate

Implementation can build the public synthetic portfolio release without a clinic partner. Claims of market fit or a real-data pilot require all of the following:

- Ten structured interviews with schedulers/practice managers.
- At least five confirm the workflow is frequent and costly.
- At least two validate the end-to-end mockup and agree to a follow-up.
- At least one can provide a legally usable anonymized schema/sample or sandbox.
- A baseline is documented for current no-show, late-cancel, time-to-fill, and staff effort.
- The clinic identifies which intervention it is actually willing and able to perform.

If those conditions fail, retain the project as a portfolio demonstration and do not claim product-market fit.


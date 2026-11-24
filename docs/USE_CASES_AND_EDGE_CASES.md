# AccessFlow use cases and edge cases

This document is the behavioral contract for the initial release. “All use cases” means all behavior inside the agreed product boundary; medical care, diagnosis, triage, billing, minors/proxy access, and full EHR functionality remain explicitly out of scope.

## 1. State models

### Appointment states

```text
Held -> PendingConfirmation -> Confirmed -> CheckedIn -> Completed
                          |          |            |
                          |          |            +-> NoShow (only if not checked in)
                          |          +-> CancelledByPatient
                          |          +-> CancelledByClinic
                          |          +-> Rescheduled
                          +-> CancelledByPatient
                          +-> Expired (unconfirmed hold/booking policy only)
```

Rules:

- `Held` is short-lived and is not an appointment visible to clinical staff.
- `Completed`, `NoShow`, `CancelledByPatient`, `CancelledByClinic`, `Rescheduled`, and `Expired` are terminal.
- A reschedule succeeds atomically or leaves the original appointment unchanged.
- A high-risk prediction never changes status.
- Staff may correct an erroneous terminal status only through a privileged correction command that writes an audit event and preserves the prior status history.

### Waitlist entry states

```text
Active <-> Paused
Active -> Offered -> Fulfilled
Active -> Withdrawn
Active -> Expired
Offered -> Active       (offer declined/expired and entry remains eligible)
Offered -> Withdrawn
```

### Slot-offer states

```text
Pending -> Accepted
Pending -> Declined
Pending -> Expired
Pending -> Superseded
Pending -> Invalidated
```

Only one offer can be `Accepted` for a slot. Acceptance does not complete until the replacement appointment is committed.

### Import states

```text
Uploaded -> Validating -> Ready -> Applying -> Completed
                  |         |         |
                  +-> Rejected        +-> PartiallyCompleted
                            +------------> Failed
```

Validation and application are separate so the user can preview consequences before any write.

## 2. Identity and account use cases

### UC-001 — Register a patient

Actor: visitor.

Main flow:

1. Visitor enters email, password, name, timezone, terms acceptance, and bot-verification token.
2. Server validates bot token, normalizes email, checks rate limits, creates an unconfirmed account, and emits a verification email through the outbox.
3. UI displays a non-enumerating confirmation message.

Acceptance and exceptions:

- A duplicate email returns the same user-facing result as a new email and may resend within a strict cooldown.
- Weak or breached passwords are rejected without logging the password.
- A used, forged, expired, or hostname-mismatched bot token is rejected.
- Public-demo copy explicitly prohibits real health information.
- If email delivery fails, the account remains unconfirmed and a bounded retry is scheduled.

### UC-002 — Verify email

- Verification token is URL-safe, expiring, purpose-bound, and effectively single-use.
- Replaying a successful link shows “already verified” without error.
- Verification does not automatically grant clinic membership.
- Changing the email invalidates outstanding verification tokens.

### UC-003 — Staff accepts an invitation

- Invitation is tenant-, email-, role-, and expiry-bound.
- Existing users authenticate before acceptance; new users register first.
- Email mismatch requires an admin to reissue; it cannot be overridden by the recipient.
- Revoked, expired, replayed, or already accepted invitations are handled idempotently.
- A user invited to two clinics receives two separate memberships.

### UC-004 — Log in

- Valid credentials plus required 2FA create a new session record and rotate the authentication cookie.
- Failed responses do not reveal whether email, password, confirmation, lockout, or membership was the exact cause.
- Login is rate-limited by normalized account key and network partition.
- An unconfirmed, disabled, deleted, or locked account receives safe recovery guidance.
- Return URLs are local/allowlisted to prevent open redirects.

### UC-005 — Complete staff 2FA

- Staff cannot enter tenant data before TOTP enrollment and verification.
- TOTP permits a small clock skew and blocks code replay within the accepted time step.
- Recovery codes are single-use, stored hashed, and regenerated only after reauthentication.
- Excessive failures trigger cooldown and audit events.

### UC-006 — Log out

- Current server-side session is revoked and the cookie is deleted.
- Logout is idempotent.
- A failed audit write cannot leave the session active; security state takes precedence.

### UC-007 — Remember this device

- Persistent cookies use an absolute maximum lifetime in addition to sliding inactivity expiry.
- Staff policies may prohibit persistence.
- A security-stamp change invalidates remembered cookies.

### UC-008 — Request password reset

- Response is identical for known and unknown emails.
- Reset emails have cooldowns and per-account/per-network limits.
- A new reset request invalidates or supersedes earlier outstanding reset intents according to policy.
- No password or raw token appears in logs or audit metadata.

### UC-009 — Reset password

- Token is purpose-bound, expiring, and single-use.
- New password policy is applied; confirmation must match.
- Success revokes other sessions and recovery links.
- Replayed links display a safe expired/used message.

### UC-010 — Change password

- Requires current password or recent strong reauthentication.
- Revokes other sessions and rotates the current one.
- Emits a security notification without exposing sensitive details.

### UC-011 — List and revoke sessions

- User sees creation, last-seen, approximate client/device, and current-session marker.
- IP is not displayed precisely and is retained only as permitted.
- Revoking the current session behaves as logout.
- “Revoke all others” is atomic and audited.

### UC-012 — Change email

- Requires recent reauthentication and verification of the new address.
- The old address receives a security notification.
- Clinic invitations and uniqueness conflicts are revalidated.
- Until verification, login behavior follows an explicit pending-change policy.

### UC-013 — Disable or restore account

- Users may request deactivation; admins may disable staff within their tenant.
- Disabling immediately revokes relevant sessions.
- Tenant admins cannot disable the last active owner without first transferring ownership.
- Restoration requires identity verification and does not silently restore expired memberships.

### UC-014 — Export or delete demo account

- Export is prepared asynchronously, expires, and excludes other tenants’ data.
- Deletion respects security/audit retention and pseudonymizes where deletion would break integrity.
- The UI explains the distinction between account data and tenant-controlled records.

### UC-015 — Recover from lost 2FA

- Recovery code is preferred.
- No self-service bypass exists after recovery codes are exhausted.
- Administrative recovery requires a documented, audited identity-verification workflow and cannot be performed by a peer with insufficient privilege.

## 3. Tenant, clinic, and staff use cases

### UC-016 — Create demo clinic

- A verified user becomes owner of a synthetic/demo tenant.
- Clinic name, timezone, and synthetic-data acknowledgement are required.
- Creation is idempotent for retried requests.

### UC-017 — Switch clinic context

- Only active memberships appear.
- Every request resolves tenant from the authenticated membership context, never a client-provided tenant id alone.
- Stale tabs from another clinic receive a conflict/reload response, not cross-tenant data.

### UC-018 — Invite staff

- Owner/admin chooses allowlisted role and email.
- Duplicate pending invitations are consolidated or rate-limited.
- Cannot assign a role the inviter is not permitted to grant.
- Invitation expiry, resend, and revoke are supported.

### UC-019 — Change role or remove staff

- Privilege changes require recent reauthentication for owners.
- Removing or downgrading staff invalidates tenant authorization immediately.
- Last-owner protection is enforced in the database transaction.
- Historical audit authorship remains intact.

### UC-020 — Configure clinic and locations

- Admin manages timezone, address label, hours, closures, and communication defaults.
- Changing timezone shows impact on future appointments and never changes stored instants silently.
- Closing a location with future bookings requires a resolution plan.

### UC-021 — Configure providers and visit types

- Provider may be active, inactive, or archived.
- Visit type defines duration, buffers, eligible providers/locations, booking horizon, cutoff, and recovery policy.
- Archiving preserves historical records and blocks new bookings.
- Shortening duration never mutates existing appointment end times automatically.

### UC-022 — Configure schedules and exceptions

- Recurring availability can be overridden by holidays, leave, closures, or one-off additions.
- Overlapping provider availability is rejected or normalized according to an explicit rule.
- Changes that collide with bookings are previewed and require resolution.

### UC-023 — Configure outreach policy

- Admin sets reminder cadence, channel priority, quiet hours, risk thresholds, maximum contacts, and escalation rules.
- Unsafe combinations are rejected, such as contact after appointment time or excessive daily attempts.
- Policy versions are immutable once used; edits create a new version.

## 4. Patient and scheduling use cases

### UC-024 — Maintain patient profile

- Patient edits preferred name, timezone, and non-clinical communication settings.
- Staff edits are audited and respect field ownership rules.
- Public demo minimizes free text and rejects obvious attempts to submit sensitive clinical content.

### UC-025 — Manage communication consent

- Consent is channel- and purpose-specific with timestamp and source.
- Opt-out is immediate and takes priority over queued non-essential messages.
- Security emails cannot be disabled; marketing is not part of MVP.

### UC-026 — Search available appointments

- Patient selects visit type, location/provider preference, and date range.
- Results are recalculated from authoritative availability and do not expose provider private events.
- Pagination/cursors remain stable enough for user navigation but availability is rechecked on selection.

### UC-027 — Hold a slot

- Hold has a short expiry and is bound to user, visit type, and slot.
- Only one active hold per patient/slot policy.
- Expiry is safe if client disconnects.
- A hold is not a confirmed booking and UI communicates the countdown accessibly.

### UC-028 — Book an appointment

- Server revalidates hold, eligibility, provider/location, conflicts, and booking horizon.
- Idempotency key prevents duplicates on retry/double click.
- Appointment, status history, audit record, and outbox event commit atomically.
- If notification fails, booking remains valid and staff can see delivery status.

### UC-029 — Staff books for patient

- Staff searches or creates a synthetic patient and records booking source.
- Duplicate-patient warning does not expose other tenants.
- Overrides require permission and a reason; clinical eligibility is not inferred by ML.

### UC-030 — View appointment

- Patient sees only minimum necessary scheduling data.
- Staff sees data allowed by role.
- Opaque public links reveal no appointment data until token validation.

### UC-031 — Confirm appointment

- Confirmation is idempotent and records channel/source.
- Confirming a cancelled, expired, completed, or superseded appointment returns its current state.
- A confirmation token cannot be used for a different appointment.

### UC-032 — Cancel appointment

- Patient sees cancellation implications and optional rescheduling/waitlist choices.
- Cutoff policy may route late cancellation to staff instead of blocking silently.
- Cancellation transaction opens capacity only after authoritative checks.
- Duplicate cancellation requests are idempotent.

### UC-033 — Reschedule appointment

- New slot is held and validated before the original is released.
- Transaction creates the replacement, marks the original `Rescheduled`, links both records, and emits events.
- If any step fails, original appointment remains valid.
- Old confirmation and offer tokens are invalidated.

### UC-034 — Staff cancels clinic-side

- Reason category and patient notification choice are required.
- Patients receive safe messaging and expedited rebooking/waitlist options.
- Mass cancellation requires a preview, rate-limited batches, and resumable processing.

### UC-035 — Check in patient

- Only appropriate staff can check in within policy window.
- Repeated check-in is idempotent.
- A cancelled appointment cannot be checked in without correction workflow.

### UC-036 — Complete appointment

- Completion follows check-in unless privileged correction includes a reason.
- Completion supplies the attendance label but no clinical data.

### UC-037 — Mark no-show

- Only after appointment grace period and only by authorized staff or approved job policy.
- Never inferred directly from the risk score.
- Correction is possible with retained history.

### UC-038 — Detect appointment conflict

- Prevent provider overlap, patient overlap, location closure conflict, and invalid buffer overlap.
- Conflict response is safe under races and suggests refreshing availability.
- External-system conflicts quarantine the record for staff resolution instead of overwriting blindly.

## 5. Waitlist and recovery use cases

### UC-039 — Join waitlist

- Patient selects an existing appointment or desired visit type and defines preferences.
- Entry requires valid consent for at least one offer channel or is staff-call-only.
- Duplicate overlapping entries are merged or explained.
- Minimum notice and blackout dates are supported.

### UC-040 — Edit, pause, resume, or withdraw waitlist entry

- Changes invalidate offers that no longer match.
- Withdrawing does not cancel an existing appointment.
- Paused entries keep waiting-time history but receive no offers.

### UC-041 — Open slot after cancellation

- Slot is reopened only if provider/location/visit type still permits booking and external source-of-truth agrees.
- A clinic may suppress recovery for specific cancellations or short-notice windows.
- Duplicate cancellation events create only one recovery workflow.

### UC-042 — Generate candidate set

- Apply tenant, visit type, provider, location, date/time preference, minimum notice, existing-conflict, consent, contact-frequency, and staff-entered priority rules.
- ML may rank only after hard eligibility filtering.
- Candidate calculation is versioned and reproducible.

### UC-043 — Create offer

- Offer reserves no appointment unless clinic policy uses a bounded soft hold.
- Token is opaque, expiring, purpose-bound, and hashed at rest.
- Notification uses minimal scheduling information.
- Duplicate job execution produces the same logical offer.

### UC-044 — Patient accepts offer

- Server locks offer and slot, revalidates all constraints, and books atomically.
- First valid commit wins; later acceptances receive a clear “slot no longer available” message and stay on the waitlist.
- Acceptance cannot cancel the patient’s existing appointment until replacement booking succeeds.

### UC-045 — Patient declines offer

- Decline is idempotent and optionally captures a structured, non-clinical reason.
- Decline does not remove the patient unless requested or policy threshold is reached.
- Candidate frequency policy is updated.

### UC-046 — Offer expires

- Worker uses database time and idempotent transitions.
- Expiry cannot override a concurrent committed acceptance.
- Next-candidate processing begins only after transaction completion.

### UC-047 — Supersede remaining offers

- After acceptance, all competing offers for the slot become superseded.
- Late clicks reveal no identity of the successful patient.
- Unsent queued notifications are suppressed.

### UC-048 — Staff manually assigns slot

- Staff sees eligibility and conflict warnings.
- Override requires permission and reason.
- Manual assignment competes through the same database constraint as automated acceptance.

### UC-049 — Failed recovery

- If no candidate accepts before cutoff, workflow closes with a structured reason.
- Staff receives one task, not repeated alerts.
- Slot remains available for normal booking when permitted.

## 6. Communication use cases

### UC-050 — Send transactional email

- Outbox event renders a versioned template at send time from approved variables.
- Provider idempotency and internal idempotency prevent duplicates.
- No sensitive reason appears in subject, preview text, or provider metadata.

### UC-051 — Simulate SMS

- Demo mode writes the fully rendered message to a tenant-scoped outbox visible only to authorized users.
- UI clearly labels it “not sent.”
- Production configuration cannot accidentally activate without an explicit provider and compliance flag.

### UC-052 — Process delivery webhook

- Signature, timestamp, event id, and expected provider are validated.
- Duplicate and out-of-order events converge to a correct delivery state.
- Unknown messages are quarantined, not attached by guesswork.

### UC-053 — Handle bounce or complaint

- Address/channel is suppressed according to policy.
- Patient and staff see a safe action request.
- The system does not repeatedly retry a permanent failure.

### UC-054 — Respect quiet hours and timezones

- Scheduled send is calculated using patient timezone with clinic fallback.
- DST gaps advance to the next valid time; DST overlaps choose the documented earlier/later rule.
- Urgent operational messages still obey the configured legal/clinic policy; “urgent” is not inferred by ML.

### UC-055 — Retry transient provider failure

- Exponential backoff with jitter and a maximum attempt count.
- Retry uses the same idempotency key.
- Exhaustion creates a staff-visible failure and dead-letter record.

### UC-056 — Disable compromised template/link

- Admin can revoke a template version or link-signing purpose.
- Outstanding tokens can be invalidated by version/security stamp.
- Security event is audited without logging raw links.

## 7. Prediction and ML use cases

### UC-057 — Score new or changed appointment

- Application sends schema-versioned, tenant-scoped features to the internal inference endpoint.
- Service returns class probabilities, model version, schema version, confidence status, and descriptive signals.
- Timeout or invalid response falls back safely and queues bounded retry.

### UC-058 — Re-score before appointment

- Material events such as confirmation or reschedule trigger a new immutable prediction.
- The former prediction remains for audit/evaluation.
- Contact policy prevents repeated outreach merely because risk fluctuates.

### UC-059 — Explain prediction to staff

- UI calls factors “signals,” not causes.
- It shows features available at score time and never leaks protected audit-only attributes.
- Patient UI never displays a stigmatizing risk score.

### UC-060 — Operate without model

- Standard reminder and waitlist rules continue.
- UI shows model unavailable/stale without blocking scheduling.
- Health checks distinguish application readiness from optional inference degradation.

### UC-061 — Evaluate candidate model

- Training is reproducible from versioned synthetic data generator/configuration.
- Chronological split, leakage audit, baselines, calibration, subgroup evaluation, and model card are mandatory.
- Failed gates prevent publication.

### UC-062 — Publish and deploy model

- Artifact checksum, feature schema, training code commit, dataset-generator version, metrics, and approval are recorded.
- Model is baked into a versioned inference image.
- Deployment uses immutable image digest and supports rollback.

### UC-063 — Monitor drift and outcomes

- Outcome labels join only after the appointment is complete/no-show/cancelled.
- Metrics distinguish missing labels from negative outcomes.
- Drift and calibration are measured by clinic and time; alerts do not automatically retrain or promote.

### UC-064 — Correct outcome label

- Correction preserves original label, reason, actor, and timestamp.
- Evaluation is recomputed; published historical reports are versioned.
- Training export uses the latest valid label and records correction provenance.

## 8. Import and integration use cases

### UC-065 — Upload CSV

- File type, size, encoding, row count, malware-safe handling, and tenant quota are checked.
- File receives a random storage name and is never executed.
- Preview contains only the current tenant’s parsed data.

### UC-066 — Map and validate columns

- Required fields, date formats, timezone, identifiers, statuses, and referential relationships are validated.
- Ambiguous local times, duplicate external ids, impossible intervals, and unknown statuses are reported by row.
- No writes occur during validation.

### UC-067 — Apply import

- User reviews create/update/skip/error counts.
- Import is idempotent by source and external id.
- Batch commits are resumable; partial completion has an explicit checkpoint and report.
- Import cannot modify another tenant even with crafted identifiers.

### UC-068 — Export tenant scheduling data

- Export is role-protected, audited, asynchronous, expiring, and encrypted in transit.
- Public demo excludes security, audit internals, raw tokens, and model-only audit attributes.

### UC-069 — Import FHIR bundle in mock mode

- Adapter maps supported resources and rejects unsupported/malformed versions explicitly.
- Patient identifiers are tenant/source namespaced.
- References are resolved without guessing.

### UC-070 — Receive external appointment update

- Verify source authentication/signature and event id.
- Deduplicate, order by source version/timestamp policy, and use optimistic concurrency.
- Conflicting local/external changes enter reconciliation instead of last-write-wins data loss.

### UC-071 — External system unavailable

- Local action follows configured source-of-truth policy.
- Pending synchronization is visible and retried.
- Waitlist offer is blocked if slot authority cannot be confirmed and clinic policy requires confirmation.

### UC-072 — Reconcile divergence

- Authorized staff compare local and external versions.
- Resolution is explicit, audited, idempotent, and emits required follow-up events.
- No model makes the reconciliation decision.

## 9. Reporting, audit, and operations use cases

### UC-073 — View dashboard

- Metrics have documented definitions, timezone, date range, refresh time, and data-completeness indicator.
- Small groups are suppressed where a real deployment could create re-identification risk.

### UC-074 — View audit trail

- Filter by actor, action, object, result, and time.
- Sensitive values are redacted; raw tokens/passwords/message bodies are never present.
- Read access itself is audited for privileged investigations where configured.

### UC-075 — Review failed jobs/dead letters

- Staff sees safe error category, correlation id, attempts, and next action.
- Retry is permission-protected and idempotent.
- Technical details remain in restricted logs.

### UC-076 — Health and readiness

- Liveness checks process health only.
- Readiness checks database/migrations and essential dependencies.
- Model/email degradation is exposed separately so the scheduler can remain usable.

### UC-077 — Reset synthetic demo tenant

- Reset is restricted to seeded demo tenants.
- User accounts and security audit events are preserved as designed.
- Operation shows exact scope, requires confirmation, is idempotent, and can be rerun after interruption.

### UC-078 — Free-tier kill switch

- Owner can put the public app into read-only showcase mode.
- Automated usage guard can stop nonessential jobs and registration before free quotas are exceeded.
- Existing data remains exportable to the maintainer.

## 10. Cross-cutting edge-case catalogue

### Time and calendar

- Daylight-saving spring gap and fall overlap.
- Patient, clinic, location, provider, and browser timezones disagree.
- Booking across midnight or year boundary.
- Leap day and month-end recurrence.
- Clinic changes timezone after bookings exist.
- Provider exception partially overlaps a slot.
- Cancellation cutoff occurs during DST transition.
- External system supplies local time without offset.
- Clock skew between app, database, email provider, and client.

Resolution: database time is authoritative for transitions; instants are UTC; scheduling rules use IANA zones; ambiguous or invalid imported local times require resolution.

### Concurrency and idempotency

- Two patients book the same slot simultaneously.
- Patient double-clicks booking or acceptance.
- Staff assignment races automated offer acceptance.
- Cancellation races check-in or completion.
- Offer expiry races acceptance.
- Reschedule races external cancellation.
- Webhook is duplicated, delayed, or out of order.
- Worker crashes after external send but before local acknowledgement.
- Import is restarted after partial commit.

Resolution: database constraints, short transactions, row/advisory locks where appropriate, idempotency records, outbox/inbox patterns, provider idempotency, and monotonic state machines.

### Authorization and tenancy

- User changes tenant id in URL/body.
- Stale tab retains a former role.
- Membership is removed during an operation.
- User belongs to multiple clinics.
- Invitation is forwarded.
- Admin attempts to grant a superior role.
- Last owner removes self.
- Support/debug endpoint exposes cross-tenant data.

Resolution: authorization is checked server-side on every command/query using current membership; tenant filters and tests are mandatory; client tenant identifiers are never trusted alone.

### Authentication and session security

- Credential stuffing, password spraying, enumeration, bot registration, reset flooding.
- Session fixation, stolen cookie, replayed reset/invitation/offer token.
- Cookie persists after role/password change.
- Data-protection keys disappear during container restart.
- Cross-site request forgery against cookie-authenticated endpoints.
- Open redirect in return URL.

Resolution: Identity, secure HTTP-only SameSite cookies, antiforgery, Turnstile on abuse-prone anonymous flows, rate limits, security stamps, persistent protected data-protection keys, token hashing/versioning, and safe redirects.

### Scheduling integrity

- Provider/location is archived with future bookings.
- Visit duration changes after booking.
- Patient has another appointment at same time.
- Slot is valid locally but unavailable externally.
- Clinic cancels a day of appointments.
- Appointment is mistakenly marked no-show.
- Buffer times overlap despite non-overlapping visible appointments.

Resolution: historical snapshots, impact previews, exclusion/unique constraints, correction workflow, and reconciliation queue.

### Communication

- Email bounces, provider throttles, webhook signature fails, link scanner opens link.
- Patient shares a confirmation/offer link.
- Message is due during quiet hours.
- User opts out after message is queued.
- Template variable is missing or contains injection content.
- Brevo free email quota is exhausted.

Resolution: token use requires a state-changing confirmation step or authenticated context where appropriate; render with encoding and a strict variable contract; recheck consent at send time; quota exhaustion creates in-app tasks rather than silent loss.

### ML and fairness

- New clinic/patient has no history.
- Model schema and app schema differ.
- Model returns NaN, probabilities outside `[0,1]`, or does not sum correctly.
- Retrospective performance is good but prospective performance degrades.
- Risk score changes after confirmation.
- Protected attribute is accidentally included.
- High-risk patients receive excessive contact or reduced access.
- Model artifact is corrupt or untrusted.

Resolution: cold-start baseline, strict contracts, checksum/signature, probability validation, fallbacks, frequency limits, feature allowlist, subgroup monitoring, model cards, no access-denying actions, and explicit deployment gates.

### Privacy and public-demo abuse

- User enters real PHI into a free-text field.
- Synthetic record looks like a real person.
- Export includes tokens or audit internals.
- Search engine indexes patient-facing pages.
- Public user creates excessive tenants/data/messages.
- Demo screenshots reveal personal registration email.

Resolution: minimize free text, visible prohibition, synthetic markers, `noindex` on authenticated routes, quotas, rate limits, redacted exports, seeded fake domains, and demo reset.

### Free-tier and provider failure

- Azure cold start causes first request timeout.
- Neon database is suspended or connection pool exhausted.
- Free email or compute quota is reached.
- Provider changes its free tier.
- GitHub Actions deployment fails midway.
- Container image cannot be pulled.

Resolution: retry-safe startup, bounded connection pool, friendly cold-start UI, health probes, pinned image digests, previous revision retention, usage guard, provider abstraction, and documented local deployment fallback.

## 11. Global acceptance rules

Every mutating endpoint must satisfy all applicable rules:

1. Authenticated actor or validated single-purpose token.
2. Current tenant membership and permission.
3. Antiforgery protection for cookie-authenticated browser commands.
4. Input validation with RFC 9457/Problem Details response.
5. Idempotency for retry-prone commands.
6. Optimistic or pessimistic concurrency as appropriate.
7. Atomic domain state, audit record, and outbox write.
8. No secrets or sensitive payloads in logs.
9. Accessible success/error feedback.
10. Unit, integration, authorization, and relevant concurrency tests before completion.


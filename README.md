# AccessFlow

AccessFlow is a planned multi-tenant outpatient-clinic scheduling and access platform. It predicts appointment non-attendance, coordinates appropriate confirmation or rescheduling outreach, and recovers cancelled capacity through a consent-based waitlist.

This directory currently contains planning artifacts only. Implementation must not begin until the planning gates in `docs/DELIVERY_PLAN.md` are accepted.

## Product boundary

AccessFlow is a scheduling operations product, not an electronic health record, diagnostic tool, triage system, or medical device. It must never automatically cancel an appointment, decide clinical urgency, deny care, or generate medical advice.

The public portfolio deployment will use synthetic data only. It must not be described as HIPAA compliant and must not accept real patient health information.

## Plan index

- [Product plan](docs/PRODUCT_PLAN.md) — problem, users, scope, workflows, success metrics, and non-functional requirements.
- [Use cases and edge cases](docs/USE_CASES_AND_EDGE_CASES.md) — actor-by-actor behavior, state transitions, failure cases, and acceptance rules.
- [Architecture, ML, and free deployment](docs/ARCHITECTURE_ML_DEPLOYMENT.md) — solution structure, data model, APIs, security, models, MLOps, GitHub, and $0 demo hosting.
- [Delivery plan](docs/DELIVERY_PLAN.md) — implementation phases, backlog, validation gates, testing, definition of done, and risks.

## Agreed direction

- Backend: C# and ASP.NET Core on .NET 10 LTS.
- Frontend: React and TypeScript, served from the ASP.NET Core origin.
- Data: PostgreSQL with Entity Framework Core.
- ML: ML.NET baselines and LightGBM models, served by a separately containerized inference process.
- Local development: Docker Compose with PostgreSQL, Mailpit, API, web UI, worker, and model service.
- Demo hosting: Azure Container Apps Consumption, Neon Postgres, GHCR, Brevo transactional email, and Cloudflare Turnstile free tiers.
- Repository: a separate public GitHub repository named `accessflow-clinic` unless renamed before repository creation.
- Delivery: trunk-based feature branches, pull requests, automated tests, CodeQL, dependency scanning, container scanning, and automated deployment from `main`.

## Free-demo constraints

The $0 architecture is intentionally a portfolio/demo profile:

- Synthetic patients and appointments only.
- One Azure Container App, scaling from zero to one replica.
- API/web container plus an ML inference sidecar.
- Scheduled worker as an Azure Container Apps Job.
- No real SMS; SMS is captured in a demo outbox. Transactional email is quota-limited.
- No uptime SLA; cold starts are expected.
- No custom domain is required; the Azure-generated HTTPS hostname is sufficient.
- No paid service is enabled automatically.
- A usage guard and kill switch are required because Azure budgets warn about spend but are not hard spending caps.

The production reference profile is documented separately and requires paid, contractually appropriate infrastructure before any real health data is processed.

## Success statement

The project is successful when a reviewer can register, authenticate, configure a synthetic clinic, book and manage appointments, join a waitlist, exercise the complete cancellation-to-rebooking workflow, inspect risk predictions with model/version metadata, and verify the entire system through public source code, tests, documentation, CI/CD, and a live synthetic demo.


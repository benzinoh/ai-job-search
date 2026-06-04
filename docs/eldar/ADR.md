# Eldar — Architecture Decision Records

**Format:** Each ADR captures a decision, the context that drove it, the options considered, the chosen option, and the consequences.

---

## ADR-001: Modular Monolith over Microservices

**Date:** 2026-06-04
**Status:** Accepted

### Context
Eldar has 8 distinct domains (auth, profile, jobs, evaluation, documents, submissions, tracking, billing). We need to decide how to deploy and organise them from day one.

### Options Considered
**A — Pure monolith:** Single FastAPI app, no internal module boundaries enforced.
**B — Microservices from day one:** 8 independent deployables, event-driven via EventBridge.
**C — Modular monolith:** Single deploy, strict internal module boundaries, interfaces between modules.

### Decision
**C — Modular monolith.**

### Rationale
- Microservices from day one create distributed debugging complexity before product-market fit is proven
- A pure monolith will require structural refactoring as load grows
- Modular monolith enforces SOLID boundaries (each module: single responsibility, interface segregation, dependency inversion) while keeping a single deployable
- When individual modules need independent scaling (e.g., document generation under heavy load), they extract to Lambda workers via SQS — which is already the pattern for async stages
- Matches the proven MyIQ pattern, reducing new architectural risk

### Consequences
- Module boundaries must be enforced by code review discipline — no automatic enforcement
- All modules share the same Lambda cold start; mitigated by keeping startup code lean
- Extracting modules to independent services later requires interface stability, not rewrite

---

## ADR-002: Puppeteer (HTML/CSS) over LaTeX for PDF Generation

**Date:** 2026-06-04
**Status:** Accepted

### Context
The CLI proof-of-concept generates CVs using LaTeX (lualatex) and cover letters using XeLaTeX. For a multi-tenant SaaS, we need PDF generation that runs in the cloud without a TeX installation.

### Options Considered
**A — LaTeX on Lambda (containerised TeX Live):** Same output quality as CLI, but container image ~1GB, cold starts ~8–12s, complex to maintain.
**B — WeasyPrint (Python):** Renders HTML/CSS to PDF, lightweight Python package, but limited CSS support (no Flexbox/Grid).
**C — Puppeteer (headless Chromium) on Lambda:** Renders full HTML/CSS as a browser would, container image ~400MB, cold starts ~3–5s, full modern CSS support.

### Decision
**C — Puppeteer on Lambda.**

### Rationale
- Full CSS support means CV templates can be designed in any modern web design tool and exported to production
- Removes all LaTeX expertise requirement for template development and maintenance
- Output quality is indistinguishable from LaTeX at screen/print resolution for this use case
- Container image is manageable; cold starts acceptable for an async worker (not in the critical path)
- HTML/CSS templates are version-controlled and diff-readable; LaTeX templates are opaque to web developers

### Consequences
- Fine-grained typographic control (kerning, hyphenation, optical margins) is reduced vs LaTeX
- Templates must be tested in Puppeteer specifically — not all CSS renders identically in all browsers
- Team needs Node.js knowledge for Puppeteer configuration (or use Python puppeteer port `pyppeteer`)

---

## ADR-003: Human-in-the-Loop CAPTCHA (no automated solving)

**Date:** 2026-06-04
**Status:** Accepted

### Context
The auto-submission pipeline is the core product differentiator. reCAPTCHA and similar challenges appear at the point of form submission on most ATS platforms. How to handle this determines the product's automation completeness and legal/compliance posture.

### Options Considered
**A — Third-party CAPTCHA solver (2captcha, Anti-Captcha):** Fully automated, ~$3/1000 solves, sits in a legal grey area on most ToS, modern reCAPTCHA v3 increasingly solver-resistant.
**B — Human-in-the-loop:** Pipeline pauses, user is notified, user opens the pre-filled form in their browser, solves CAPTCHA manually, and clicks Submit.
**C — Avoid CAPTCHA by targeting ATS platforms only:** Most enterprise ATS platforms (Greenhouse, Lever, Ashby) have lighter bot protection at the application layer specifically. Focus automation here.

### Decision
**B + C — Human-in-the-loop, ATS-first targeting.**

### Rationale
- Third-party solvers put Eldar in violation of target job board ToS — this is a business risk that could result in user accounts being banned
- Modern reCAPTCHA v3 uses behavioural signals that third-party solvers cannot reliably reproduce
- The user experience of "review the pre-filled form and click Submit" is still 95% automation — the value proposition holds
- Targeting ATS platforms (Greenhouse, Lever, Ashby) as the primary submission path means CAPTCHA appears less frequently and is more predictable
- Full ToS compliance is a marketing advantage, not a limitation ("Eldar never violates platform terms")

### Consequences
- Pipeline cannot complete without user interaction at CAPTCHA step
- Users must be reachable via notification (email + in-app) to complete submissions
- Session state (pre-filled form) must persist until user returns, up to 4 hours

---

## ADR-004: DynamoDB over RDS for Primary Data Store

**Date:** 2026-06-04
**Status:** Accepted

### Context
Eldar needs to store user profiles, jobs, applications, documents metadata, pipeline state, and billing data. The data model is primarily hierarchical (everything owned by a user) with known access patterns.

### Options Considered
**A — PostgreSQL on RDS:** Full relational queries, ACID transactions, mature ecosystem, but requires instance management, VPC, min ~$50/month, and doesn't scale automatically.
**B — DynamoDB:** Serverless, scales to zero cost when idle, sub-millisecond reads, automatic scaling, no instance management.
**C — Aurora Serverless v2:** Serverless Postgres, but has a minimum ACU cost, slower cold start, more complex than DynamoDB for simple key-value patterns.

### Decision
**B — DynamoDB.**

### Rationale
- All access patterns are known and key-based (user_id → profile, user_id → applications)
- No complex joins needed — all data fetched by partition key
- Serverless: zero cost when no users, automatic scaling as users grow
- Consistent with MyIQ architecture (Ben has deep DynamoDB experience)
- Eliminates VPC requirement for the database tier (simpler infra, lower cold start for Lambda)

### Consequences
- No ad-hoc queries or complex aggregations without a GSI or separate analytics store
- Analytics dashboard (Phase 2) may require a read replica to Redshift or Athena over S3
- Schema migrations are structural DynamoDB table changes (careful attribute naming from day one)

---

## ADR-005: SQS over EventBridge for Pipeline Messaging

**Date:** 2026-06-04
**Status:** Accepted

### Context
Pipeline stages (scrape → evaluate → generate → fill → track) are async. We need a message bus to decouple the API from worker Lambdas.

### Options Considered
**A — SQS (Standard + FIFO):** Simple queue, direct trigger for Lambda, built-in DLQ, retry with visibility timeout.
**B — EventBridge:** Rich event routing, schema registry, cross-account, but more complex for simple linear pipelines.
**C — Step Functions:** Managed workflow with state machine, but adds pricing overhead and complexity for what is a linear pipeline with simple error handling.

### Decision
**A — SQS Standard queues.**

### Rationale
- Pipeline is linear — messages flow one stage forward, not fanned out to multiple consumers
- SQS → Lambda trigger is a first-class AWS integration with built-in retry and DLQ
- Lower cognitive overhead than EventBridge schema registry for a team-of-one or small team
- Cost: SQS is effectively free at Eldar's expected volume (< 1M messages/month)
- Step Functions adds cost per state transition with no benefit over direct SQS orchestration for a 5-stage linear pipeline

### Consequences
- No complex event routing — if future stages need fan-out, migrate queue to EventBridge or SNS
- Message deduplication for FIFO queues adds latency overhead if needed; standard queues acceptable for pipeline stages that are idempotent

---

## ADR-006: Monorepo over Polyrepo

**Date:** 2026-06-04
**Status:** Accepted

### Context
Eldar has three codebases: FastAPI backend, Next.js frontend, AWS CDK infrastructure. How to organise repositories.

### Options Considered
**A — Polyrepo:** Three separate repos (`eldar-api`, `eldar-web`, `eldar-infra`). Independent versioning, separate CI/CD.
**B — Monorepo:** Single repo (`eldar/`) with `apps/api`, `apps/web`, `infra/`. Shared CI/CD configuration.

### Decision
**B — Monorepo.**

### Rationale
- API contract changes and frontend changes always ship together — monorepo makes this atomic
- Infrastructure changes that affect both API and frontend are easier to review as a single PR
- Consistent with the proven MyIQ pattern
- Simpler local development (one `git clone`, one set of GitHub Actions secrets)
- At MVP scale, the complexity of polyrepo tooling (changesets, cross-repo dependency management) is not justified

### Consequences
- CI/CD workflows must be scoped by changed paths to avoid unnecessary builds
- Large monorepos can become slow to clone — not a concern at Eldar's scale
- All engineers have access to all code — acceptable for a small team

---

## ADR-007: Anthropic Claude via AWS Bedrock (not direct Anthropic API)

**Date:** 2026-06-04
**Status:** Accepted

### Context
Eldar uses Claude for job evaluation (fit scoring) and document generation (CV + cover letter). Two options for accessing Claude: direct Anthropic API or AWS Bedrock.

### Options Considered
**A — Direct Anthropic API:** Simpler client, access to latest models immediately on release, single API key.
**B — AWS Bedrock:** AWS-native, IAM-based auth (no API key to manage), consistent with rest of infrastructure, prompt caching supported, but Claude model availability may lag Anthropic direct by days/weeks.

### Decision
**B — AWS Bedrock.**

### Rationale
- IAM auth removes one secret to manage (no `ANTHROPIC_API_KEY` in Secrets Manager)
- Consistent with MyIQ architecture (Ben has Bedrock expertise)
- Prompt caching via Bedrock reduces token costs on repeat profile-based calls
- AWS-to-AWS networking keeps data within the AWS perimeter (relevant for enterprise customers later)
- Model lag is typically days/weeks — acceptable tradeoff for the architecture consistency benefit

### Consequences
- New Claude model versions (e.g., next Sonnet release) available on Bedrock 1–4 weeks after Anthropic direct
- Bedrock pricing has slight overhead vs direct API — offset by prompt caching savings
- If Bedrock support for a critical Claude feature lags, can switch to direct API per-module without system-wide change

# Eldar — AI Job Application SaaS

## What This Is

Eldar is a full-pipeline AI job application SaaS for senior tech professionals. It automates the complete journey: job discovery → fit evaluation → CV + cover letter generation → ATS form auto-fill → human CAPTCHA checkpoint → submitted → tracked.

This CLAUDE.md is for the **Eldar product repo** (`eldar/`). It does not apply to the `ai-job-search` CLI tool.

---

## Repo Structure

```
eldar/
├── apps/
│   ├── api/          FastAPI backend — modular monolith
│   └── web/          Next.js 14 + TypeScript + Tailwind + shadcn/ui
├── infra/            AWS CDK (TypeScript) — 4 stacks
├── .github/          CI/CD — 4-workflow pipeline
└── docs/             PRD, TRD, ADRs
```

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend | FastAPI (Python 3.12), Pydantic, boto3 |
| Frontend | Next.js 14, TypeScript, Tailwind CSS, shadcn/ui |
| Infrastructure | AWS CDK (TypeScript) |
| Database | DynamoDB |
| Storage | S3 |
| Auth | AWS Cognito |
| AI | Anthropic Claude via AWS Bedrock |
| PDF | Puppeteer (headless Chrome on Lambda) |
| Form automation | Playwright |
| Queue | SQS |
| Payments | Stripe |
| CI/CD | GitHub Actions |
| Monitoring | Datadog |

---

## Domain Modules (FastAPI)

The backend is a **modular monolith**. Each module owns its domain completely — its own models, services, repositories, and interface. No cross-module direct database access. Modules communicate through service interfaces only.

| Module | Owns |
|--------|------|
| `auth` | Cognito integration, JWT validation |
| `profile` | User profiles, work history, skills, preferences |
| `jobs` | Job scraping, URL ingestion, job storage |
| `evaluation` | Claude fit scoring against user profile |
| `documents` | CV + cover letter generation, PDF rendering |
| `submissions` | Playwright form automation, ATS adapters |
| `tracking` | Application status, history, notes |
| `billing` | Stripe webhooks, quota enforcement |

---

## SOLID Principles (enforced)

**Single Responsibility:** Each module has one domain. Each class/function has one reason to change.

**Open/Closed:** New ATS adapters (`GreenhouseAdapter`, `LeverAdapter`) implement `IATSAdapter` — no changes to submission core. New CV templates implement `ICVTemplate` — no changes to document service.

**Liskov Substitution:** All ATS adapters are interchangeable. PDF renderer is swappable. Billing gateway is swappable.

**Interface Segregation:** Modules expose minimal interfaces. `IJobScraper`, `ICVGenerator`, `IFormSubmitter`, `IBillingGateway` — callers depend only on what they use.

**Dependency Inversion:** All modules depend on abstractions, never on concrete implementations. Injected via FastAPI's dependency injection.

---

## Pipeline Architecture

Document generation and form submission are **async background workers** (SQS + Lambda). The API never blocks on slow operations. Users see live status updates.

```
POST /jobs/submit-url
  → SQS: job.scrape
  → Worker: scrape + store
  → SQS: job.evaluate
  → Worker: Claude fit score
  → [User approves]
  → SQS: documents.generate
  → Worker: Claude CV + cover letter + Puppeteer PDF
  → [User reviews]
  → SQS: submissions.fill
  → Worker: Playwright fills ATS form
  → ⏸ Paused: user notified to solve CAPTCHA
  → User submits in browser
  → PATCH /applications/:id/submitted
  → tracking.record
```

---

## Application Run (billing unit)

One **Application Run** = evaluate + generate CV + generate cover letter + form fill + submit.

Job evaluation and application tracking are **always free**. A run is consumed when document generation starts.

| Tier | Price | Runs/mo |
|------|-------|---------|
| Free | $0 | 2 |
| Pro | $79 | 20 |
| Power | $149 | Unlimited |

---

## CDK Stacks

| Stack | Contains |
|-------|---------|
| `AuthStack` | Cognito User Pool, Identity Pool |
| `StorageStack` | DynamoDB tables, S3 buckets |
| `ApiStack` | FastAPI Lambda, API Gateway, SQS, Worker Lambdas, Bedrock, Secrets Manager |
| `FrontendStack` | Next.js on Amplify, CloudFront |

---

## ATS Support

**Phase 1 (MVP):** Greenhouse, Lever, Ashby
**Phase 2:** Workday, Taleo, iCIMS
**Phase 3:** LinkedIn Easy Apply, others

CAPTCHA is **always human-in-the-loop**. Never use third-party CAPTCHA solvers. This is both a compliance decision and a reliability decision.

---

## Engineering Rules

1. **No cross-module DB access.** Modules talk through service interfaces, never DynamoDB directly across boundaries.
2. **Async everything slow.** Document generation, PDF rendering, Playwright — all via SQS. API responses must be fast.
3. **Type everything.** Pydantic models for all API contracts. TypeScript strict mode on the frontend. No `any`.
4. **Test the boundaries.** Unit test each module's service layer. Integration test the pipeline end-to-end. No mocking DynamoDB in integration tests — use a real table or DynamoDB Local.
5. **IaC for all infrastructure.** Nothing manually provisioned. CDK is the source of truth.
6. **Secrets in Secrets Manager.** No secrets in environment variables, no secrets in code, no secrets in SSM.
7. **One ATS adapter per file.** `greenhouse.py`, `lever.py`, `ashby.py`. Each implements `IATSAdapter`. Easy to add, easy to maintain, easy to test in isolation.
8. **Document generation is deterministic.** Given the same profile + job posting, the same CV must be produced. No randomness in templates.
9. **Quota enforcement at the service layer, not the API layer.** Billing module exposes `can_consume_run(user_id)` — document service calls it before generating. API layer does not enforce quotas directly.
10. **Claude Code is the development tool.** Reference it by name in any documentation about AI-assisted development on this codebase.

---

## CI/CD (GitHub Actions — 4 workflows)

| Workflow | Trigger | Does |
|----------|---------|------|
| `frontend-ci` | PR touching `apps/web/` | Type check, lint, test (Vitest) |
| `backend-ci` | PR touching `apps/api/` | Lint (ruff), type check (mypy), test (pytest) |
| `infra-ci` | PR touching `infra/` | CDK synth + diff — blocks merge on synth failure |
| `deploy` | Push to `main` | CDK deploy, sequenced: Storage → Auth → Api → Frontend |

---

## Target User

Senior tech professionals (engineers, TPMs, architects, technical leads) with 5–15 years experience, actively job searching at $150K–$400K+ salary levels. Growth strategy: nail this segment first, expand iteratively — do not broaden the product prematurely.

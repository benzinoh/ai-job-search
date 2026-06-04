# Eldar — Technical Requirements Document

**Version:** 0.1 (MVP)
**Date:** 2026-06-04
**Status:** Draft

---

## 1. System Overview

Eldar is a multi-tenant SaaS application deployed on AWS. The backend is a FastAPI modular monolith on Lambda. The frontend is a Next.js 14 application on Amplify. Long-running pipeline stages run as async workers orchestrated by Step Functions Express. Form submission runs on ECS Fargate Spot (zero cold starts). Real-time updates delivered via SSE.

```
                    ┌──────────────────────────────────────────────────────┐
                    │                 AWS Infrastructure                    │
  User Browser      │                                                      │
       │            │  CloudFront + Amplify                                │
       ├────HTTPS──▶│  Next.js 14 (SSR + SSE EventSource)                 │
       │            │                                                      │
       ├────API────▶│  API Gateway (HTTP) → FastAPI Lambda                 │
       │            │        │                                              │
       │            │        ├── DynamoDB (data)                           │
       │            │        ├── S3 (PDFs, docs)                           │
       │            │        ├── Cognito (auth, custom UI)                 │
       │            │        ├── Secrets Manager                           │
       │            │        └── Step Functions Express                    │
       │            │               ├── Lambda: scrape / evaluate / gen    │
       │            │               │     └──▶ Bedrock ILLMClient (Claude) │
       │            │               │     └──▶ Puppeteer Lambda (PDF)      │
       │            │               └── ECS Fargate Spot: Playwright       │
       └────────────└──────────────────────────────────────────────────────┘
```

---

## 2. Tech Stack

### Backend
- **Runtime:** Python 3.12
- **Framework:** FastAPI with Pydantic v2
- **Deployment:** AWS Lambda (ARM64, 1024MB default, workers 2048MB)
- **ORM/DB:** boto3 + custom DynamoDB repository pattern
- **HTTP client:** httpx (async)
- **Testing:** pytest, pytest-asyncio, moto (DynamoDB mock for unit tests), real DynamoDB Local for integration tests

### Frontend
- **Framework:** Next.js 14 (App Router)
- **Language:** TypeScript (strict mode)
- **Styling:** Tailwind CSS + shadcn/ui
- **State management:** React Query (server state) + Zustand (client state)
- **Auth:** AWS Amplify Auth (Cognito)
- **PDF preview:** react-pdf
- **Testing:** Vitest + React Testing Library

### Infrastructure
- **IaC:** AWS CDK v2 (TypeScript)
- **Dev iteration:** `cdk watch` for Lambda hot-deploy (~15s loop). Next.js local dev for frontend. SAM CLI for local Lambda invocation.
- **CI/CD:** GitHub Actions (4-workflow pattern: frontend-ci, backend-ci, infra-ci, deploy)
- **Secrets:** AWS Secrets Manager (no env vars for secrets)
- **Monitoring:** Datadog (dashboards, alerting, MTTR baselines)
- **Tracing:** AWS X-Ray

### AI
- **Provider:** Anthropic Claude via AWS Bedrock (Phase 1 — covered by AWS credits)
- **Abstraction:** `ILLMClient` interface with `BedrockClaudeClient` (active) and `AnthropicDirectClient` (ready). Switch is a one-file change when credits exhaust or model lag becomes a competitive issue.
- **Model:** claude-sonnet-4-6 (document generation), claude-haiku-4-5 (fit scoring, lighter tasks)
- **Prompt caching:** Enabled for profile context (reduces cost ~70% on repeat calls)

### Document Generation
- **Renderer:** Puppeteer (headless Chromium) in a Lambda container image
- **Templates:** HTML/CSS — one template per CV style, one per cover letter style
- **Output:** PDF stored in S3 with pre-signed URL for user download
- **No LaTeX dependency**

### Form Automation
- **Tool:** Playwright (Python)
- **Deployment:** ECS Fargate Spot (persistent container, zero cold starts). Capacity provider strategy: Spot preferred, on-demand fallback. Target: 0.25 vCPU / 0.5GB per task.
- **ATS adapters:** One file per ATS (greenhouse.py, lever.py, ashby.py), each implementing `IATSAdapter`
- **CAPTCHA:** Step Functions execution pauses via callback token. User notified via email + SSE push. User opens pre-filled form, solves CAPTCHA, submits. Webhook triggers callback token return, execution resumes.

### Pipeline Orchestration
- **Tool:** AWS Step Functions Express Workflows
- **Cost:** $0.00001/state transition — ~$0.16/month at 2,000 runs
- **Human gates:** Callback token pattern (`.waitForTaskToken`). Execution pauses on approval gates and CAPTCHA checkpoint.
- **Retry:** Each state configures `Retry` with exponential backoff (3 attempts, 2× multiplier, 30s max interval)
- **Failure handling:** `Catch` on each state routes to a FAILED terminal state with error context. DLQ catches Step Functions execution failures.

### Real-time Updates
- **Protocol:** Server-Sent Events (SSE) via API Gateway HTTP API
- **Endpoint:** `GET /pipeline/{application_id}/stream` — returns `text/event-stream`
- **Flow:** Step Functions state change → EventBridge rule → Lambda → pushes SSE event to open client connection
- **Client:** Browser `EventSource` API — auto-reconnects, no library required

---

## 3. Data Models

### User
```
PK: USER#{user_id}
SK: PROFILE
Fields: email, name, created_at, plan (free|pro|power), runs_used, runs_reset_at
```

### Profile
```
PK: USER#{user_id}
SK: PROFILE#DETAIL
Fields: work_history[], skills[], education[], certifications[], preferences{}, behavioral_profile{}
```

### Job
```
PK: USER#{user_id}
SK: JOB#{job_id}
Fields: url, title, company, location, posting_text, scraped_at, status (scraped|evaluated|applied|closed)
GSI: BY_STATUS (for dashboard queries)
```

### Application
```
PK: USER#{user_id}
SK: APPLICATION#{application_id}
Fields: job_id, cv_s3_key, cover_letter_s3_key, fit_score{}, status, submitted_at, notes, pipeline_state
```

### Pipeline State
```
PK: USER#{user_id}
SK: PIPELINE#{application_id}
Fields: stage (scraping|evaluating|generating|filling|awaiting_captcha|submitted|failed), updated_at, error
TTL: 7 days after terminal state
```

---

## 4. API Design

All endpoints under `/api/v1/`. Authenticated via Cognito JWT (Authorization header).

### Core Endpoints

```
POST   /jobs                          Ingest a job URL → triggers scrape pipeline
GET    /jobs                          List user's jobs with status
GET    /jobs/{job_id}                 Get job detail + fit score

POST   /applications                  Start application pipeline for a job
GET    /applications                  List user's applications
GET    /applications/{id}             Get application detail + pipeline state
PATCH  /applications/{id}/status      Update status (screening/interview/offer/rejected)
POST   /applications/{id}/submitted   Mark as submitted after human CAPTCHA completion

GET    /documents/{application_id}/cv             Pre-signed S3 URL for CV PDF
GET    /documents/{application_id}/cover-letter   Pre-signed S3 URL for cover letter PDF

GET    /profile                       Get user profile
PUT    /profile                       Update user profile

GET    /billing/subscription          Current plan + usage
POST   /billing/portal                Stripe customer portal URL
```

### WebSocket
```
WS /ws/pipeline/{application_id}     Real-time pipeline stage updates
```

---

## 5. Pipeline Stage Contracts

Each SQS message follows this contract:

```json
{
  "stage": "documents.generate",
  "user_id": "usr_abc123",
  "application_id": "app_xyz789",
  "job_id": "job_def456",
  "attempt": 1,
  "initiated_at": "2026-06-04T14:30:00Z"
}
```

Workers update DynamoDB pipeline state on start, completion, and failure. The API polls pipeline state for WebSocket pushes.

---

## 6. ATS Adapter Interface

```python
class IATSAdapter(ABC):
    @abstractmethod
    async def detect(self, url: str) -> bool:
        """Return True if this adapter handles the given ATS URL."""

    @abstractmethod
    async def discover_fields(self, page: Page) -> list[FormField]:
        """Return all form fields on the application page."""

    @abstractmethod
    async def fill(self, page: Page, profile: UserProfile, documents: GeneratedDocuments) -> FillResult:
        """Fill all form fields. Return FillResult with captcha_present flag."""
```

Phase 1 implementations: `GreenhouseAdapter`, `LeverAdapter`, `AshbyAdapter`

---

## 7. Document Generation

### Flow
1. Worker receives `documents.generate` SQS message
2. Calls Claude (claude-sonnet-4-6) with profile + job posting → structured JSON (CV sections, cover letter paragraphs)
3. Renders JSON into HTML using Jinja2 template
4. Puppeteer converts HTML to PDF
5. PDF uploaded to S3 (`{user_id}/applications/{application_id}/cv.pdf`)
6. DynamoDB updated with S3 keys
7. Pipeline state advanced to `awaiting_review`

### CV Template Contract
```python
@dataclass
class CVTemplateData:
    name: str
    contact: ContactInfo
    profile_statement: str
    core_competencies: list[Competency]
    experience: list[ExperienceEntry]
    education: list[EducationEntry]
    certifications: list[str]
    projects: list[Project] | None
```

### Cost Controls
- Prompt caching: system prompt + profile context cached (saves ~70% on repeat calls)
- Haiku for fit scoring, Sonnet for document generation
- Target: < $0.50 Claude API cost per complete application run

---

## 8. Security Requirements

| Requirement | Implementation |
|-------------|---------------|
| Authentication | Cognito JWT, validated on every request |
| Authorization | Every DynamoDB query partitioned by `USER#{user_id}` — no cross-user data access possible at query level |
| Secrets | AWS Secrets Manager only — no secrets in env vars, SSM, or code |
| PDFs in S3 | Private bucket, access via pre-signed URLs (15-minute expiry) |
| HTTPS | All traffic via CloudFront (TLS 1.2+) |
| Input validation | All inputs validated via Pydantic v2 models — no raw dict access to user input |
| OWASP Top 10 | Addressed by framework (FastAPI/Next.js XSS protection, no SQL, Pydantic validation) |
| Data isolation | Cognito user pool per environment (dev/staging/prod) |
| IaC security | Checkov scans CDK synth output in infra-ci workflow |

---

## 9. Performance Requirements

| Metric | Target | Notes |
|--------|--------|-------|
| API response time (p95) | < 200ms | For non-pipeline endpoints |
| Job scraping | < 10s | From URL submission to stored posting |
| Fit evaluation | < 15s | Claude Haiku call + DynamoDB write |
| Document generation | < 45s | Claude Sonnet + Puppeteer PDF |
| Form fill (pre-CAPTCHA) | < 60s | Playwright navigation + field fill |
| Pipeline total (p95) | < 3 min | URL to pre-filled form ready for CAPTCHA |
| Frontend TTFB | < 500ms | CloudFront + Next.js SSR |
| Frontend LCP | < 2.5s | Core Web Vitals target |

---

## 10. Reliability & Observability

- **SQS retry:** All worker queues have DLQ. Failed jobs retried 3× with exponential backoff. After 3 failures, application pipeline state set to `failed`, user notified.
- **Lambda timeouts:** Scraping: 30s. Evaluation: 30s. Document generation: 120s. Form fill: 180s.
- **Datadog:** Dashboards for pipeline stage latency, error rates, and quota consumption. Alerts on p95 latency > 2× target or error rate > 1%.
- **X-Ray:** Full trace per application pipeline for debugging.

---

## 11. Cost Estimates (at 100 Pro subscribers, 20 runs/month each = 2,000 runs/month)

| Service | Estimate/month | AWS Credits? |
|---------|---------------|-------------|
| Lambda (API + workers) | ~$15 | ✅ Yes |
| DynamoDB | ~$8 | ✅ Yes |
| S3 (PDFs + storage) | ~$5 | ✅ Yes |
| Bedrock (Claude) | ~$400 (avg $0.20/run) | ✅ Yes |
| Step Functions Express | ~$0.16 | ✅ Yes |
| ECS Fargate Spot (Playwright) | ~$3 | ✅ Yes |
| CloudFront + Amplify | ~$20 | ✅ Yes |
| Cognito | $0 (free up to 50k MAU) | ✅ Yes |
| Datadog | ~$35 | ❌ No |
| **Total infra** | **~$486/month** | |
| **Revenue (100 × $79)** | **$7,900/month** | |
| **Gross margin** | **~94%** | |
| **With AWS credits** | **Bedrock + all AWS = ~$54/month net** | |

Target: keep total infra cost below 8% of MRR at all subscriber levels.

---

## 12. Tradeoffs and Known Limitations

| Decision | Tradeoff |
|----------|---------|
| Puppeteer over LaTeX | Loses fine-grained typography control; gains zero infrastructure dependency and maintainable HTML templates |
| Lambda for Playwright | Cold starts can add 5–15s to form fill time; mitigated by provisioned concurrency on the submission worker if needed |
| Modular monolith over microservices | Slightly harder to scale individual modules independently; gained: faster initial build, easier local development, simpler debugging |
| Human CAPTCHA only | Not 100% automated; gained: full ToS compliance, zero grey-area tools, reliable on all ATS platforms |
| DynamoDB over RDS | No complex joins or ad-hoc queries; gained: serverless, zero maintenance, automatic scaling, cost-effective at scale |
| SQS over EventBridge | Less rich event routing; gained: simpler mental model, lower cost, direct trigger semantics for pipeline stages |

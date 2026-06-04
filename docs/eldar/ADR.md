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
- **ILLMClient abstraction (see ADR-012) makes this switch a one-file change**

---

## ADR-008: Step Functions Express over Raw SQS for Pipeline Orchestration

**Date:** 2026-06-04
**Status:** Accepted

### Context
The Eldar pipeline has 6 sequential stages with human approval gates (fit evaluation, document review, CAPTCHA). Each stage runs asynchronously. We need an orchestration mechanism that handles retries, failures, and pause/resume at human checkpoints.

### Options Considered
**A — Raw SQS queues:** Each stage publishes to the next queue. Pipeline state tracked manually in DynamoDB.
**B — Step Functions Express Workflows:** Managed state machine. Each stage is a state. Human gates use the callback token pattern (task heartbeat).
**C — Step Functions Standard Workflows:** Similar to Express but $0.025/state transition vs $0.00001 — 2500× more expensive for high-volume orchestration.

### Decision
**B — Step Functions Express Workflows.**

### Rationale
- Pipeline state is inherent to a state machine — modelling it as DynamoDB records adds accidental complexity
- Callback token pattern handles human approval gates natively (pause execution, resume on token return)
- Visual debugger in AWS Console shows each execution's path — critical for diagnosing "why did this user's submission fail at 2am"
- Built-in retry with exponential backoff per state — eliminates hand-rolled retry logic in each worker
- Cost: $0.00001/transition × 8 transitions × 2,000 runs/month = **$0.16/month** — negligible
- Covered by AWS credits

### Consequences
- Step Functions Express has a 5-minute maximum execution duration — sufficient for pipeline (target p95 < 3 min). Upgrade to Standard if pipeline ever exceeds this.
- Each execution generates CloudWatch logs — minor log storage cost
- Human approval gates require callback tokens to be stored and retrieved — handled by the submissions module

---

## ADR-009: ECS Fargate Spot for Playwright Submission Worker

**Date:** 2026-06-04
**Status:** Accepted

### Context
The form submission worker runs Playwright (headless Chromium) to fill ATS application forms. This is the most reliability-critical step in the pipeline — a failure here means the application was not submitted. Deployment options are Lambda (container image) or ECS Fargate.

### Options Considered
**A — Lambda container image:** Playwright + Chromium bundled (~400MB). Cold starts of 8–15s. Max 15-minute timeout.
**B — ECS Fargate on-demand:** Persistent container, zero cold starts. ~$0.04/vCPU-hour.
**C — ECS Fargate Spot:** Same as on-demand but ~70% cheaper via Spot pricing. ~$0.013/vCPU-hour. Can be interrupted (rare for short tasks).

### Decision
**C — ECS Fargate Spot with on-demand fallback via capacity provider strategy.**

### Rationale
- Cold starts on Lambda cause 8–15s delays at the moment the user is waiting for their form to be filled — worst possible UX moment
- Fargate containers stay warm between runs — zero cold start
- Spot interruption risk is low for tasks under 2 minutes (form fill target: 60s)
- Capacity provider strategy: Fargate Spot preferred, fall back to Fargate on-demand if Spot unavailable — zero reliability loss
- Fargate Spot with 0.25 vCPU + 0.5GB at ~1 warm container: **~$2–3/month** covered by AWS credits
- Scale to 0 when no active submissions — near-zero cost at low volume

### Consequences
- ECS cluster adds complexity to CDK stack vs a Lambda function
- Container image must include Playwright + Chromium — same image used in Lambda approach, no change
- Spot can be interrupted mid-task — Step Functions will retry the filling stage automatically (ATS forms are idempotent — filling again is safe)

---

## ADR-010: Server-Sent Events over WebSocket for Real-time Pipeline Updates

**Date:** 2026-06-04
**Status:** Accepted

### Context
Users need live status updates as the pipeline progresses through stages (scraping → evaluating → generating → filling → awaiting CAPTCHA). Updates flow server → client only.

### Options Considered
**A — API Gateway WebSocket API:** Bidirectional. Requires connect/disconnect/default route management. $0.00025/connection-hour.
**B — Server-Sent Events (SSE) via HTTP API Gateway:** One-directional. Uses browser's native `EventSource` API. Standard HTTP streaming.
**C — Polling:** Client polls `/applications/:id/status` every 2s. Simple but wasteful and adds latency to status display.

### Decision
**B — Server-Sent Events.**

### Rationale
- Pipeline updates are server → client only — bidirectional WebSocket protocol is unnecessary complexity
- `EventSource` is native to all modern browsers — no client library required
- Auto-reconnects on disconnect without application code
- Standard HTTP — works through proxies, load balancers, and API Gateway HTTP API without special configuration
- Connection cost is negligible vs WebSocket hourly charges
- Simpler to implement, test, and debug than WebSocket connection lifecycle management

### Consequences
- SSE connections are long-lived HTTP responses — Lambda timeout (30s for API functions) requires periodic heartbeat or chunked streaming pattern. Use a dedicated SSE Lambda with longer timeout (300s) or deliver events via DynamoDB Streams → Lambda → SSE push.
- One browser tab = one SSE connection per active pipeline — acceptable at MVP scale

---

## ADR-011: Cognito with Custom UI over Hosted Cognito UI or Clerk

**Date:** 2026-06-04
**Status:** Accepted

### Context
Eldar needs user authentication. Options evaluated: hosted Cognito UI, Cognito with custom frontend components, or Clerk (third-party auth service).

### Options Considered
**A — Hosted Cognito UI:** AWS-managed, minimal setup. But severely limited customisation — looks generic and dated out of the box.
**B — Clerk:** Best-in-class auth UX, pre-built Next.js components, social login. $25/month minimum, not covered by AWS credits.
**C — Cognito + custom UI (Amplify Auth + Next.js components):** AWS-native backend, fully custom frontend. Free under AWS credits. Full UX control.

### Decision
**C — Cognito with custom-built UI components.**

### Rationale
- Cognito is free up to 50,000 MAU under AWS free tier, and covered by AWS credits — $0 cost at MVP
- Clerk costs $25/month ($300/year) from day one regardless of user count — real cost when pre-revenue
- Hosting auth on a third-party service (Clerk) introduces a critical-path dependency; a Clerk outage means users cannot log in
- Cognito + Amplify Auth SDK + custom Next.js sign-in/sign-up components delivers professional UX with ~2 days frontend work vs 30 minutes for Clerk — acceptable tradeoff at MVP
- Both Cognito and Clerk have 99.9% SLA — reliability is equivalent

### Consequences
- 2 extra days of frontend work for custom auth UI vs Clerk
- Custom UI must handle all edge cases (password reset, email verification, MFA) — manageable with Amplify Auth SDK
- If user growth exceeds 50,000 MAU, Cognito at $0.0055/MAU is still cheaper than Clerk's per-MAU pricing

---

## ADR-012: ILLMClient Abstraction for AI Provider Portability

**Date:** 2026-06-04
**Status:** Accepted

### Context
Eldar uses Anthropic Claude for job evaluation and document generation. Two access paths exist: AWS Bedrock (credits apply, 1–4 week model lag) and direct Anthropic API (no credits, immediate model access). The optimal choice may change over time as credits are consumed and new models release.

### Decision
Abstract Claude access behind an `ILLMClient` interface. Use Bedrock today. Switch to direct API when AWS credits are exhausted or model lag becomes a competitive issue — requiring a change to one file, not eight modules.

```python
class ILLMClient(ABC):
    @abstractmethod
    async def generate(self, messages: list[Message], system: str, model: LLMModel) -> LLMResponse: ...

    @abstractmethod
    async def generate_structured(self, messages: list[Message], schema: type[T]) -> T: ...

class BedrockClaudeClient(ILLMClient): ...   # Active now
class AnthropicDirectClient(ILLMClient): ...  # Ready to switch
```

### Rationale
- AWS credits offset ~$400/month in Claude API costs during the MVP phase — Bedrock is effectively free
- When credits are exhausted, direct Anthropic API enables access to the latest models immediately on release
- The abstraction costs one interface file and one concrete class per provider — zero ongoing maintenance overhead
- SOLID Liskov Substitution: both clients are interchangeable; callers never know which is active

### Consequences
- The `ILLMClient` interface must be stable — adding methods requires updating both concrete classes
- Model enum (`LLMModel`) must map to correct model IDs per provider (e.g., `claude-sonnet-4-6` has different IDs on Bedrock vs Anthropic direct)

---

## ADR-013: CDK Watch over SST for Development Iteration Speed

**Date:** 2026-06-04
**Status:** Accepted

### Context
Lambda development iteration speed is a known pain point — full `cdk deploy` takes 2+ minutes. SST (Serverless Stack Toolkit) was evaluated as a faster alternative.

### Options Considered
**A — SST v3 (Ion):** Fast Lambda hot reload (~100ms). But SST v3 moved from CDK to Pulumi/OpenTofu — a fundamentally different IaC paradigm. Ben's CDK expertise does not transfer. SST Inc is a startup — vendor risk for a mission-critical build tool.
**B — SST v2:** Built on CDK, familiar constructs. But v2 is in maintenance mode — no new features, security patches only.
**C — `cdk watch` (built-in CDK v2):** Monitors source files, triggers incremental deployments (~15s for Lambda code changes). Zero new tools, zero new vendor, full CDK compatibility.

### Decision
**C — `cdk watch` for development, standard `cdk deploy` for production.**

### Rationale
- `cdk watch` is part of CDK v2 — already installed, no new dependency
- 15s iteration loop is a significant improvement over 2-min full deploy, sufficient for productive Lambda development
- SST v3's paradigm shift (CDK → Pulumi) means re-learning IaC primitives with no credit for existing CDK knowledge
- Vendor dependency on SST Inc for a mission-critical build tool is not justified when a built-in alternative exists
- AWS credits cover all CDK-managed resources regardless of whether SST is used

### Consequences
- `cdk watch` is ~15s vs SST's ~100ms — meaningful difference for rapid UI iteration; mitigated by Next.js local dev (Amplify mock) for frontend work
- Local Lambda invocation still requires `aws lambda invoke` or SAM CLI for unit testing Lambda logic locally

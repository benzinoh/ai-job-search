# Eldar SaaS — Brainstorming Design Spec
*2026-06-04*

## Product Summary

**Eldar** is an AI-powered job application SaaS for senior tech professionals (5–15 years experience). It automates the full application pipeline: job discovery → fit evaluation → CV + cover letter generation → ATS form auto-fill → human CAPTCHA checkpoint → submitted → tracked.

The core product promise is the complete end-to-end pipeline. Partial automation is not compelling enough. The product only ships when the full pipeline is reliable.

## Target User

**MVP:** Senior tech professionals — engineers, TPMs, architects, technical leads — actively job searching. High income, high willingness to pay, complex applications, targeting senior roles at tech companies.

**Growth strategy:** Facebook/Spotify model — nail one segment first, create measurable baselines, then expand iteratively to adjacent segments (non-tech professionals, recent graduates, international markets).

## Architecture

### Repository
Monorepo (`eldar/`) — same structure as MyIQ:
- `apps/api/` — FastAPI backend (modular monolith)
- `apps/web/` — Next.js 14 + TypeScript + Tailwind + shadcn/ui
- `infra/` — AWS CDK (TypeScript), 4 stacks
- `.github/` — 4-workflow CI/CD (frontend-ci, backend-ci, infra-ci, deploy)

### Backend Architecture: Modular Monolith → Microservices
FastAPI with 8 domain modules, strict internal boundaries enforced from day one. Single deployable for MVP; modules extract to independent services as load demands without rewrite.

**Domain modules:**
| Module | Responsibility |
|--------|---------------|
| `auth` | Cognito integration, JWT validation, session management |
| `profile` | Work history, skills, preferences, target sectors |
| `jobs` | Job scraping, URL ingestion, feed storage |
| `evaluation` | Claude-powered fit scoring (5 dimensions) |
| `documents` | CV + cover letter generation, Puppeteer PDF rendering |
| `submissions` | Playwright form automation, ATS adapters, CAPTCHA coordination |
| `tracking` | Application status, history, notes, analytics |
| `billing` | Stripe webhooks, quota enforcement, plan management |

### Pipeline (Async via SQS)
```
User adds job URL
  → jobs: scrape + store
  → evaluation: Claude fit score (async Lambda)
  → User approves
  → documents: generate CV + cover letter → Puppeteer PDF (async Lambda)
  → User reviews documents
  → submissions: Playwright fills form (async Lambda)
  → ⏸ CAPTCHA: user notified, solves in browser
  → tracking: record outcome
```

Document generation and form filling are background Lambda workers. Users see live status updates. CAPTCHA is always a human checkpoint — never bypassed.

### Infrastructure (CDK — 4 Stacks)
- **AuthStack** — Cognito User Pool + Identity Pool
- **StorageStack** — DynamoDB (user data, jobs, applications) + S3 (generated PDFs, profile docs)
- **ApiStack** — FastAPI on Lambda + API Gateway + SQS queues + Worker Lambdas + Bedrock access + Secrets Manager
- **FrontendStack** — Next.js on Amplify + CloudFront CDN

### Key Technical Decisions
- **PDF generation:** Puppeteer (headless Chrome) on Lambda. Renders HTML/CSS templates to PDF. No LaTeX dependency — templates are maintainable HTML/CSS.
- **CAPTCHA handling:** Human-in-the-loop always. Pipeline pauses, user is notified, user solves CAPTCHA in-browser, then submits. Fully compliant with all job board ToS.
- **ATS strategy:** Target major enterprise ATS platforms with predictable form structure. Phase 1: Greenhouse, Lever, Ashby. Phase 2: Workday, Taleo. Phase 3: LinkedIn Easy Apply.

## Monetization

**Unit of sale:** One Application Run = evaluate + generate CV + generate cover letter + form fill + submit. Job evaluation and application tracking are always free.

| Tier | Price | Runs/month | Auto-submission |
|------|-------|-----------|----------------|
| Free | $0 | 2 | No |
| Pro | $79/mo | 20 | Yes (Greenhouse, Lever, Ashby) |
| Power | $149/mo | Unlimited | Yes (all ATS platforms) |

**LTV estimate:** Average active job search = 2–4 months. Pro LTV = $158–$316.

## SOLID Principles Application
- **S:** Each module has a single domain responsibility
- **O:** New ATS adapters added without modifying core submission logic; new CV templates without modifying document module
- **L:** ATS adapters are interchangeable; PDF renderer is swappable (Puppeteer → WeasyPrint if needed)
- **I:** Each module exposes a clean service interface; no cross-module direct DB access
- **D:** Modules depend on abstractions (IJobScraper, ICVGenerator, IFormSubmitter, IBillingGateway) not concretions

## Out of Scope for MVP
- Chrome extension
- LinkedIn Easy Apply automation
- Interview prep module
- Third-party CAPTCHA solving
- Team/enterprise accounts
- Mobile app

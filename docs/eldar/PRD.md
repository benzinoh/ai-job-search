# Eldar — Product Requirements Document

**Version:** 0.1 (MVP)
**Date:** 2026-06-04
**Status:** Draft

---

## 1. Problem Statement

Senior tech professionals spend 2–4 hours per job application: researching the company, tailoring their CV, writing a cover letter, and filling out the application form. Most of this work is mechanical and repeatable — it does not require human creativity. Yet no product fully automates it. Existing tools stop at document generation; none close the loop by submitting the application.

Eldar closes the loop.

---

## 2. Product Vision

> Eldar turns a job URL into a submitted application in under 5 minutes, with the candidate reviewing and approving each step.

The product does not replace human judgment. The candidate evaluates the fit score, approves the documents, and solves the CAPTCHA before submission. Eldar removes everything else.

---

## 3. Target User

### MVP Persona: Senior Tech Professional

- **Role:** Software engineer, technical program manager, architect, engineering lead
- **Experience:** 5–15 years
- **Salary target:** $150K–$400K+ (US/Canada)
- **Job search frequency:** Once every 1–3 years
- **Pain:** Applying to 20–50 roles over 2–4 months; each application takes 2–4 hours of mostly-mechanical work
- **Willingness to pay:** High — one successful hire from Eldar represents a $20K–$80K salary uplift

### Growth Strategy

MVP targets senior tech only. Phase 2 expands to adjacent tech roles (IC to mid-level). Phase 3 opens to non-tech white-collar professionals. Facebook/Spotify model: nail one segment before scaling.

---

## 4. Core User Journey (MVP)

```
1. User signs up and builds profile
   → Work history, skills, certifications, preferences, target sectors
   → Upload existing CV (optional, for import)

2. User adds a job
   → Paste a job URL or enter company + role manually
   → Eldar scrapes and stores the posting

3. Eldar evaluates fit
   → Claude scores against profile: skills match, experience match,
     behavioral fit, location, career alignment
   → User sees score + strengths + gaps

4. User approves application
   → "Apply to this role" triggers document generation

5. Eldar generates documents
   → Tailored CV (HTML → PDF via Puppeteer)
   → Tailored cover letter (HTML → PDF)
   → Both available for review and download

6. User reviews and optionally edits
   → In-browser document preview
   → Edit specific sections if needed

7. Eldar fills the form
   → Playwright navigates to the ATS form
   → All fields filled automatically: name, contact, experience, documents uploaded
   → Pauses at CAPTCHA

8. User completes CAPTCHA and submits
   → Notification sent to user (email + in-app)
   → User clicks link → sees pre-filled form → solves CAPTCHA → hits Submit

9. Eldar records the outcome
   → Application status: Applied
   → Tracks: date submitted, role, company, documents used
   → User can update status as it progresses (screening, interview, offer, rejected)
```

---

## 5. Feature Requirements

### P0 — Must ship for MVP

| Feature | Description |
|---------|-------------|
| User auth | Sign up, login, password reset via Cognito |
| Profile builder | Work history, skills, education, preferences (multi-step form) |
| Job ingestion | Paste URL → scrape and store job posting |
| Fit evaluation | Claude scores job against profile, returns structured result |
| CV generation | Tailored HTML/CSS CV → PDF via Puppeteer |
| Cover letter generation | Tailored cover letter → PDF via Puppeteer |
| Document preview | In-browser PDF preview before submission |
| Form auto-fill | Playwright fills Greenhouse, Lever, Ashby forms |
| CAPTCHA handoff | Notifies user, presents filled form, waits for human submit |
| Application tracker | Status board: Applied / Screening / Interview / Offer / Rejected |
| Billing | Stripe subscription: Free / Pro / Power |
| Quota enforcement | Block document generation when run quota exceeded |
| Pipeline status | Live status updates as pipeline stages complete |

### P1 — High value, post-MVP

| Feature | Description |
|---------|-------------|
| Document editor | Rich text editing of generated CV/cover letter before submission |
| Job scraper / feed | Scheduled scraping of job boards by search query |
| Email notifications | Status updates and CAPTCHA prompts via email |
| Interview prep | AI-generated STAR examples and likely questions per role |
| Analytics dashboard | Application funnel, response rates, time-to-response |
| LinkedIn profile import | Import work history from LinkedIn instead of manual entry |

### P2 — Future

| Feature | Description |
|---------|-------------|
| Workday / Taleo ATS adapters | Expand ATS coverage beyond Phase 1 |
| Chrome extension | 1-click apply from any job board |
| Team accounts | Recruiter/coach access to client applications |
| Salary benchmarking | Compensation data alongside fit score |
| Referral network | Match user to connections at target companies |

---

## 6. Success Metrics

### MVP Launch Criteria (before public launch)
- [ ] Full pipeline completes end-to-end without errors for 3 distinct ATS platforms
- [ ] PDF output quality matches or exceeds current LaTeX output (visual review)
- [ ] Pipeline latency: job URL → documents available < 60 seconds (p95)
- [ ] Form fill accuracy: all required fields populated correctly on first attempt ≥ 95%
- [ ] Zero data leakage between users (pen test + manual audit)
- [ ] Stripe billing enforces quota correctly

### Product-Market Fit Signals (first 90 days post-launch)
- **Weekly active users:** ≥ 40% of paid subscribers use the product at least once per week
- **Retention:** ≥ 60% of Pro subscribers renew after month 1
- **Completion rate:** ≥ 70% of started pipelines reach "submitted" status
- **NPS:** ≥ 50 from paying users
- **Support tickets on core pipeline:** < 5% of applications generate a support request

### Revenue Milestones
- Month 3: 50 paying subscribers ($3,950/mo)
- Month 6: 200 paying subscribers ($15,800/mo)
- Month 12: 500 paying subscribers ($39,500/mo)

---

## 7. Out of Scope (MVP)

- Third-party CAPTCHA solving
- Chrome extension
- LinkedIn Easy Apply automation
- Interview scheduling
- Mobile app
- Team/enterprise accounts
- Non-English job markets
- Salary negotiation tooling

---

## 8. Constraints

- **CAPTCHA:** Always human-in-the-loop. No exceptions. This is a compliance and reliability constraint.
- **Data privacy:** User profile data and generated documents are private to the user. No data shared across users or used to train models without explicit consent.
- **ATS ToS:** Form automation must not violate ATS platform terms of service. Target platforms that permit programmatic form interaction.
- **Claude API costs:** Document generation uses Claude API credits. Cost per application run must remain below $0.50 to maintain margin at Pro tier.

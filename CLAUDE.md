# Job Application Assistant for Benjamin Nwokoye

## Role
This repo is a job application workspace. Claude acts as a career advisor and application assistant for Benjamin Nwokoye, helping with:
1. **Job fit evaluation** - Assess job postings against your profile (skills, experience, behavioral traits)
2. **CV tailoring** - Adapt existing CV templates (LaTeX/moderncv) to target specific roles
3. **Cover letter writing** - Draft targeted cover letters using existing templates (LaTeX)
4. **Interview preparation** - Prepare answers, questions, and talking points for interviews
5. **Career strategy** - Advise on positioning and personal branding

## Candidate Profile

### Identity
- **Name:** Benjamin Nwokoye
- **Location:** New Jersey, USA (open to in-person, hybrid, and remote; US, Canada, and remote-first globally)
- **LinkedIn:** https://www.linkedin.com/in/benjamin-nwokoye/
- **GitHub:** https://github.com/benzinoh
- **Website:** bennwokoye.com
- **Languages:** English (native)
- **Status:** Available — left WIS International May 2026
- **LinkedIn headline:** "Lead AI Agent Architect · LLM Systems · Conversational AI · Enterprise Cloud · AWS Community Builder"

### Education
- **BSc in Computer Science** — institution not listed on CV

### Professional Experience
- **Senior Technical Program Manager (AI Platform & Cloud Infrastructure)** (11/2024 – 05/2026) - **WIS International** (Toronto, Canada)
  - Built the SRE function from scratch: on-call rotation, SEV classification (SEV 1–4), incident management SOPs, post-mortem process; reduced MTTR by 50%
  - Designed and deployed AI-assisted CI/CD pipelines on AWS and Azure using GitHub Actions and GitHub Copilot AI; cut release cycle time by 60%
  - Rolled out Datadog observability across multi-cloud environments; operationalized Snyk/SonarQube vulnerability scanning inside CI/CD
  - Directed IAM, RBAC, and SSO lifecycle governance; delivered audit readiness for SOC 2, ISO, and GDPR
  - Delivered 40% infrastructure cost reduction through right-sizing and auto-scaling enforcement
  - Embedded partner to the CTO: translated infrastructure complexity into executive-level decision frameworks

- **Senior Project Manager (AWS Cloud Platform Programs)** (05/2023 – 11/2024) - **Caylent** (AWS Premier Tier Partner, Ontario, Canada)
  - Primary technical advisor to enterprise customers; 98% on-time delivery and <2% budget variance across a multi-client portfolio
  - Directed 17+ VM workload migrations, EKS implementations, and GitHub CI/CD pipeline design; cut CI/CD cycle time by 60%
  - Established program governance, risk registers, and escalation frameworks; improved delivery quality by 67%

- **IT Project Manager (Cloud Infrastructure)** (01/2022 – 05/2023) - **Kubra Data Transfer** (Mississauga, Canada)
  - Led AWS legacy migration (EC2, RDS, S3, Lambda) under Well-Architected guidance; delivered 45% infrastructure cost reduction
  - Drove Terraform module standardization, cutting IaC configuration drift by 75%
  - 97% risk mitigation success rate across globally distributed teams; average 8-week delivery cycles

- **Software Engineer → Principal Program Manager** (2012 – 2021) - **Standards Organisation of Nigeria (SON)** (Lagos, Nigeria)
  - Led end-to-end digitization of the SONCAP conformity certification platform and integration with the national trade portal (trade.gov.ng); 20,000+ daily transactions
  - Architected cross-agency data exchange systems, document management workflows, and national-scale security and digital policy frameworks
  - ISO 9001:2015 Lead Auditor: built and conducted QMS audit frameworks across financial, manufacturing, retail and enterprise organizations

- **Co-Founder & Principal Engineer** (02/2015 – 01/2022) - **PyCloud Solutions** (Toronto, Canada)
  - Scaled to $500K+ ARR delivering AWS cloud transformation programs; achieved AWS Select Tier Partner status
  - Deployed AWS Control Tower multi-account landing zones; architected reusable Terraform modules cutting deployment time by 60%

### Live Project
- **MyIQ** (app.bennwokoye.com) — Production GenAI SaaS on AWS, sole engineer. Full-stack serverless LLM application: FastAPI (Python 3.12) on Lambda, React 18 + TypeScript on Amplify, AWS CDK across 4 stacks, Cognito, DynamoDB, Bedrock (Claude), Stripe webhooks, Secrets Manager. 118 passing tests (pytest + moto + Vitest), 4-workflow GitHub Actions CI/CD with CDK synth/diff gating, Checkov IaC scanning. Freemium model with Stripe subscription lifecycle management.

### Technical Skills
- **Primary:** AWS (Lambda, API Gateway, Cognito, DynamoDB, Amplify, Bedrock, Secrets Manager, CloudWatch, S3, EC2, EKS, RDS, Route 53), AWS CDK (TypeScript), Terraform, GitHub Actions, Python (FastAPI, Pydantic, boto3, pytest, moto), TypeScript (CDK, React 18)
- **Secondary:** Microsoft Azure, Azure DevOps, Datadog, Snyk, SonarQube, Checkov, SQL, Bash
- **Domain:** AI/LLM engineering, GenAI SaaS architecture, agentic pipeline design, prompt engineering (CoT, few-shot, structured output), Strands Agents, MLOps fundamentals, SRE, cloud platform programs, CI/CD modernization
- **Software:** Jira, Confluence, Datadog, GitHub, Slack, Stripe API, AWS Control Tower, AWS Well-Architected

### Certifications
- **AWS Solutions Architect**
- **AWS Developer Associate**
- **AWS AI Practitioner**
- **Microsoft Azure (AZ-900)**
- **HashiCorp Terraform Associate**
- **PMP** (Project Management Professional)
- **CBAP** (Certified Business Analyst Professional)
- **CSM** (Certified Scrum Master)
- **ISO 9001:2015 Lead Auditor**

### Awards
- AWS Community Builder

### Behavioral Profile
- **Builder-Operator** — ships production code (MyIQ sole engineer) and runs enterprise delivery programs simultaneously
- **Thrives in ambiguity** — self-described strength; builds from zero (SRE function, SONCAP, PyCloud)
- **CTO-embedded partner** — prefers operating close to technical leadership, not above engineering orgs
- **Strengths:** Technical credibility, stakeholder translation, eliminating blockers before they become outages, 0-to-1 program builds
- **Growth areas:** Consumer product experience (deep cloud/AI/government background); prefers to frame as "infrastructure and reliability depth that product-first teams lack"
- **Thrives in:** Fast-moving environments, high ambiguity, technical depth valued, proximity to engineering leadership

### What Excites You
- Building production AI systems end-to-end, not just advising on them
- Solving cloud infrastructure and reliability problems at enterprise scale
- Translating AI/cloud complexity into business outcomes for executive stakeholders

### Target Sectors
- AI / Cloud companies: Anthropic, AWS, Microsoft, Google, Meta, Palantir, Databricks
- Enterprise SaaS: Salesforce, ServiceNow, Workday, Datadog, Snowflake
- AWS Premier/Advanced Partners: Caylent, Slalom, Accenture AWS practice
- FinTech / RegTech: companies with complex cloud compliance requirements

### Deal-breakers
- None stated

### Salary Expectations
- $180,000 USD base minimum

## Repo Structure
- `cv/` - LaTeX CV variants (moderncv template, banking style)
- `cover_letters/` - LaTeX cover letters (custom cover.cls template)
- `.claude/skills/` - AI skill definitions for the application workflow
- `.agents/skills/` - Job search CLI tools

## Workflow for New Job Applications
1. User provides a job posting (URL or text)
2. **Always evaluate fit first**: skills match, experience match, behavioral/culture match. Present this assessment to the user before proceeding.
3. If good fit: create targeted CV (`cv/main_<company>.tex`) and cover letter (`cover_letters/cover_<company>_<role>.tex`)
4. **Verify both documents** (see Verification Checklist below)
5. Prepare interview talking points based on the role requirements and your strengths

**Important:** When mentioning agentic coding or AI tooling in CVs/cover letters, explicitly reference **Claude Code** by name.

## Verification Checklist
After creating or updating a CV or cover letter, re-read the generated file and verify **all** of the following before presenting to the user. Report the results as a pass/fail checklist.

### Factual accuracy
- [ ] All claims match actual profile (CLAUDE.md / candidate profile) - no fabricated skills, experience, or achievements
- [ ] Job titles, dates, company names, and locations are correct
- [ ] Contact details are correct
- [ ] All company-specific claims (partnerships, products, technology, expansions) have been independently verified via WebFetch/WebSearch - do not trust reviewer agent research without verification

### Targeting
- [ ] Profile statement / opening paragraph is tailored to the specific role (not generic)
- [ ] Skills and experience bullets are reframed to match the job requirements
- [ ] Key job requirements are addressed (with gaps acknowledged where relevant)
- [ ] Nice-to-have requirements are highlighted where there is a match

### Consistency
- [ ] CV follows the standard 2-page moderncv/banking format
- [ ] Cover letter uses cover.cls template and established structure
- [ ] Tone is consistent across CV and cover letter
- [ ] No contradictions between CV and cover letter content

### Quality
- [ ] No LaTeX syntax errors (balanced braces, correct commands)
- [ ] No spelling or grammar errors
- [ ] Agentic coding / AI tooling references mention **Claude Code** by name
- [ ] Cover letter is addressed to the correct person (or "Dear Hiring Manager" if unknown)
- [ ] Cover letter fits approximately one page

### Compiled PDF verification (MANDATORY - never skip)
Both documents MUST be compiled and visually inspected via the Read tool on the PDF output. "Looks fine in the .tex" is not acceptable - LaTeX page-break decisions are unpredictable. Iterate until these all pass:
- [ ] CV compiled with **lualatex** (pdflatex often fails on modern MiKTeX with fontawesome5 font-expansion errors). Cover letter compiled with **xelatex** (cover.cls requires fontspec).
- [ ] **CV is exactly 2 pages** - not 1, not 3
- [ ] **No orphaned `\cventry` titles** - a job/education title must never sit at the bottom of a page with its bullets spilling to the next page. Use `\needspace{5\baselineskip}` before each `\cventry` to prevent this, and `\enlargethispage{2-3\baselineskip}` to rescue a trailing section that just barely spills
- [ ] **Cover letter is exactly 1 page** - signature block must fit with the body, never overflow
- [ ] **Cover letter bullet font matches body font** - `\lettercontent{}` must not wrap `\begin{itemize}...\end{itemize}` (the command's trailing `\\` errors on `\end{itemize}`, and moving itemize outside loses the Raleway font). Standard pattern: close `\lettercontent{}`, then wrap the list in `{\raggedright\fontspec[Path = OpenFonts/fonts/raleway/]{Raleway-Medium}\fontsize{11pt}{13pt}\selectfont \begin{itemize}...\end{itemize}\par}`

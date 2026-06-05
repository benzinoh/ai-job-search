# Search Queries for Job Scraper

## Search Sites

Primary (US & Canada job market):
- **linkedin.com/jobs** — primary source for senior TPM/AI roles; filter by location and date posted
- **lever.co / greenhouse.io** — most AI/cloud companies (Anthropic, Databricks, Palantir, etc.) post here
- **jobs.ashbyhq.com** — common at Series B+ AI startups
- **careers pages** — Anthropic, AWS, Microsoft, Google, Palantir, Databricks, Snowflake, Datadog direct career pages

Secondary:
- **builtin.com** — strong for US tech roles, especially NYC/NJ area
- **wellfound.com** — AI startup roles
- **welcometothejungle.com** — strong for tech/AI companies with good editorial profiles; US + global remote roles; public search at `/en/jobs?query=...` (note: `/en/jobs-matches` requires login — use public search URL instead)

## Query Categories

Queries are organized by priority. All queries assume US and Canada geography unless otherwise specified. For LinkedIn, always apply: Date Posted = Past Week, Experience Level = Senior/Director.

### Priority 1: Senior Technical Program Manager (Core Target)

These match the strongest and most desired career direction.

```
site:linkedin.com/jobs "Senior Technical Program Manager" "AI" remote
site:linkedin.com/jobs "Senior Technical Program Manager" "AWS" "New Jersey" OR "New York" OR "remote"
site:linkedin.com/jobs "Staff Technical Program Manager" "cloud" remote
site:linkedin.com/jobs "Principal Technical Program Manager" "AI" OR "LLM" remote
site:linkedin.com/jobs "Engineering Program Manager" "AI platform" remote
"Senior Technical Program Manager" "GenAI" OR "Bedrock" OR "LLM" site:greenhouse.io OR site:lever.co
```

### Priority 2: AI Platform / LLM Program Management

These match the AI engineering depth shown in the profile.

```
site:linkedin.com/jobs "AI Program Manager" "AWS" OR "Bedrock" OR "LLM" remote
site:linkedin.com/jobs "GenAI Program Manager" remote
site:linkedin.com/jobs "LLM" "Technical Program Manager" remote
site:linkedin.com/jobs "AI Platform" "TPM" OR "Program Manager" remote
"AI infrastructure" "program manager" site:greenhouse.io OR site:lever.co
site:anthropic.com/careers "program manager"
site:jobs.ashbyhq.com "technical program manager"
```

### Priority 3: Cloud Platform / Infrastructure Program Management

Adjacent roles matching cloud and SRE program background.

```
site:linkedin.com/jobs "Cloud Platform Program Manager" "AWS" remote
site:linkedin.com/jobs "Infrastructure Program Manager" "SRE" OR "reliability" remote
site:linkedin.com/jobs "DevOps Program Manager" "AWS" OR "Azure" remote
site:linkedin.com/jobs "Platform Engineering" "TPM" "AWS" remote
"cloud transformation" "Technical Program Manager" site:lever.co OR site:greenhouse.io
```

### Priority 3b: Welcome to the Jungle

WTTJ has strong coverage of AI/tech companies that invest in employer branding — often surfaces roles not visible on LinkedIn yet.

```
site:welcometothejungle.com "Technical Program Manager" "AI" OR "cloud" remote
site:welcometothejungle.com "Senior TPM" OR "Staff TPM" remote
site:welcometothejungle.com "Technical Program Manager" "AWS" OR "platform"
```

Alternatively, search directly on WTTJ:
- https://www.welcometothejungle.com/en/jobs?query=technical+program+manager&refinementList%5Boffice_country_codes%5D%5B%5D=US
- https://www.welcometothejungle.com/en/jobs?query=senior+tpm+AI+remote

### Priority 4: Broader / Adjacent Roles (Wider Net)

Roles that leverage the technical depth but have different titles.

```
site:linkedin.com/jobs "Engineering Manager" "AI platform" "AWS" remote
site:linkedin.com/jobs "Director of Engineering Programs" "cloud" OR "AI" remote
site:linkedin.com/jobs "Solutions Architect" "AI" "AWS" "New Jersey" OR remote
site:linkedin.com/jobs "AI Solutions Engineer" "enterprise" remote
site:linkedin.com/jobs "Technical Lead" "GenAI" "program" remote
```

## Target Company Direct Career Pages

Check these weekly — they rarely appear in aggregators first:

| Company | Why | Career URL |
|---------|-----|------------|
| Anthropic | AI safety + Claude/Bedrock alignment | anthropic.com/careers |
| AWS | Deep AWS expertise; Bedrock org specifically | amazon.jobs |
| Datadog | Observability; knows the stack cold | careers.datadoghq.com |
| Palantir | TPM culture, enterprise AI, built a targeting CV for them | palantir.com/careers |
| Databricks | AI/data platform, MLOps angle | databricks.com/company/careers |
| Snowflake | Cloud data + AI programs | snowflake.com/company/careers |
| Microsoft (Azure/AI) | Azure Cloud + Copilot AI program angle | careers.microsoft.com |
| Google (Cloud/DeepMind) | GCP + AI programs | careers.google.com |
| Caylent | Former employer; strong rehire signal | caylent.com/careers |

## Location Filter

When evaluating results, verify the job location is within scope:

- **Ideal:** Remote-first, or New Jersey / New York City metro
- **Acceptable:** Any US location (willing to travel or relocate for strong fit), Canada (Toronto, Vancouver)
- **Borderline:** Requires 5+ days/week in-office outside NY metro (flag, discuss)
- **Exclude:** Outside US/Canada unless fully remote with US-timezone primary hours

## Salary Filter

Only pursue roles where the compensation range (if listed) reaches or exceeds $180K USD base. If not listed, proceed — many strong roles don't post ranges. Flag during evaluation if Glassdoor/levels.fyi suggests below-range.

## Date Filter

Only include jobs posted within the last 14 days, or with an application deadline that has not yet passed. If a posting date cannot be determined, include it but flag as "date unknown."

## Adapting Queries

If the user specifies a focus area, select queries from the matching category and generate 2–3 custom queries for that focus. Examples:
- `/scrape anthropic` → Priority 2 AI queries + direct site:anthropic.com/careers search
- `/scrape palantir` → Priority 1 TPM queries targeting Palantir specifically + site:palantir.com/careers
- `/scrape remote only` → All priority queries with `remote` filter only, excluding location-specific terms

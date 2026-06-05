<p align="center">
  <img src="claude_animation.gif" alt="Claude Job Search Assistant" width="200">
</p>

# AI Job Search — Benjamin Nwokoye

An AI-powered job application framework built on [Claude Code](https://claude.com/claude-code), customized for a senior AI/cloud engineering and TPM job search targeting US and Canadian markets.

This is a fork of [MadsLorentzen/ai-job-search](https://github.com/MadsLorentzen/ai-job-search) with the full workflow adapted for a US/Canada job market, a production LaTeX CV pipeline with verified PDF output, Playwright-based Greenhouse application automation, and a candidate profile configured for Benjamin Nwokoye.

## What this is

A structured workflow that turns Claude Code into a full-stack job application assistant. The core flow covers profile setup, job scraping, fit evaluation, LaTeX CV and cover letter generation with verified PDF compilation, and optional Playwright-driven Greenhouse form submission.

```
/setup          /scrape              /apply <url>
  |                |                     |
  v                v                     v
Fill in        Search US/CA          Evaluate fit
your profile   job portals           Score & recommend
  |                |                     |
  v                v                     v
Profile        Present matches      Draft CV + Cover Letter
files ready    with fit ratings     (LaTeX, tailored)
                   |                     |
                   v                     v
               Pick a match         Reviewer agent critiques
               -> /apply            -> Revise -> Final output
                                         |
                                         v
                                    Compile & inspect PDFs
                                    (lualatex CV / xelatex CL)
                                         |
                                         v
                                    Playwright auto-fill
                                    scripts/submit_<company>.py
```

## Prerequisites

- [Claude Code](https://claude.com/claude-code) (CLI)
- Python 3.10+ with `playwright` (`pip install playwright && playwright install chromium`)
- LaTeX distribution with `lualatex` and `xelatex`: [TeX Live](https://tug.org/texlive/) or [MiKTeX](https://miktex.org/)
  - CV compiles with `lualatex` (pdflatex fails on modern MiKTeX with `fontawesome5` font-expansion errors)
  - Cover letter compiles with `xelatex` (`cover.cls` requires `fontspec`)

## Quick start

### 1. Fork and clone

```bash
gh repo fork MadsLorentzen/ai-job-search --clone
cd ai-job-search
```

### 2. Set up your profile

```bash
claude
# Then inside Claude Code:
/setup
```

`/setup` offers three paths: read your `documents/` folder (CV PDF, LinkedIn export, diplomas, references, past applications), import a single CV pasted in chat, or walk through an interview. It auto-detects what you have and asks. Documents-folder mode is idempotent and safe to re-run as you add more material; see `documents/README.md` for the layout.

### 3. Search for jobs

```bash
/scrape
```

Searches multiple portals for positions matching your profile, deduplicates results, and presents them sorted by fit. US and Canada job market is the default. Pick a match to run `/apply` on it directly.

### 4. Apply to a job

```bash
/apply https://boards.greenhouse.io/company/jobs/12345
```

If the URL can't be fetched, paste the job description directly:

```bash
/apply <paste the full job description here>
```

### 5. (Optional) Auto-submit via Playwright

Once a CV has been compiled and reviewed, run the matching submission script to auto-fill the Greenhouse form. The browser stays open for you to review and submit:

```bash
python3 scripts/submit_<company>.py
```

## Commands

| Command | Purpose |
|---------|---------|
| `/setup` | Profile onboarding (documents folder, CV import, or interview) |
| `/scrape` | Search job portals and present ranked matches |
| `/apply <url\|text>` | Full drafter-reviewer workflow: evaluate → draft CV + CL → review → compile |
| `/expand` | Enrich profile from GitHub, portfolio site, and named certifications |
| `/upskill [url]` | Skill-gap heatmap and learning plan vs tracked postings or a single role |
| `/reset [profile\|documents\|all]` | Wipe profile data or documents folder (confirms before deleting) |

## File structure

```
ai-job-search/
├── CLAUDE.md                          # Candidate profile + workflow rules (Benjamin Nwokoye)
├── .claude/
│   ├── commands/
│   │   ├── apply.md                   # /apply drafter-reviewer workflow
│   │   ├── setup.md                   # /setup onboarding
│   │   ├── expand.md                  # /expand competency enrichment
│   │   └── reset.md                   # /reset wipe profile data
│   ├── skills/
│   │   ├── job-application-assistant/ # Core application skill
│   │   │   ├── 01-candidate-profile.md # Education, experience, skills
│   │   │   ├── 02-behavioral-profile.md# Behavioral / PI-style self-assessment
│   │   │   ├── 03-writing-style.md    # Tone, structure, do's and don'ts
│   │   │   ├── 04-job-evaluation.md   # Fit-scoring framework + target roles
│   │   │   ├── 05-cv-templates.md     # LaTeX CV structure + tailoring rules
│   │   │   ├── 06-cover-letter-templates.md # LaTeX cover letter templates
│   │   │   └── 07-interview-prep.md   # STAR examples + interview framework
│   │   ├── job-scraper/
│   │   │   └── search-queries.md      # US/Canada job portal queries (LinkedIn, Greenhouse, Lever, Ashby, WTTJ)
│   │   └── upskill/                   # /upskill skill gap analysis
│   └── settings.local.json            # Claude Code permissions
├── cv/
│   ├── main_example.tex               # moderncv banking template
│   ├── main_anthropic.tex/.pdf        # Anthropic TPM Launches
│   ├── main_anthropic_alignment.tex/.pdf # Anthropic TPM Alignment
│   ├── main_anthropic_research.tex/.pdf  # Anthropic TPM Research
│   ├── main_cerebras.tex/.pdf         # Cerebras Senior TPM
│   └── main_reddit.tex/.pdf           # Reddit Principal TPM
├── cover_letters/
│   ├── cover.cls                      # Custom LaTeX cover letter class (Lato/Raleway)
│   ├── OpenFonts/                     # Lato + Raleway font files
│   ├── cover_anthropic_tpm_launches.tex/.pdf
│   ├── cover_anthropic_tpm_alignment.tex/.pdf
│   ├── cover_anthropic_tpm_research.tex/.pdf
│   ├── cover_cerebras_tpm_infra.tex/.pdf
│   └── cover_reddit_tpm_devprod.tex/.pdf
├── scripts/                           # Playwright Greenhouse auto-fill scripts
│   ├── submit_anthropic.py            # Anthropic TPM Launches
│   ├── submit_anthropic_alignment.py  # Anthropic TPM Alignment
│   ├── submit_anthropic_research.py   # Anthropic TPM Research
│   ├── submit_cerebras.py             # Cerebras Senior TPM
│   └── submit_reddit.py               # Reddit Principal TPM
├── documents/                         # Career source materials for /setup and /expand
│   ├── cv/                            # Master CVs (PDF variants)
│   ├── linkedin/                      # LinkedIn profile export
│   ├── diplomas/                      # Degree certificates
│   ├── references/                    # Reference letters
│   └── applications/                  # Past application records
├── job_search_tracker.csv             # Application tracking (company, role, status, fit score, files)
├── salary_lookup.py                   # Salary benchmarking tool (BYO data)
├── tools/
│   ├── convert_salary_excel.py        # Convert salary Excel to JSON
│   └── README_SALARY_TOOL.md          # Salary tool setup instructions
├── job_scraper/                       # Scraper state (seen_jobs.json)
├── upskill/                           # /upskill report output
└── SETUP.md                           # Detailed setup guide
```

## How `/apply` works

The `/apply` command runs a **drafter-reviewer workflow** with mandatory PDF compilation and visual inspection:

1. **Parse** the job posting (URL or text)
2. **Evaluate fit** against the candidate profile (skills, experience, culture, location, career alignment)
3. **Draft** a tailored CV and cover letter in LaTeX
4. **Spawn a reviewer agent** that researches the company and critiques the drafts
5. **Revise** based on the reviewer's feedback
6. **Compile and inspect** both PDFs:
   - CV: `lualatex` → must be exactly 2 pages, no orphaned `\cventry` titles
   - Cover letter: `xelatex` → must be exactly 1 page, bullet font must match body font
   - Iterate with `\needspace`, `\enlargethispage`, and font-matching wrappers until layout is clean
7. **Present** the final output with a pass/fail verification checklist

All claims are verified against the actual profile. No fabricated skills or experience.

### What makes this workflow different

- **PDF verification loop.** Compiles and visually inspects every PDF — catches orphaned job titles, cover letters spilling to page 2, and bullet font fallbacks that look fine in `.tex` but break in the rendered output.
- **Relevance-weighted CV cutting.** When a CV overflows 2 pages, cuts by lowest score of (relevance to posting × uniqueness in document × cover letter dependency), not by recency.
- **Drafter-reviewer separation.** A second Claude agent with fresh context researches the company and critiques the drafts. Catches missed keywords, weak framing, and generic language a single pass leaves in.
- **Playwright submission scripts.** Per-company scripts in `scripts/` auto-fill Greenhouse forms — standard fields, file uploads, custom questions, dropdowns — and pause for human CAPTCHA + Submit. Application data is inlined in each script so all fields are pre-verified.

## Customization

### Which files to edit

| File | What to change |
|------|---------------|
| `CLAUDE.md` | Full candidate profile (name, experience, skills, goals, deal-breakers, salary floor) |
| `01-candidate-profile.md` | Structured CV data for skill loading |
| `02-behavioral-profile.md` | Behavioral self-assessment (PI/DISC style) |
| `04-job-evaluation.md` | Skill match areas, career goals, fit scoring weights |
| `05-cv-templates.md` | Profile statement templates per role type |
| `07-interview-prep.md` | STAR examples from actual experience |
| `search-queries.md` | Job portal queries, target companies, location/salary filters |

### Updating search queries without full re-setup

```
/setup --section search
```

Re-runs just the search configuration: which roles, skills, locations, and portals. Also suggests role types based on your profile.

### LaTeX templates

The CV uses [moderncv](https://ctan.org/pkg/moderncv) (banking style) compiled with `lualatex`. The cover letter uses a custom `cover.cls` with Lato/Raleway fonts compiled with `xelatex`. Swap in your own templates by updating `05-cv-templates.md` and `06-cover-letter-templates.md`.

### Job search portals

`search-queries.md` is configured for the US and Canadian job market: LinkedIn, Greenhouse, Lever, Ashby HQ, Builtin, Wellfound, and Welcome to the Jungle. The original Danish portals (Jobbank, Jobdanmark, Jobindex, Jobnet) in `.agents/skills/` are retained for reference but not used in the active search queries.

### Greenhouse auto-submission scripts

Each `scripts/submit_<company>.py` is self-contained. To add a new company:

1. Copy an existing script
2. Update the constants block (name, email, phone, CV path, custom question IDs, answer text)
3. Map Greenhouse field IDs using browser DevTools (inspect `#field_id` on the form)
4. Run: `python3 scripts/submit_<company>.py`

The script opens a headed browser, fills all fields, and pauses for human review + CAPTCHA + Submit.

### Starting over

```
/reset profile    # clears skill files, preserves framework rules
/reset documents  # deletes files from documents/ folder
/reset all        # both
```

`/reset` shows exactly what will be deleted and requires typing `RESET` to confirm.

## Applications tracked

| Date | Company | Role | Fit | Status |
|------|---------|------|-----|--------|
| 2026-06-04 | Anthropic | TPM Launches | 85% | Applied |
| 2026-06-04 | Reddit | Principal TPM Developer Productivity | 82% | Applied |
| 2026-06-04 | Anthropic | TPM Research | 78% | Applied |
| 2026-06-04 | Anthropic | TPM Alignment | 72% | Applied |
| 2026-06-04 | Cerebras Systems | Senior TPM AI Infrastructure | 69% | Applied |

Full tracking in `job_search_tracker.csv`.

## Tips for better results

**Profile depth matters.** The single biggest factor in output quality. Don't just list job titles — describe specific projects, tools, responsibilities, and measurable achievements. Skills in context ("built ML pipelines for customer churn prediction in Python") give the system far more to work with than bare keywords.

**Use the documents folder.** Dropping multiple CV variants into `documents/cv/` gives `/setup` raw material to synthesize a richer profile than a single master document. Reference letters, diplomas, and past applications add signal the system can draw on during fit evaluation and cover letter writing.

**CAPTCHA is intentional.** The Playwright scripts pause for human review and CAPTCHA — they do not attempt to bypass it. Review every filled field before submitting.

## Acknowledgements

- [MadsLorentzen](https://github.com/MadsLorentzen/ai-job-search) — original framework
- [Mikkel Krogholm](https://github.com/mikkelkrogsholm) ([skills repo](https://github.com/mikkelkrogsholm/skills)) — job search CLI skills
- Built with [Claude Code](https://claude.com/claude-code) by [Anthropic](https://anthropic.com)

## License

MIT

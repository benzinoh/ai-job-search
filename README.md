<p align="center">
  <img src="claude_animation.gif" alt="AI Job Search Assistant" width="200">
</p>

# AI Job Search

An AI-powered job application framework built on [Claude Code](https://claude.com/claude-code) that automates the end-to-end job search pipeline: scraping and evaluating job postings, generating tailored CVs and cover letters, and auto-submitting applications — all driven by a match score against your candidate profile.

## What this does

Turns Claude Code into a full-stack job application assistant. Give it your profile once; it handles the rest.

```
/setup          /scrape              /apply <url>
  |                |                     |
  v                v                     v
Fill in        Search job           Evaluate fit
your profile   portals              Score & recommend
  |                |                     |
  v                v                     v
Profile        Present matches      Draft CV + Cover Letter
files ready    with fit scores      (LaTeX, tailored)
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
gh repo fork benzinoh/ai-job-search --clone
cd ai-job-search
```

### 2. Set up your profile

```bash
claude
# Then inside Claude Code:
/setup
```

`/setup` offers three paths: read your `documents/` folder (CV PDF, LinkedIn export, diplomas, references, past applications), import a single CV pasted in chat, or walk through a guided interview. Documents-folder mode is idempotent and safe to re-run as you add material.

### 3. Search for jobs

```bash
/scrape
```

Searches multiple portals for positions matching your profile, deduplicates results, and presents them sorted by fit score. Pick a match to run `/apply` directly.

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
├── CLAUDE.md                          # Candidate profile + workflow rules
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
│   │   │   └── search-queries.md      # Job portal queries, target roles, filters
│   │   └── upskill/                   # /upskill skill gap analysis
│   └── settings.local.json            # Claude Code permissions
├── cv/
│   ├── main_example.tex               # moderncv banking template (base)
│   └── main_<company>.tex             # Per-application tailored CV (generated)
├── cover_letters/
│   ├── cover.cls                      # Custom LaTeX cover letter class (Lato/Raleway)
│   ├── OpenFonts/                     # Lato + Raleway font files
│   └── cover_<company>_<role>.tex     # Per-application cover letter (generated)
├── scripts/
│   └── submit_<company>.py            # Playwright Greenhouse auto-fill (generated)
├── documents/                         # Career source materials for /setup and /expand
│   ├── cv/                            # Master CVs (PDF variants)
│   ├── linkedin/                      # LinkedIn profile export
│   ├── diplomas/                      # Degree certificates
│   ├── references/                    # Reference letters
│   └── applications/                  # Past application records
├── job_search_tracker.csv             # Application tracking (company, role, status, fit score)
├── salary_lookup.py                   # Salary benchmarking tool (BYO data)
└── tools/
    ├── convert_salary_excel.py        # Convert salary Excel to JSON
    └── README_SALARY_TOOL.md          # Salary tool setup instructions
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
- **Playwright submission scripts.** Per-company scripts in `scripts/` auto-fill Greenhouse forms — standard fields, file uploads, custom questions, dropdowns — and pause for human CAPTCHA + Submit.

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

Re-runs just the search configuration: which roles, skills, locations, and portals.

### LaTeX templates

The CV uses [moderncv](https://ctan.org/pkg/moderncv) (banking style) compiled with `lualatex`. The cover letter uses a custom `cover.cls` with Lato/Raleway fonts compiled with `xelatex`. Swap in your own templates by updating `05-cv-templates.md` and `06-cover-letter-templates.md`.

### Job search portals

`search-queries.md` controls which portals are searched: LinkedIn, Greenhouse, Lever, Ashby HQ, Builtin, Wellfound, and Welcome to the Jungle. Update the query categories and location/salary filters to match your market.

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

## Tips for better results

**Profile depth matters.** The single biggest factor in output quality. Don't just list job titles — describe specific projects, tools, responsibilities, and measurable achievements. Skills in context give the system far more to work with than bare keywords.

**Use the documents folder.** Dropping multiple CV variants into `documents/cv/` gives `/setup` raw material to synthesize a richer profile than a single master document. Reference letters, diplomas, and past applications add signal the system can draw on during fit evaluation and cover letter writing.

**CAPTCHA is intentional.** The Playwright scripts pause for human review and CAPTCHA — they do not attempt to bypass it. Review every filled field before submitting.

## Acknowledgements

- [MadsLorentzen](https://github.com/MadsLorentzen/ai-job-search) — original framework
- [Mikkel Krogholm](https://github.com/mikkelkrogsholm) ([skills repo](https://github.com/mikkelkrogsholm/skills)) — job search CLI skills
- Built with [Claude Code](https://claude.com/claude-code) by [Anthropic](https://anthropic.com)

## License

MIT

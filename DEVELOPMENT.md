# DEVELOPMENT.md

Guidance for working on the AI Job Search **framework itself** (the template,
its skills, commands, and tooling) — not for running the job-search workflow
as an end user. For that, see `README.md` and `AGENTS.md`.

## What this repo is

A Claude Code-native job application framework, distributed as a **fork-and-adapt
template**. Two audiences share this codebase and must not be conflated:

- **Framework maintainers/contributors** (this file's audience): edit `.claude/`
  skills/commands, `.agents/` portal CLIs, and `tools/` Python scripts.
- **Fork users**: run `/setup`, `/scrape`, `/apply`, etc. against their own
  personal data, which is git-ignored and never committed upstream.

Read `CONTRIBUTING.md` before proposing changes — it explains what upstream
accepts (universal, market-agnostic features; robustness fixes with a
reproducible failing case) and what it declines (market-specific portal
skills, personal profile data, alternate-runtime ports). `CLAUDE.md` at the
repo root is **not** a Claude Code config file in the usual sense — it is the
candidate-profile entry point that `/setup` populates for a fork user, kept as
a placeholder template (`[YOUR_NAME]`, etc.) upstream. Do not repurpose or
overwrite it.

## Commands

```bash
# Python: skill/command/settings lint (frontmatter, allowed-tools targets, settings.json shape)
python3 tools/lint_skills.py

# Python: full unit test suite
python3 -m unittest discover -s tests -t . -v

# Python: single test file / single test
python3 -m unittest tests.test_salary_lookup -v
python3 -m unittest tests.test_salary_lookup.SalaryLookupTest.test_exact_match -v

# Security guards: permissions allowlist, .gitignore personal-data rules, package.json lifecycle scripts
python3 tools/security_guards.py

# Framework version guard (upstream-only in CI; requires PyYAML)
python3 tools/check_framework_version.py

# Per portal-CLI (run from .agents/skills/<portal>/cli/)
bun install
bun run typecheck
bun test          # only where tests/*.test.ts exists (network-free fixtures/mocks)
```

`tools/lint_skills.py` requires `pyyaml` (`pip install pyyaml`). Everything
else in `tools/` and `tests/` is stdlib-only.

CI (`.github/workflows/ci.yml`) runs, in order: skill/command lint →
security guards → Python unit tests → dependency review (upstream PRs only) →
per-portal Bun typecheck/tests → placeholder-integrity (upstream repo only) →
LaTeX smoke compiles. Reproduce a failure locally with the commands above
before pushing.

## Architecture

### Thin-pointer, single-source-of-truth design

The instructive rule, from `AGENTS.md`: every agent runtime (Claude Code,
Codex, Antigravity, Gemini CLI, ...) should read the *same* files rather than
each getting its own copy:

- Candidate profile & workflow methodology → `CLAUDE.md` +
  `.claude/skills/job-application-assistant/*.md`
- Workflow specs (setup/scrape/rank/apply/interview/upskill) →
  `.claude/skills/` and `.claude/commands/`
- Portal search tools → `.agents/skills/*/SKILL.md` (portable Agent Skills
  format, auto-discovered by non-Claude runtimes too)

Never duplicate a spec across two files "for convenience" — one copy drifts
from the other the moment either changes. This is also why alternate-runtime
ports (a second implementation of the workflow for another agent CLI) are
declined upstream (see `CONTRIBUTING.md`): the markdown specs under
`.claude/` **are** the implementation.

### The command lifecycle and how commands connect

```
/setup → /scrape → /rank → /apply <url> → /interview → /outcome → (calibrates /setup)
                                              ^                         |
                              /expand, /upskill, /add-template,        |
                              /add-portal, /reset ------------------- (support cast)
```

Each command is a `.claude/commands/<name>.md` file: a numbered-steps prompt
script executed directly by Claude Code, not by any program. State flows
between commands through plain files, not APIs:

- `job_scraper/seen_jobs.json` — dedup state written by `/scrape`, read by
  `/scrape` and `/rank`
- `job_search_tracker.csv` — application status, written by `/outcome` (and
  optionally `/scrape`), read by `/scrape`/`/rank` for exclusion and by
  `/html-report`
- `documents/applications/<company>_<role>/` — per-application archive
  (posting, submitted CV/cover-letter, `outcome.md`) written by `/outcome`,
  mined by `/setup` to calibrate `04-job-evaluation.md` and surface new STAR
  examples
- `.claude/skills/job-application-assistant/0*-*.md` — the candidate profile
  and methodology files, written by `/setup`/`/expand`, read by every other
  command

When adding a new command, the bar (per `CONTRIBUTING.md`) is that it must
operationalize something that already exists but nothing currently
reads/writes — not just be "useful."

### `/apply`'s drafter-reviewer pipeline

`/apply` (`.claude/commands/apply.md`) is the most involved command: it
fetches/parses the posting, evaluates fit against
`04-job-evaluation.md`/`01-candidate-profile.md`, drafts CV + cover letter
LaTeX from `05-cv-templates.md`/`06-cover-letter-templates.md`, dispatches a
reviewer agent to critique the draft, revises, and only then runs the
Verification Checklist in `CLAUDE.md` (LaTeX compiles, exact page counts, ATS
text-layer extraction via `pdftotext -layout`). The checklist step is
mandatory and not skippable by design — `.tex` source "looking fine" does not
predict page-break behavior in the compiled PDF.

### Portal search skills (`.agents/skills/`)

Each portal (`jobindex-search`, `jobnet-search`, `jobbank-search`,
`jobdanmark-search`, `freehire-search`, `linkedin-search`) is a self-contained
TypeScript CLI under `<portal>/cli/`, built with Bun, exposed to agents via
its own `SKILL.md`. The portal-skill contract (see `/add-portal` and
`linkedin-search` as the reference implementation):

- `search` and `detail` subcommands
- `--format json|table|plain`
- Errors as stderr JSON with exit code 1
- Backoff on HTTP 429/5xx
- Zero runtime dependencies by default (no `package.json` lifecycle scripts —
  `tools/security_guards.py` enforces this in CI)
- An `enabled` frontmatter flag in `SKILL.md` (default true) lets a fork keep
  a portal installed but have `/scrape` skip it

`/scrape` (`.claude/skills/job-scraper/SKILL.md`) discovers portals by
globbing `.agents/skills/*/SKILL.md` at runtime — adding a new portal via
`/add-portal` requires no changes to `/scrape` itself.

Only `linkedin-search` and the country-agnostic pattern are meant to live
upstream; the four Danish portals are the maintainer's own demonstration
instance. New country/market-specific portal skills are out of scope for
upstream PRs (fork them instead) — see `CONTRIBUTING.md`.

### LaTeX templates

Stock templates: `cv/main_example.tex` (moderncv, compiled with **lualatex**
— pdflatex breaks on `fontawesome5` under modern MiKTeX) and
`cover_letters/cover_example.tex` (custom `cover.cls`, compiled with
**xelatex** because it needs `fontspec` for bundled Lato/Raleway fonts under
`cover_letters/OpenFonts/fonts/`). `/apply` generates
`cv/main_<company>.tex` and `cover_letters/cover_<company>_<role>.tex` from
these as starting points; both are git-ignored (personal output).

Custom templates registered via `/add-template` live under `templates/{cv,cover_letters}/<name>/`
as profile-agnostic `[PLACEHOLDER]`-token skeletons plus a `TEMPLATE.md`
manifest (engine, fonts, page limit, pitfalls). Activating one writes a
managed block into `05-cv-templates.md` or `06-cover-letter-templates.md`,
which is what `/apply` actually reads — `templates/` itself is never read
directly by `/apply`.

Any LaTeX change must keep both stock templates compiling with their
documented engines and holding their exact page counts (CI smoke-checks
this); `tools/verify_pdf.py` is the same check `/apply` runs on generated
output.

### Security posture (why `tools/security_guards.py` exists)

This is a template that ships pre-approved Claude Code permissions
(`.claude/settings.json`) and CLI code every fork user executes verbatim.
`tools/security_guards.py` fails CI if a PR:

- adds a `.claude/settings.json` permission not already in
  `ALLOWED_PERMISSIONS` (widening auto-approved commands affects every fork)
- removes a personal-data rule from `.gitignore`'s `REQUIRED_IGNORE_RULES`
- adds npm/bun lifecycle scripts (`preinstall`/`install`/`postinstall`/
  `prepare`/`prepack`) or `trustedDependencies` to any `.agents/**/package.json`

If a change intentionally needs one of these, update the corresponding
allowlist in `tools/security_guards.py` in the same diff — the guard is
designed to make the change loud and reviewable, not to block it outright.

### Framework version markers

Files under `.claude/skills/job-application-assistant/*.md` and `AGENTS.md`
carry a `framework_version` frontmatter key. `tools/check_framework_version.py`
(upstream CI only) fails if one of these files has a non-trivial diff without
a corresponding version bump — this is what lets a runtime fork or an older
personalized fork detect that upstream's methodology changed and merge
deliberately rather than silently drifting.

## Personal data boundary

Nothing under `documents/*/` (except `README.md`/`.gitkeep`), no
`cv/main_*.tex` / `cover_letters/cover_*.tex` (except the `_example` stock
files), no `job_search_tracker.csv`, `salary_data.json`, or
`job_scraper/seen_jobs.json` is meant to be committed — `.gitignore` enforces
this, and `tools/security_guards.py` checks the ignore rules stay intact.
Upstream's own `CLAUDE.md` and `cv/main_example.tex` /
`cover_letters/cover_example.tex` are required to keep their `[YOUR_NAME]` /
`[YOUR NAME]` placeholder tokens — `placeholder-integrity` in CI (upstream
repo only) fails otherwise.

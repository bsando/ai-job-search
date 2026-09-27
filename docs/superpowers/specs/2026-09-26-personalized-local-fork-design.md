# Personalized local fork — design

**Date:** 2026-09-26
**Status:** approved in chat, pending spec review

## Goal

Turn `~/Documents/projects/ai-job-search` (a fork of
`MadsLorentzen/ai-job-search`) into Brian's private, local-only job
search workspace: personalized profile, US job sources, and a curated
company watchlist — while still able to pull upstream updates.

## Non-goals

- Replacing or migrating career-ops. It stays as-is; this fork only
  reads from it once to seed the profile.
- Salary benchmarking. The bundled `salary_lookup.py` is Danish-data
  specific; compensation checks rely on posted ranges + web search.
- Any automated submission. Drafts only, same as career-ops.

## 1. Local-only repo

- Remove the `origin` remote (`bsando/ai-job-search`, a public fork).
  Keep `upstream`. Nothing in this repo is ever pushed.
- All personalization lives on the local `brian` branch. `master`
  tracks `upstream/master` untouched; updates come in via
  `git fetch upstream && git merge upstream/master` on `brian`.
- Master was fast-forwarded to upstream (`120f476`) on 2026-09-26
  before branching (it was 227 commits behind).
- Deleting the public GitHub fork is left to Brian — it contains only
  the unmodified template, so it leaks nothing.
- Personal data is committed locally in tracked files (the upstream
  design). The folder is iCloud-synced; that's accepted.

## 2. Profile

Populate `CLAUDE.md` and
`.claude/skills/job-application-assistant/01`–`07` by following
`/setup`'s CV-import path, with these sources:

| Target | Source |
|---|---|
| Identity, location, work auth, comp floor | `career-ops/config/profile.yml` |
| Education, experience, skills | `career-ops/cv.md` |
| Target roles + fit framework (`04`) | `profile.yml` `target_roles` / archetypes |
| Narrative, proof points, STAR seeds (`07`) | `profile.yml` `narrative` |
| Workflow house rules | `career-ops/modes/_custom.md` (triage-first, cost discipline, never auto-submit) |
| Behavioral profile (`02`), writing style (`03`) | No source — short interview with Brian, or derive style from past cover letters if supplied |

Key facts to carry: San Jose, CA; Bay Area hybrid/on-site or US
remote; US citizen, no sponsorship; floor $200K base, target
$200–250K; current employer Turo (excluded from search).

## 3. Job sources

| Portal | State |
|---|---|
| Danish portals (5) | Already `enabled: false` upstream — leave untouched |
| `linkedin-search` | Enabled; US / Bay Area / remote queries |
| `freehire-search` | Enabled; `--country US` + remote |
| `ats-search` (new) | Enabled; watchlist of ~50 companies |

`.claude/skills/job-scraper/search-queries.md` gets Brian's role
keywords (see §4 filters).

## 4. `ats-search` skill (new)

Location: `.agents/skills/ats-search/`, scaffolded to match
`linkedin-search` (Bun, zero runtime deps, `cli/src`, `cli/tests`,
`SKILL.md` with `enabled:` flag). `/add-portal`'s structure is the
reference.

**Watchlist:** `.agents/skills/ats-search/watchlist.yml`

```yaml
- company: Airbnb
  ats: greenhouse      # greenhouse | lever | ashby | none
  slug: airbnb
- company: Netflix
  ats: none            # not on a supported ATS → LinkedIn company search
```

**Sources** (public, unauthenticated JSON):

- Greenhouse: `https://boards-api.greenhouse.io/v1/boards/{slug}/jobs?content=true`
- Lever: `https://api.lever.co/v0/postings/{slug}?mode=json`
- Ashby: `https://api.ashbyhq.com/posting-api/job-board/{slug}?includeCompensation=true`

**Commands:**

- `search [--query <kw>] [--since <days>] [--format json|table]` —
  fetch all watchlist boards (bounded concurrency, per-request timeout,
  one failing board never fails the run), normalize, filter, print.
- `detail <url>` — full description for one posting.
- `check` — report each watchlist entry as reachable / slug_gone.

**Normalized output** matches the fields `job-scraper` already reads
from the other CLIs: title, company, location, date, url, plus
`salary` when the ATS exposes it (Ashby compensation, Greenhouse pay
ranges in content).

**Default filters** (overridable by `--query`):

- Title seniority: Director, Head of, VP, Sr/Senior Manager, Staff,
  Principal, Lead.
- Title function: analytics, data science, data, growth, marketing
  science, measurement, insights, decision science, applied AI, AI,
  machine learning.
- Location: Bay Area (San Francisco, San Jose, Palo Alto, Mountain
  View, Menlo Park, Oakland, Sunnyvale, Redwood City, South San
  Francisco), or remote with US eligibility. Non-US-only remote is
  excluded.

`ats: none` entries are not fetched; the skill lists them so
`job-scraper` runs a LinkedIn search with `company:<name>` for each.

## 5. Watchlist (initial, ~50)

**Consumer tech:** Airbnb, DoorDash, Instacart, Lyft, Pinterest,
Reddit, Discord, Duolingo, Robinhood, Chime, Coinbase, Figma, Notion,
Canva, Spotify, Strava, Peloton, Nextdoor, Thumbtack, Faire, Etsy,
Eventbrite, Grammarly, Calm, Roblox, Patreon, Hims & Hers, Match Group.

**AI:** Anthropic, OpenAI, Perplexity, Scale AI, Databricks, Glean,
Sierra, Harvey, Anysphere (Cursor), ElevenLabs, Runway, Character.AI,
Replit, Vercel, Hugging Face, Cohere, Writer, Hebbia, Abridge, Suno,
Lovable, Decagon.

**Excluded:** Turo.

Each company's ATS + slug is verified live during implementation;
unverifiable ones become `ats: none`.

## 6. Testing

- `bun test` in `ats-search/cli`: per-ATS parsing against recorded
  fixture JSON, title filter, location filter (incl. non-US remote
  rejection), one-board-fails-run-continues.
- Existing `pytest` suite stays green.
- Live smoke: `ats-search check`, then one real `/scrape` pass.

## Risks

- ATS slugs drift (career-ops saw ~5% `slug_gone`). `check` surfaces
  it; fix by editing `watchlist.yml`.
- Upstream merges could conflict with personalized files — expected
  and small, since upstream only changes the templates.

---
tags: [operations, technology, website, dashboard, eos]
---

# Nexus Leadership Dashboard

## Vision

A private, password-protected section of the [[nexus-website|Nexus website]] that gives leadership a real-time visual dashboard of the entire business. Loosely follows the [[EOS]] (Entrepreneurial Operating System) framework — Scorecard, Rocks, Issues, To-Dos, Accountability Chart — extended with domain-specific pages for every area of the business.

This is not where work gets done. Work happens in the [[Claude Code]] vault. This is the **presentation layer** — a beautiful, always-current visual representation of what's happening across Nexus, powered by data that Claude maintains automatically.

## Architecture

```
Claude Code vault (source of truth)
    ↓ generates
dashboard-data.json (this folder)
    ↓ git push to
GitHub repo (public raw URL)
    ↓ fetched by (TanStack Query, 5-min staleTime)
Nexus Website (Replit) — React/TS/Vite + shadcn/ui
    ↓ rendered as
8 private leadership pages (behind LeadershipGuard auth)
```

**GitHub Repo:** https://github.com/jbearup1981/nexus-leadership-data
**Raw URL:** `https://raw.githubusercontent.com/jbearup1981/nexus-leadership-data/main/dashboard-data.json`
**Public** — no auth needed for fetch. Data is non-sensitive (no PII, no financials beyond aggregate numbers).

## Pages (8 total)

### Command Central (`/leadership`)
The master dashboard. At-a-glance view of the entire business.
- **Domain cards grid** — one card per domain with color accent, headline, active project count. Click navigates to domain page.
- **Systems status panel** — all Nexus systems ([[NexHub CRM|NexHub]], [[nexmail|NexMail]], [[nexbench-hris|NexBench]], etc.) with online/offline indicators.
- **Recent activity feed** — last 5 things that happened across the business.
- **Quick links** — jump to EOS dashboard or any domain page.
- **JSON key:** `commandCentral`

### EOS Dashboard (`/leadership/eos`)
The company operating system view, loosely following the EOS framework from *Traction*.
- **Summary stats row** — rocks on track, to-dos done, issues count (computed from data).
- **Scorecard** — weekly/monthly measurables with target, current, owner, status, notes. Tracks: revenue, active clients, advisors, commissions, retention, automation jobs, blog output, reconciliation gap.
- **Rocks** — 90-day priorities with status (complete/on track/behind/not started), owner, due date, notes.
- **Issues List** — IDS (Identify, Discuss, Solve) format. Priority badges (high/medium/low), owner, notes.
- **To-Dos** — 7-day action items with checkbox, owner, due date, completion progress bar.
- **Accountability Chart** — team cards with initials, name, role, type badge (Leadership/Advisor/Support), responsibilities.
- **JSON key:** `eos`

### Domain Pages (6 pages)
Each domain of the business gets its own page following a shared template. All powered by the same `DomainPage` React component, just different data.

| Route | Domain | What It Shows |
|-------|--------|---------------|
| `/leadership/sales` | **Sales** | Pipeline status, active deals, prospect tracking, upcoming sales meetings, client work, resources. Metrics: pipeline value, active prospects, deals in progress, win rate. |
| `/leadership/finance` | **Finance** | Revenue, commissions by month, agent pay, expenses. Emphasized metrics cards. Tracks [[commission-tracker|commission tracker]] status, [[financial-cleanup|financial cleanup]], tax compliance. |
| `/leadership/marketing` | **Marketing** | Blog calendar, content pipeline, campaign status, website metrics. Tracks posts published, drafts ready, social media, ACG panel. |
| `/leadership/operations` | **Operations** | Business processes, team onboarding, compliance, client implementations, [[cobra-notice-requirements-and-penalties|COBRA]]/FSA/HRA. Tracks [[va-onboarding|VA onboarding]], [[nexbench-hris|NexBench]] HRIS, [[renewal-process|renewal process]]. |
| `/leadership/technology` | **Technology** | Systems health panel (all Nexus systems + automation jobs), CRM status, [[nexmail|NexMail]] status, infrastructure projects. Special "System Health" section with online/offline indicators + job counts. |
| `/leadership/talent` | **Talent & Training** | Recruiting pipeline, onboarding progress, team development, training programs. Tracks advisor pipeline, contractor onboarding, Enneagram, mentoring. |

**Each domain page includes:**
- **Header** — domain name + summary sentence
- **Metrics cards** — 3-4 KPIs specific to that domain
- **Projects table** — name, status, priority, next action, target date
- **Resources grid** — internal tools, docs, links relevant to that domain
- **Domain-specific sections** — meetings (sales), system health (technology), emphasized financials (finance)

**JSON keys:** `domains.sales`, `domains.finance`, `domains.marketing`, `domains.operations`, `domains.technology`, `domains.talent`

## Website Tech Stack (confirmed from Replit agent)

| Component | Detail |
|-----------|--------|
| Frontend | React + TypeScript + Vite |
| UI Library | Tailwind CSS + shadcn/ui (Card, Badge, Button, Progress, etc.) |
| Backend | Express.js (same origin, same port) |
| Data fetching | TanStack Query (`useQuery`) |
| Routing | Explicit in `App.tsx` — lazy import + Route entry |
| Existing EOS | `client/src/pages/team/TeamEOS.tsx` (hardcoded, being replaced) |
| Auth | `sessionStorage` — new `leadership-auth` key + server-side `POST /api/leadership/auth` |
| Fonts | DM Serif Display (headings), system sans (body) |
| Palette | Forest & Rust — `#1F3D2E` (green), `#B8522A` (rust), warm cream bg |
| Icons | Lucide React |
| Hosting | Standalone on [[Replit]] (no GitHub CI/CD) |

## Data File

**`dashboard-data.json`** — 1,094 lines of structured data. Top-level keys:

```json
{
  "meta": { "lastUpdated", "updatedBy", "version" },
  "eos": { "scorecard", "rocks", "issues", "todos", "accountability" },
  "domains": { "sales", "finance", "marketing", "operations", "technology", "talent" },
  "commandCentral": { "systemsOnline", "recentActivity", "domainSummaries" },
  "advisorMetrics": { "<advisor_name>": { "current_week", "prior_week", "month_to_date" } }
}
```

### Advisor Metrics (`advisorMetrics`)
Per-advisor activity data fed from [[nexmail|NexMail]]'s `advisor_metrics.py` collector. Source files at `~/.config/[[M365|m365]]-mcp/advisor_metrics/`.

**Per advisor, three time windows:**
- `current_week` — Mon-today (rolling)
- `prior_week` — last full Mon-Sun
- `month_to_date` — 1st of month through today

**Fields per window:**
```json
{
  "emails_sent": 42,
  "emails_received": 187,
  "emails_client": 35,
  "meetings_total": 8,
  "meetings_client": 5,
  "meeting_hours": 12.5,
  "unique_clients_touched": 14
}
```

**Generated by:** `advisor_metrics.py export_dashboard()` → writes to `advisorMetrics` key in `dashboard-data.json`.
**Consumed by:** Leadership Dashboard team page, domain advisor cards, EOS accountability view.

**Status values used across the JSON:**
- Scorecard: `on_track`, `in_progress`, `at_risk`, `complete`, `not_started`
- Rocks: `complete`, `on_track`, `behind`, `not_started`
- Projects: `active`, `queued`, `parked`, `complete`, `blocked`
- Issues priority: `high`, `medium`, `low`

## Key Files

| File | Purpose |
|------|---------|
| `CLAUDE.md` | This doc — project entry point |
| `dashboard-data.json` | Live data file, pushed to GitHub |
| `[[Replit|replit]]-agent-instructions.md` | Complete build spec for the [[Replit]] agent |

## Data Refresh Process

**How Claude updates the dashboard:**
1. During session wrapup (or on request), regenerate `dashboard-data.json` from vault data
2. `git add . && git commit && git push` to the GitHub repo
3. Website picks up new data within 5 minutes (TanStack Query `staleTime`)

**Data sources:**
- Scorecard metrics → vault dashboard + [[commission-tracker|commission tracker]] API + CRM
- Rocks → `DASHBOARD.md` active projects + target dates
- Issues → `DASHBOARD.md` blockers + open items
- To-Dos → `DASHBOARD.md` "This Week" section
- Accountability → memory (team roster) + vault docs
- Domain projects → `PROJECTS.md` + individual project CLAUDE.md files
- Domain metrics → various (commissions from tracker, pipeline from CRM, blog from marketing docs)

## Current Status

- [x] Scraped existing EOS dashboard via [[Playwright]] (Mar 14)
- [x] Read website codebase — confirmed React/TS/Vite/shadcn/ui stack
- [x] Project folder created at `3-operations/nexus-leadership-dashboard/`
- [x] JSON data file built with current vault data (1,094 lines, 8 scorecard, 9 rocks, 8 issues, 18 todos, 10 team, 6 domains)
- [x] [[Replit]] agent instructions written with accurate tech details
- [x] GitHub repo created — `jbearup1981/nexus-leadership-data` — raw URL verified working
- [x] **Hand off to [[Replit]] agent** — gave `[[Replit|replit]]-agent-instructions.md` + `dashboard-data.json`
- [x] [[Replit]] agent builds all 8 pages (per Jason — built, awaiting deploy)
- [x] Data refresh automated — Domain Agents push daily at 5:45/6:00 AM via `dashboard_builder.py` → GitHub
- [ ] **Deploy website** — Jason needs to deploy the [[Replit]] project to make pages live
- [ ] Test on desktop + phone after deploy
- [ ] Add financial data sources (revenue/ARR needs a source beyond commissions)

## Open Items

- **Deploy** — pages are built but website hasn't been redeployed yet
- **Financial depth** — commissions come from tracker API, but revenue/ARR/expenses need a source ([[QuickBooks]]?)
- **Repo visibility** — currently public. Consider private + GitHub token if data gets more sensitive.
- **Calendar integration** — domain pages could show upcoming meetings from [[M365]] for that domain's contacts. Would need the data refresh to pull calendar data.
- **Historical tracking** — could version the JSON over time to show trends in scorecard metrics.

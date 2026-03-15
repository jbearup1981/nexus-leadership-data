---
tags: [operations, technology, website, dashboard]
---

# Replit Agent Instructions — Nexus Leadership Dashboard

## Overview

We're adding a private leadership dashboard section to the [[nexus-website|Nexus website]]. The current EOS dashboard has hardcoded data. We're replacing it with a data-driven system: a JSON file hosted on GitHub, fetched via TanStack Query on page load, rendered by React + shadcn/ui components.

This gives us 8 private pages behind a shared password gate, all powered by one JSON data file that gets updated automatically by our AI operations system.

## What Exists Today

- **Current EOS component:** `client/src/pages/team/TeamEOS.tsx`
- **Ops hub wrapper:** `client/src/pages/team/TeamOps.tsx`
- **Auth:** `sessionStorage.getItem('team-auth') === '1'`
- **Design:** "Forest & Rust" palette — dark green `#1F3D2E`, rust `#B8522A`, warm cream backgrounds
- **Fonts:** DM Serif Display (headings), system sans-serif (body)
- **Components:** Tailwind CSS + shadcn/ui (Card, Badge, Button, Progress, etc.)
- **Data fetching pattern:** TanStack Query (`useQuery`)
- **Routing:** Explicit route config in `App.tsx` — lazy import + `<Route>` entry

## What We Want

### Architecture Change

Replace hardcoded data with a TanStack Query fetch to a GitHub-hosted JSON endpoint:

```typescript
const { data, isLoading, error } = useQuery({
  queryKey: ['leadership-dashboard'],
  queryFn: () => fetch('https://raw.githubusercontent.com/jbearup1981/nexus-leadership-data/main/dashboard-data.json')
    .then(r => r.json()),
  staleTime: 5 * 60 * 1000, // refresh every 5 minutes
});
```

### New Route Structure

All pages under `/leadership/` prefix with a shared `LeadershipGuard` auth wrapper.

| Route | Page | JSON Key |
|-------|------|----------|
| `/leadership` | [[Command Central]] (master dashboard) | `commandCentral` |
| `/leadership/eos` | EOS Dashboard (redesigned) | `eos` |
| `/leadership/sales` | Sales Domain | `domains.sales` |
| `/leadership/finance` | Finance Domain | `domains.finance` |
| `/leadership/marketing` | Marketing Domain | `domains.marketing` |
| `/leadership/operations` | Operations Domain | `domains.operations` |
| `/leadership/technology` | Technology Domain | `domains.technology` |
| `/leadership/talent` | Talent & Training Domain | `domains.talent` |

**Keep `/eos-dashboard` as a redirect to `/leadership/eos` for backwards compatibility.**

### Auth — LeadershipGuard Wrapper

Create a `LeadershipGuard` component that wraps all `/leadership/*` routes:

```typescript
// One login gates all 8 pages
// sessionStorage key: 'leadership-auth'
// Password: validated server-side via POST /api/leadership/auth
// On success: sessionStorage.setItem('leadership-auth', '1')
```

Add a server-side endpoint:
```typescript
app.post("/api/leadership/auth", (req, res) => {
  if (req.body.password === "leadership") {
    return res.json({ success: true });
  }
  return res.status(401).json({ error: "Unauthorized" });
});
```

### Shared Layout

All `/leadership/*` pages share:

1. **Top nav bar** — Nexus logo + "Leadership Dashboard" + nav links to all 8 pages (use shadcn NavigationMenu or simple links)
2. **LeadershipGuard** — checks `sessionStorage`, shows login form if not authenticated
3. **Footer** — "Nexus Benefit Solutions · Leadership Dashboard · Last updated: {meta.lastUpdated}"
4. **Responsive** — works on desktop, tablet, and phone

### Page Designs

#### 1. Command Central (`/leadership`)

The home page. At-a-glance view of the entire business.

**Layout:**
- **Header:** "[[Command Central]]" with subtitle "Nexus Benefit Solutions — Operational Overview"
- **Domain Cards Grid** (responsive 2x3 → 1-col on mobile): One shadcn `Card` per domain, each showing:
  - Domain name with colored left border (use `commandCentral.domainSummaries[].color`)
  - Headline text (e.g., "Pipeline building, [[Howard]] meeting Mar 21")
  - Active project count
  - Click navigates to that domain's page
- **Systems Status Panel:** shadcn `Card` with list of systems + online/offline `Badge` indicators (from `commandCentral.systemsOnline`)
- **Recent Activity Feed:** Last 5 items (from `commandCentral.recentActivity`) — date + description
- **Quick Links:** Jump to EOS dashboard, specific domain pages

#### 2. EOS Dashboard (`/leadership/eos`)

Redesigned version of the current TeamEOS page. Same sections, data-driven.

**Sections:**
- **Summary Stats Row** — 3 shadcn `Card` components: Rocks on track (computed), To-dos done (computed), Issues count (computed)
- **Scorecard Table** — from `eos.scorecard[]` — measurable, target, current, owner, status `Badge`, notes
- **Rocks** — from `eos.rocks[]` — name, status badge, owner, due date, note. Group by status (complete first, then on_track, behind, not_started)
- **Issues List** — from `eos.issues[]` — name, priority `Badge` (red for high, yellow for medium, gray for low), owner, note
- **To-Dos** — from `eos.todos[]` — checkbox-style list, name, owner, due date. Show `Progress` bar for completion ratio
- **Accountability Chart** — from `eos.accountability[]` — shadcn `Card` per person with initials avatar, name, role, type badge (Leadership/Advisor/Support), responsibilities list

#### 3. Domain Pages (`/leadership/{domain}`)

All 6 domain pages use the same `DomainPage` component, just pass different data:

**Layout:**
- **Header:** Domain name (DM Serif Display heading) + summary text
- **Metrics Cards Row:** 3-4 shadcn `Card` components (from `domains.{domain}.metrics`) — label, value, optional description
- **Projects Table:** from `domains.{domain}.projects[]` — shadcn table or card list. Columns: name, status `Badge`, priority `Badge`, next action, target date
- **Resources Section:** from `domains.{domain}.resources[]` — grid of shadcn `Card` components (name, description). Internal tools and links for this domain.

**Domain-specific additions (render conditionally if data exists):**
- **Sales:** "Upcoming Meetings" section from `domains.sales.upcomingMeetings[]`
- **Technology:** "System Health" panel from `domains.technology.systemHealth`
- **Finance:** Larger/emphasized metrics cards for revenue, commissions, expenses

### JSON Data Format

The complete JSON file is at `dashboard-data.json` in this folder. Key status values:

| Context | Values |
|---------|--------|
| Scorecard | `on_track`, `in_progress`, `at_risk`, `complete`, `not_started` |
| Rocks | `complete`, `on_track`, `behind`, `not_started` |
| Projects | `active`, `queued`, `parked`, `complete`, `blocked` |
| Issues | `high`, `medium`, `low` |

**Color mapping for status badges:**
| Status | Color | Tailwind |
|--------|-------|----------|
| Complete / On Track | Dark green `#1F3D2E` | `bg-green-900/10 text-green-900` |
| In Progress / On Track | Copper `#B8522A` | `bg-orange-700/10 text-orange-700` |
| At Risk / Behind | Red | `bg-red-600/10 text-red-600` |
| Not Started | Sage `#7A9282` | `bg-gray-500/10 text-gray-500` |
| Blocked | Red | `bg-red-700/10 text-red-700` |

### Routing Setup in App.tsx

```typescript
// Add lazy imports
const LeadershipLayout = lazy(() => import("@/pages/leadership/LeadershipLayout"));
const CommandCentral = lazy(() => import("@/pages/leadership/CommandCentral"));
const LeadershipEOS = lazy(() => import("@/pages/leadership/EOSDashboard"));
const DomainPage = lazy(() => import("@/pages/leadership/DomainPage"));

// Add routes (inside Router)
<Route path="/leadership" component={LeadershipLayout}>
  <Route path="/" component={CommandCentral} />
  <Route path="/eos" component={LeadershipEOS} />
  <Route path="/sales" component={() => <DomainPage domain="sales" />} />
  <Route path="/finance" component={() => <DomainPage domain="finance" />} />
  <Route path="/marketing" component={() => <DomainPage domain="marketing" />} />
  <Route path="/operations" component={() => <DomainPage domain="operations" />} />
  <Route path="/technology" component={() => <DomainPage domain="technology" />} />
  <Route path="/talent" component={() => <DomainPage domain="talent" />} />
</Route>

// Redirect old route
<Route path="/eos-dashboard" component={() => { window.location.href = '/leadership/eos'; return null; }} />
```

### File Organization

```
client/src/pages/leadership/
  ├── LeadershipLayout.tsx    (shared nav, LeadershipGuard, footer, useQuery for data)
  ├── CommandCentral.tsx      (home page)
  ├── EOSDashboard.tsx        (EOS page — scorecard, rocks, issues, todos, accountability)
  ├── DomainPage.tsx          (shared template — takes domain prop, renders from domains.{domain})
  └── components/
      ├── StatusBadge.tsx     (reusable badge for all status types)
      ├── MetricCard.tsx      (KPI card for domain metrics)
      ├── ProjectTable.tsx    (project list with status/priority badges)
      ├── ResourceGrid.tsx    (grid of resource cards)
      └── SystemHealthPanel.tsx (technology domain system status)
```

### Migration Notes

- Existing `client/src/pages/team/TeamEOS.tsx` stays as-is until the new pages are confirmed working, then can be removed or redirected
- The `LeadershipGuard` is separate from the existing `team-auth` — different sessionStorage key (`leadership-auth`), different password
- Server-side auth endpoint: `POST /api/leadership/auth` in `server/routes.ts`

## Summary

1. Create `LeadershipGuard` + server-side auth endpoint
2. Fetch JSON from GitHub via TanStack Query (`useQuery`)
3. Build 8 pages under `/leadership/*` using shadcn/ui components
4. Shared layout with nav, auth, footer
5. Domain pages all use one `DomainPage` template component
6. [[Command Central]] is the overview home page
7. EOS Dashboard replaces hardcoded TeamEOS with data-driven version
8. Keep the Forest & Rust brand — extend the existing design language
9. Redirect `/eos-dashboard` → `/leadership/eos`

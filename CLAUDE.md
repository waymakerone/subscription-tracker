# Subscription Tracker

Track every SaaS subscription, monitor costs, get renewal alerts, and identify redundancies — powered by Commander Tables, Calendar, and Spreadsheets.

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React + TypeScript + Vite + Tailwind CSS |
| Auth | Clerk (via WaymakerOS) |
| Data | Commander Tables (Supabase PostgreSQL) |
| Calendar | Commander Calendar (renewal events + alerts) |
| Spreadsheets | Commander Spreadsheets (cost reports + budget tracking) |
| Email | Commander Journeys (renewal reminders) |
| Charts | recharts |
| Hosting | Waymaker Host (EX app) |

## Documentation

| Folder | Contents |
|--------|----------|
| `docs/01-planning/product-requirements/` | PRD — data model, views, integration map |
| `docs/02-working/prompts/active/` | Build prompts — 4 phases with YAML front matter |
| `docs/02-working/sessions/` | Session briefs as you build |
| `docs/03-knowledge/` | Patterns discovered during the build |

## Build Phases

| Phase | Prompt | Status |
|-------|--------|--------|
| 1 — Subscription Register | `docs/02-working/prompts/active/phase-1-subscription-register.md` | todo |
| 2 — Renewal Calendar & Alerts | `docs/02-working/prompts/active/phase-2-renewal-calendar.md` | todo |
| 3 — Cost Analysis & Spreadsheets | `docs/02-working/prompts/active/phase-3-cost-analysis.md` | todo |
| 4 — Dashboard & Polish | `docs/02-working/prompts/active/phase-4-dashboard-and-polish.md` | todo |

## Data Model (Quick Reference)

```
st_categories ──< st_subscriptions ──< st_cost_history
                        │
                   renewal events
                   (Commander Calendar)
```

- **Category** defines subscription types (Design, Engineering, Marketing) with optional monthly budgets
- **Subscription** is the core record — one row per SaaS tool, with cost, billing cycle, renewal date, status, and seat tracking
- **Cost History** tracks price changes over time for trend analysis and annual projections

## Commander Integration

| Action | Commander Tool | API |
|--------|---------------|-----|
| All subscription data | Tables | `commander-table-operations` → CRUD |
| Schema creation | Tables | `commander-table-ddl` → CREATE TABLE |
| Renewal events | Calendar | `commander-calendar` → create/update/delete events |
| Renewal reminders | Email/Journeys | `journeys-send-email` → 30/7/1 day reminders |
| Cost reports | Spreadsheets | `commander-sheet-operations` → monthly summary sheets |
| Spend targets | Goals | `commander-goals-operations` → link to OKRs |
| Reminder automation | Automations | `commander-automation-operations` → scheduled alerts |

## Critical Rules

- All data lives in Commander Tables — the app is the view + logic layer
- Auth via Clerk: `useAuth().getToken()` → Bearer token on all API calls
- API calls POST to `${SUPABASE_URL}/functions/v1/{function-name}` with `{ action, data }` body
- Renewal events sync to Commander Calendar — changes to renewal dates must update calendar events
- Cost normalisation: all costs displayed as monthly equivalent (annual / 12, quarterly / 3) for comparison
- Design tokens: Sky blue `#0EA5E9` primary, White `#FFFFFF` background, Green `#059669` active/healthy, Amber `#D97706` renewing soon, Red `#DC2626` expired/cancelled
- Use Geist font family
- All tables scoped by `organization_id` with RLS policies

## How to Build

1. Read the PRD: `docs/01-planning/product-requirements/subscription-tracker-prd.md`
2. Work through each phase prompt in order
3. Update the YAML `status` field as you go: `todo` → `in-progress` → `review` → `done`
4. Write session briefs in `docs/02-working/sessions/completed/` between sessions

## Quick Reference

| What | How |
|------|-----|
| Dev server | `npm run dev` |
| Build | `npm run build` |
| Tables API | `commander-table-operations` → `query`, `insert`, `update`, `delete` |
| Schema API | `commander-table-ddl` → `create_table`, `alter_table` |
| Calendar API | `commander-calendar` → `create`, `update`, `delete` |
| Spreadsheets API | `commander-sheet-operations` → `create`, `update` |
| Email API | `journeys-send-email` → renewal reminders |
| Goals API | `commander-goals-operations` → spend targets |

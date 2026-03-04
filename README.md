# Subscription Tracker

A complete SaaS subscription management app built on WaymakerOS. Track every subscription, monitor costs, get renewal alerts, detect redundancies, and manage budgets — all integrated with Commander Tables, Calendar, Spreadsheets, and Journeys.

## What You Get

- **Subscription List** — All subscriptions in a searchable, filterable table with status badges, cost, renewal dates, and seat utilisation
- **Category Management** — Organise subscriptions by department (Design, Engineering, Marketing) with optional monthly budgets
- **Renewal Calendar** — Month/week calendar view of upcoming renewals synced to Commander Calendar
- **Renewal Alerts** — Automated email reminders at 30, 7, and 1 day before renewal via Commander Journeys
- **Cost Analysis** — Spend by category donut chart, monthly trend line, annual projections, and budget vs actual tracking
- **Redundancy Detection** — Flag multiple tools in the same category with low seat utilisation
- **Dashboard** — Executive summary with total spend, upcoming renewals, cost trends, and alerts

## Commander Tools Used

| Tool | How the Subscription Tracker Uses It |
|------|------|
| **Tables** | All subscription data (subscriptions, categories, cost history) |
| **Calendar** | Renewal date events with visual timeline |
| **Spreadsheets** | Monthly cost reports, budget vs actual summaries |
| **Email/Journeys** | Automated renewal reminder emails |
| **Automations** | Scheduled renewal alert triggers |
| **Goals** | Link spend reduction targets to OKRs |

## How to Build

1. Clone this blueprint into your project
2. Open `CLAUDE.md` — it's the router file for your AI coding tool
3. Read the PRD in `docs/01-planning/product-requirements/`
4. Work through the 4 phase prompts in `docs/02-working/prompts/active/`
5. Point Claude Code, Cursor, or Codex at each phase and build

**Estimated build time:** 2-3 hours across all 4 phases.

## Build Phases

| Phase | What You Get |
|-------|-------------|
| 1 — Subscription Register | App scaffold, schema (3 tables), subscription CRUD, list with filters, category management |
| 2 — Renewal Calendar & Alerts | Calendar view of renewals, upcoming renewal panel, email reminders at 30/7/1 days |
| 3 — Cost Analysis & Spreadsheets | Spend charts, monthly trends, annual projections, budget vs actual, Spreadsheet export |
| 4 — Dashboard & Polish | Executive dashboard, redundancy detection, CSV export, mobile responsive, quick actions |

## Prerequisites

- WaymakerOS organization with Commander access
- Commander Tables enabled
- Commander Calendar enabled
- (Optional) Commander Spreadsheets for cost reports
- (Optional) Commander Journeys for email reminders

# Subscription Tracker — Product Requirements

**Status:** Approved
**Author:** Waymaker
**Date:** 2026-03-04
**Last Updated:** 2026-03-04

## Problem Statement

Every growing organisation accumulates SaaS subscriptions. What starts as a few tools — Slack, Figma, GitHub — quietly becomes 30, 50, or 100+ subscriptions spread across credit cards, invoices, and department budgets. The pain is universal: no one knows the true total spend, renewals surprise finance teams, unused seats waste money, and redundant tools overlap without anyone noticing.

The market has recognised this: Zylo, Torii, and Cledara all charge $5-15/seat/month for subscription management. But they're standalone silos — another subscription to manage your subscriptions. They don't connect to your project management, your calendar, your team's goals, or your budget tracking.

This blueprint builds a subscription tracker as a WaymakerOS app. Subscriptions live in Commander Tables (the same database that holds tasks, goals, and projects). Renewal dates sync to Commander Calendar. Renewal reminders go out through Commander Journeys. Cost reports export to Commander Spreadsheets. Spend reduction targets link to Commander Goals. The result is subscription management that's part of the operating system, not another standalone tool.

## Goals

1. **One register for all subscriptions** — Every SaaS tool recorded with cost, billing cycle, renewal date, status, owner, and seat utilisation
2. **Renewal visibility** — Calendar view of all upcoming renewals with automated email reminders at 30, 7, and 1 day before renewal
3. **Cost analysis** — Spend breakdown by category, monthly trends, annual projections, and budget vs actual tracking
4. **Redundancy detection** — Flag multiple tools in the same category with overlapping features and low seat utilisation
5. **Category budgets** — Set monthly spend limits per department/category and track actuals against them

## Non-Goals

- This is NOT a procurement tool — no purchase order workflows or vendor negotiations
- This is NOT a contract management system — no legal clause tracking or contract storage
- This is NOT an SSO/identity management tool — no automatic user provisioning or deprovisioning
- This is NOT a bank feed integration — subscription costs are entered manually or via cost history, not imported from bank transactions
- This is NOT an app usage analytics platform — seat utilisation is manually tracked, not measured via login data

## Data Model

### Core Objects

**Subscription** — A single SaaS subscription the organisation pays for.
- Core fields: name, vendor, cost_amount, cost_currency, billing_cycle, next_renewal_date, start_date
- Status: active, cancelled, trial, expired, paused
- Ownership: owner_id (Clerk user ID — person responsible for this subscription)
- Seats: seats_purchased, seats_used (for utilisation tracking)
- Renewal: auto_renew, cancellation_notice_days, end_date (for fixed-term)
- Metadata: login_url, notes, tags

**Category** — A way to classify subscriptions by department or function. Seeded with defaults, user-editable.
- Fields: name, icon, colour, budget_monthly (nullable), is_active
- Seeds: Design, Engineering, Marketing, Sales, Operations, HR, Finance, Other

**Cost History** — Tracks price changes over time for a subscription.
- Fields: subscription_id, amount, currency, effective_date, change_reason
- Used for: trend analysis, annual projections, detecting price increases

### Tables Schema (Commander Tables)

```
st_subscriptions
├── id (uuid, PK)
├── organization_id (text, FK → Clerk org)
├── name (text, required — e.g. "Slack", "Figma")
├── vendor (text, nullable)
├── category_id (uuid, FK → st_categories)
├── cost_amount (numeric, required)
├── cost_currency (text, default 'USD')
├── billing_cycle (text: monthly, quarterly, annual)
├── next_renewal_date (date, required)
├── start_date (date, required)
├── end_date (date, nullable — for fixed-term)
├── status (text: active, cancelled, trial, expired, paused)
├── owner_id (text, nullable, Clerk user ID)
├── seats_purchased (integer, nullable)
├── seats_used (integer, nullable)
├── login_url (text, nullable)
├── notes (text, nullable)
├── tags (text[], nullable)
├── auto_renew (boolean, default true)
├── cancellation_notice_days (integer, nullable)
├── created_at (timestamptz)
└── updated_at (timestamptz)

st_categories
├── id (uuid, PK)
├── organization_id (text)
├── name (text, required)
├── icon (text, nullable)
├── colour (text, nullable — hex code)
├── budget_monthly (numeric, nullable)
├── is_active (boolean, default true)
├── created_at (timestamptz)
└── updated_at (timestamptz)

st_cost_history
├── id (uuid, PK)
├── organization_id (text)
├── subscription_id (uuid, FK → st_subscriptions)
├── amount (numeric, required)
├── currency (text)
├── effective_date (date, required)
├── change_reason (text, nullable — e.g. "Price increase", "Upgraded plan")
└── created_at (timestamptz)
```

### Default Category Seeds

| Name | Icon | Colour |
|------|------|--------|
| Design | palette | #8B5CF6 |
| Engineering | code | #3B82F6 |
| Marketing | megaphone | #F59E0B |
| Sales | handshake | #10B981 |
| Operations | settings | #6B7280 |
| HR | users | #EC4899 |
| Finance | calculator | #06B6D4 |
| Other | circle | #9CA3AF |

### Commander Integration Map

| Action | Commander Tool | How |
|--------|---------------|-----|
| All subscription data | Tables | `commander-table-operations` → CRUD on st_subscriptions, st_categories, st_cost_history |
| Schema creation | Tables | `commander-table-ddl` → CREATE TABLE for all 3 tables |
| Renewal calendar events | Calendar | `commander-calendar` → create/update/delete renewal events |
| Renewal reminders | Email/Journeys | `journeys-send-email` → automated emails at 30, 7, 1 day before renewal |
| Reminder scheduling | Automations | `commander-automation-operations` → schedule renewal check triggers |
| Cost report sheets | Spreadsheets | `commander-sheet-operations` → monthly spend summary, budget vs actual |
| Spend targets | Goals | `commander-goals-operations` → link cost reduction targets to OKRs |

## Proposed Solution

### Overview

An internal (EX) app deployed to Waymaker Host that provides complete SaaS subscription management: subscription register, renewal calendar with alerts, cost analysis with charts, redundancy detection, and budget tracking. All data lives in Commander Tables. Renewal events sync to Commander Calendar. Reminders via Commander Journeys. Cost reports via Commander Spreadsheets.

The app is the view layer and the subscription logic. Commander is the data layer and the integration layer.

### Key Views

1. **Subscription List** (`/`) — Home page. Table of all subscriptions, filterable by status, category, billing cycle, and owner. Summary stats at top (total monthly spend, active count, upcoming renewals). Quick actions per row.

2. **Add/Edit Subscription** (modal) — Form with all subscription fields: name, vendor, cost, billing cycle, renewal date, category, owner, seats, login URL, notes, tags, auto-renew toggle, cancellation notice days.

3. **Renewals Calendar** (`/renewals`) — Month and week views showing renewal dates. Click a date to see renewal details. "Renewing This Week" and "Renewing This Month" panels. Cancellation deadline warnings.

4. **Cost Analysis** (`/analysis`) — Spend by category donut chart, monthly cost trend line (last 12 months), annual projection, budget vs actual per category, cost history timeline per subscription.

5. **Dashboard** (`/dashboard`) — Executive summary: total monthly/annual spend, upcoming renewals count, average cost per subscription, spend trend chart, category breakdown, renewal timeline, redundancy alerts.

6. **Categories** (`/categories`) — Manage categories: add, edit, deactivate. Each category shows monthly budget (if set) and current month spend. Budget vs actual bar chart.

### User Flow

1. Finance lead opens Subscription Tracker from Host
2. Adds all known subscriptions with costs, billing cycles, and renewal dates
3. Assigns owners to each subscription (who's responsible for this tool?)
4. Reviews the dashboard — sees total monthly spend is $12,400 across 47 subscriptions
5. Spots two design tools (Figma + Sketch) in the Design category — redundancy alert fires
6. Calendar shows 3 renewals next week — email reminders already sent at 30 days
7. Clicks into Cost Analysis — Marketing category is at 95% of budget, amber warning
8. Exports monthly spend summary to Commander Spreadsheet for the CFO
9. Links a "Reduce SaaS spend by 15%" goal in Commander Goals

## Scope

### Phase 1 (MVP) — Subscription Register

- [ ] App scaffold: React + Vite + Tailwind + Clerk auth
- [ ] Tables schema: st_subscriptions, st_categories, st_cost_history (via commander-table-ddl)
- [ ] API layer: CRUD operations for all tables via authenticated edge function
- [ ] Seed default subscription categories
- [ ] Subscription list with search, filter by status/category/billing cycle/owner
- [ ] Add/edit subscription form with all fields
- [ ] Cost normalisation helper (monthly equivalent for annual/quarterly)
- [ ] Status lifecycle (active, cancelled, trial, expired, paused)
- [ ] Seat utilisation display (seats_used / seats_purchased)
- [ ] Category management page (list, add, edit, deactivate)

### Phase 2 — Renewal Calendar & Alerts

- [ ] Calendar view (month/week) showing renewal dates
- [ ] Sync renewal events to Commander Calendar (create/update/delete)
- [ ] "Renewing This Week" and "Renewing This Month" panels
- [ ] Email reminder automation (30, 7, 1 day before renewal) via Commander Journeys
- [ ] Cancellation deadline alerts (if cancellation_notice_days set)
- [ ] Auto-renew indicator on calendar events
- [ ] Automation setup via commander-automation-operations

### Phase 3 — Cost Analysis & Spreadsheets

- [ ] Spend by category donut chart (recharts)
- [ ] Monthly cost trend line chart (last 12 months)
- [ ] Annual projection (based on current active subscriptions)
- [ ] Cost history tracking (price changes over time per subscription)
- [ ] Commander Spreadsheet export — monthly spend summary sheet
- [ ] Budget vs actual per category with progress bars
- [ ] Redundancy detection (multiple tools in same category with low utilisation)
- [ ] Goal integration — link spend targets to Commander Goals

### Phase 4 — Dashboard & Polish

- [ ] Executive dashboard with key metrics and charts
- [ ] Redundancy alerts (same category, overlapping features, low utilisation)
- [ ] Export all subscriptions to CSV
- [ ] Mobile responsive (card view for subscription list)
- [ ] Quick actions (renew, cancel, pause)
- [ ] waymaker.config.ts manifest for Host Schema
- [ ] Deploy-ready configuration

### Out of Scope

- Bank feed or credit card integration (automatic cost import)
- SSO/identity management (automatic user provisioning tracking)
- Contract document storage and legal clause tracking
- Vendor negotiation workflows or procurement approvals
- App usage analytics via login monitoring
- Multi-currency conversion (user enters the local amount)

## Success Criteria

| Metric | Target |
|--------|--------|
| Subscription entry | < 60 seconds to add a subscription with all key fields |
| Renewal visibility | All renewals visible on calendar within 1 click from home |
| Cost overview | Total monthly spend visible on dashboard within 2 seconds |
| Redundancy detection | Overlapping tools flagged automatically by category + utilisation |
| Build time (with AI) | < 3 hours for all 4 phases |

## Dependencies

- WaymakerOS organization with Commander access
- Commander Tables for data storage
- Commander Calendar for renewal events
- (Optional) Commander Journeys for email reminders
- (Optional) Commander Spreadsheets for cost reports
- (Optional) Commander Goals for spend targets

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Manual data entry burden | Medium | Medium | Bulk import in Phase 1; future: bank feed integration |
| Stale seat utilisation data | Medium | Low | Manual update prompts; out of scope: automatic tracking |
| Calendar sync failures | Low | Medium | Retry logic; calendar events are secondary to table data |
| Email reminder not delivered | Low | Low | Phase 2 feature — calendar view shows renewals without email |

## Open Questions

- [x] Internal (EX) or external (CX)? **EX — team members only**
- [x] Multi-currency support? **Out of scope — user enters local amount**
- [x] Bank feed integration? **Out of scope — manual entry + cost history**
- [ ] Vendor logo fetching? **Nice to have — could use Clearbit Logo API or manual upload**

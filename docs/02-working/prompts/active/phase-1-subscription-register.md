---
sync:
  type: doc
  layer: Subscription Tracker
build:
  status: todo
  phase: 1
  priority: P0
  depends_on: []
  started_at: null
  completed_at: null
---

# Phase 1: Subscription Register

**Goal:** App scaffold with auth, data schema (3 tables), subscription CRUD, filterable list view, category management with budgets, cost normalisation, and seed data.

**PRD Reference:** `docs/01-planning/product-requirements/subscription-tracker-prd.md` — Phase 1

---

## What to Build

### 1. Project Setup

Create a React + Vite + TypeScript + Tailwind app.

**Files:**
- `src/main.tsx` — Clerk provider wrapper
- `src/App.tsx` — Router with routes: `/`, `/renewals`, `/analysis`, `/categories`, `/dashboard`
- `src/lib/api.ts` — Authenticated fetch helper for Commander Tables
- `src/lib/types.ts` — TypeScript interfaces for Subscription, Category, CostHistory
- `src/lib/costs.ts` — Cost normalisation helpers (monthly equivalent calculations)
- `src/index.css` — Tailwind base + design tokens

**Auth pattern:**
```typescript
import { useAuth } from '@clerk/clerk-react'

const { getToken } = useAuth()
const token = await getToken()

const res = await fetch(`${SUPABASE_URL}/functions/v1/commander-table-operations`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`,
  },
  body: JSON.stringify({ action: 'query', data: { table: 'st_subscriptions', filters: {} } }),
})
```

### 2. Database Schema

Create these 3 tables using `commander-table-ddl`. Use the API to run DDL operations.

**st_categories** — Seed with defaults:

| name | icon | colour | budget_monthly |
|------|------|--------|---------------|
| Design | palette | #8B5CF6 | null |
| Engineering | code | #3B82F6 | null |
| Marketing | megaphone | #F59E0B | null |
| Sales | handshake | #10B981 | null |
| Operations | settings | #6B7280 | null |
| HR | users | #EC4899 | null |
| Finance | calculator | #06B6D4 | null |
| Other | circle | #9CA3AF | null |

**st_subscriptions** — See PRD for full schema. Key fields: name, vendor, cost_amount, cost_currency, billing_cycle, next_renewal_date, start_date, end_date, status, owner_id, seats_purchased, seats_used, auto_renew, cancellation_notice_days.

**st_cost_history** — See PRD. Tracks price changes over time. Each record links to a subscription with amount, effective_date, and change_reason.

**DDL API pattern:**
```typescript
const res = await fetch(`${SUPABASE_URL}/functions/v1/commander-table-ddl`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`,
  },
  body: JSON.stringify({
    action: 'create_table',
    data: {
      table_name: 'st_subscriptions',
      columns: [
        { name: 'id', type: 'uuid', primary_key: true, default: 'gen_random_uuid()' },
        { name: 'organization_id', type: 'text', nullable: false },
        { name: 'name', type: 'text', nullable: false },
        { name: 'vendor', type: 'text', nullable: true },
        { name: 'category_id', type: 'uuid', nullable: true },
        { name: 'cost_amount', type: 'numeric', nullable: false },
        { name: 'cost_currency', type: 'text', nullable: false, default: "'USD'" },
        { name: 'billing_cycle', type: 'text', nullable: false },
        { name: 'next_renewal_date', type: 'date', nullable: false },
        { name: 'start_date', type: 'date', nullable: false },
        { name: 'end_date', type: 'date', nullable: true },
        { name: 'status', type: 'text', nullable: false, default: "'active'" },
        { name: 'owner_id', type: 'text', nullable: true },
        { name: 'seats_purchased', type: 'integer', nullable: true },
        { name: 'seats_used', type: 'integer', nullable: true },
        { name: 'login_url', type: 'text', nullable: true },
        { name: 'notes', type: 'text', nullable: true },
        { name: 'tags', type: 'text[]', nullable: true },
        { name: 'auto_renew', type: 'boolean', nullable: false, default: 'true' },
        { name: 'cancellation_notice_days', type: 'integer', nullable: true },
        { name: 'created_at', type: 'timestamptz', nullable: false, default: 'now()' },
        { name: 'updated_at', type: 'timestamptz', nullable: false, default: 'now()' },
      ],
    },
  }),
})
```

### 3. TypeScript Interfaces

**File:** `src/lib/types.ts`

```typescript
export interface Subscription {
  id: string;
  organization_id: string;
  name: string;
  vendor: string | null;
  category_id: string | null;
  cost_amount: number;
  cost_currency: string;
  billing_cycle: 'monthly' | 'quarterly' | 'annual';
  next_renewal_date: string;  // ISO date
  start_date: string;         // ISO date
  end_date: string | null;    // ISO date, for fixed-term
  status: 'active' | 'cancelled' | 'trial' | 'expired' | 'paused';
  owner_id: string | null;
  seats_purchased: number | null;
  seats_used: number | null;
  login_url: string | null;
  notes: string | null;
  tags: string[] | null;
  auto_renew: boolean;
  cancellation_notice_days: number | null;
  created_at: string;
  updated_at: string;
}

export interface Category {
  id: string;
  organization_id: string;
  name: string;
  icon: string | null;
  colour: string | null;
  budget_monthly: number | null;
  is_active: boolean;
  created_at: string;
  updated_at: string;
}

export interface CostHistory {
  id: string;
  organization_id: string;
  subscription_id: string;
  amount: number;
  currency: string;
  effective_date: string;  // ISO date
  change_reason: string | null;
  created_at: string;
}
```

### 4. Cost Normalisation Helper

**File:** `src/lib/costs.ts`

All costs should be displayable as a monthly equivalent for comparison.

```typescript
/**
 * Normalise any billing cycle amount to a monthly equivalent.
 */
export function toMonthlyAmount(amount: number, billingCycle: 'monthly' | 'quarterly' | 'annual'): number {
  switch (billingCycle) {
    case 'monthly': return amount;
    case 'quarterly': return amount / 3;
    case 'annual': return amount / 12;
  }
}

/**
 * Normalise any billing cycle amount to an annual equivalent.
 */
export function toAnnualAmount(amount: number, billingCycle: 'monthly' | 'quarterly' | 'annual'): number {
  switch (billingCycle) {
    case 'monthly': return amount * 12;
    case 'quarterly': return amount * 4;
    case 'annual': return amount;
  }
}

/**
 * Format currency amount for display.
 */
export function formatCurrency(amount: number, currency: string = 'USD'): string {
  return new Intl.NumberFormat('en-US', { style: 'currency', currency }).format(amount);
}

/**
 * Calculate seat utilisation percentage.
 */
export function seatUtilisation(used: number | null, purchased: number | null): number | null {
  if (used === null || purchased === null || purchased === 0) return null;
  return Math.round((used / purchased) * 100);
}
```

### 5. API Layer

Create a service module for each entity:

```
src/services/
├── subscriptions.ts  — list, get, create, update, delete
├── categories.ts     — list, get, create, update, toggleActive, seed
└── costHistory.ts    — list (by subscription), create
```

All table operations go through `commander-table-operations` edge function:

```typescript
// src/services/subscriptions.ts
import { apiClient } from '../lib/api';

export async function listSubscriptions(filters?: {
  status?: string;
  category_id?: string;
  billing_cycle?: string;
  owner_id?: string;
  search?: string;
}) {
  return apiClient('commander-table-operations', {
    action: 'query',
    data: {
      table: 'st_subscriptions',
      filters: filters || {},
      order_by: [{ column: 'next_renewal_date', direction: 'asc' }],
    },
  });
}

export async function createSubscription(subscription: Omit<Subscription, 'id' | 'created_at' | 'updated_at'>) {
  return apiClient('commander-table-operations', {
    action: 'insert',
    data: {
      table: 'st_subscriptions',
      record: subscription,
    },
  });
}

export async function updateSubscription(id: string, updates: Partial<Subscription>) {
  return apiClient('commander-table-operations', {
    action: 'update',
    data: {
      table: 'st_subscriptions',
      id,
      record: updates,
    },
  });
}

export async function deleteSubscription(id: string) {
  return apiClient('commander-table-operations', {
    action: 'delete',
    data: {
      table: 'st_subscriptions',
      id,
    },
  });
}
```

### 6. Subscription List (/)

The home page. A table of all subscriptions with summary stats and quick actions.

**Summary bar at top:**
- Total monthly spend (normalised — all subscriptions converted to monthly equivalent)
- Active subscriptions (count)
- Upcoming renewals (count — renewing in next 30 days)
- Average cost per subscription (monthly equivalent)

**Table columns:**
- Name (searchable, with vendor in muted text below)
- Category (colored badge matching category colour)
- Cost (formatted currency, right-aligned, with billing cycle label and normalised monthly in muted text)
- Billing Cycle (monthly, quarterly, annual — badge)
- Next Renewal (date, with "X days" relative time)
- Status (badge: active=green, trial=blue, paused=yellow, cancelled=red, expired=grey)
- Owner (user name or "Unassigned")
- Seats (X/Y with mini progress bar, or dash if not tracked)

**Filters:**
- Status dropdown (multi-select: active, cancelled, trial, expired, paused)
- Category dropdown (multi-select, from st_categories)
- Billing cycle dropdown (monthly, quarterly, annual)
- Owner dropdown (team members)
- Search by name or vendor

**Actions:**
- "Add Subscription" button → opens add subscription modal
- Click row → opens subscription detail/edit view
- Row actions menu: Edit, Pause, Cancel, View Cost History

**Empty state:** "No subscriptions tracked yet. Add your first subscription to get started." with Add Subscription button.

### 7. Add/Edit Subscription (Modal)

A modal form for adding or editing a subscription.

**Section 1: Basics**
- Name (text input, required — e.g. "Slack")
- Vendor (text input, optional — e.g. "Slack Technologies")
- Category (dropdown from st_categories where is_active = true, with colour dots)
- Status (dropdown: active, trial, paused — cancelled and expired are set via actions, not form)

**Section 2: Cost & Billing**
- Cost amount (number input, required, 2 decimal places)
- Currency (dropdown, defaults to USD)
- Billing cycle (radio: Monthly, Quarterly, Annual)
- Monthly equivalent (calculated, read-only display)
- Auto-renew (toggle, default on)
- Cancellation notice days (number input, optional — e.g. 30)

**Section 3: Dates**
- Start date (date picker, required)
- Next renewal date (date picker, required)
- End date (date picker, optional — for fixed-term subscriptions)

**Section 4: Ownership & Seats**
- Owner (dropdown of team members, optional)
- Seats purchased (number input, optional)
- Seats used (number input, optional)

**Section 5: Details**
- Login URL (URL input, optional)
- Tags (tag input, optional — comma separated)
- Notes (textarea, optional)

**On submit:**
1. Create/update subscription record in Commander Tables
2. Create initial cost history record (amount + effective_date = start_date)
3. Close modal, refresh subscription list

### 8. Category Management (/categories)

List of all subscription categories with management actions.

**Table/grid view:**
- Category name with icon and colour dot
- Monthly budget (if set, otherwise "No budget")
- Current month spend (calculated from active subscriptions — normalised monthly)
- Subscription count (active subscriptions in this category)
- Active/inactive toggle

**Actions:**
- "Add Category" → modal: name, icon (picker), colour (picker), monthly budget (optional)
- Edit category → same modal in edit mode
- Toggle active/inactive (inactive categories hidden from subscription form dropdown)

**Budget display:**
- If budget set: progress bar showing spend vs budget
- Green if under 80%, yellow if 80-100%, red if over 100%
- Amount text: "$X / $Y" format

**Seed logic:**
- On first load, check if categories exist for this org
- If none, seed the 8 default categories
- Use a `seeded` flag or count check to avoid re-seeding

### 9. Design Tokens

Use a clean, cost-conscious design:
- Font: `Geist` (load from Google Fonts or CDN)
- Primary: `#0EA5E9` (sky blue) — primary buttons, links, active states
- Background: `#FFFFFF` (white) — page background
- Surface: `#F9FAFB` (off-white) — card backgrounds, table rows
- Text: `#111827` (near-black)
- Text Muted: `#6B7280` (grey)
- Success/Active: `#059669` (green) — active status, under-budget indicators
- Warning: `#D97706` (amber) — renewing soon, nearing budget
- Danger: `#DC2626` (red) — expired, cancelled, over-budget
- Trial: `#3B82F6` (blue) — trial status badge
- Paused: `#D97706` (amber) — paused status badge
- Border: `#E5E7EB`
- Cards: white, `border-radius: 12px`, subtle shadow
- Spacing: 8px base grid
- Tabular figures for all monetary amounts

---

## Acceptance Criteria

- [ ] App loads with Clerk auth (sign-in required)
- [ ] 3 tables created via commander-table-ddl: st_subscriptions, st_categories, st_cost_history
- [ ] Default categories seeded on first load (8 categories with icons and colours)
- [ ] Subscription list shows all subscriptions with correct columns
- [ ] Search by name/vendor works
- [ ] Filters work: status, category, billing cycle, owner
- [ ] Summary stats show at top of subscription list (total monthly spend, active count, upcoming renewals, avg cost)
- [ ] Add subscription modal creates a subscription with all fields
- [ ] Edit subscription works via the same modal in edit mode
- [ ] Cost normalisation displays monthly equivalent for quarterly and annual subscriptions
- [ ] Billing cycle radio buttons show correct monthly equivalent in real-time as user types cost
- [ ] Status badges use correct colours (active=green, trial=blue, paused=amber, cancelled=red, expired=grey)
- [ ] Seat utilisation shows as X/Y with mini progress bar
- [ ] Category list shows all categories with spend/budget
- [ ] Add/edit category works with icon and colour pickers
- [ ] Toggle category active/inactive works
- [ ] Budget progress bars show correct colours (green < 80%, yellow 80-100%, red > 100%)
- [ ] Cost history record created when subscription is added or cost changes
- [ ] Delete subscription works with confirmation dialog
- [ ] Empty states for all views
- [ ] Responsive: table scrolls horizontally on mobile

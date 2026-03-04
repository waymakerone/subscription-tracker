---
sync:
  type: doc
  layer: Subscription Tracker
build:
  status: todo
  phase: 4
  priority: P0
  depends_on: ["phase-3-cost-analysis"]
  started_at: null
  completed_at: null
---

# Phase 4: Dashboard & Polish

**Goal:** Executive dashboard with key metrics, redundancy alerts, CSV export, mobile-responsive layouts, quick actions (renew, cancel, pause), and waymaker.config.ts manifest for Host Schema.

**PRD Reference:** `docs/01-planning/product-requirements/subscription-tracker-prd.md` — Phase 4

---

## What to Build

### 1. Executive Dashboard (/dashboard)

A summary view pulling together all subscription data for leadership.

**Layout — grid of cards:**

**Row 1: Key Metrics (4 cards)**
- Total Monthly Spend (normalised amount + vs last month change percentage)
- Total Annual Spend (normalised amount)
- Upcoming Renewals (count — next 30 days, link to renewals calendar)
- Average Cost Per Subscription (monthly equivalent, with active subscription count)

**Row 2: Charts (2 cards, equal width)**
- Monthly Spend Trend (line chart, last 6 months — reuse `MonthlyCostTrend` component with `months={6}`)
- Category Breakdown (donut chart, current month — reuse `CategorySpendDonut` component)

**Row 3: Alerts & Upcoming (2 cards)**
- Redundancy Alerts: list of categories with potential redundancies (from `detectRedundancies`)
  - Each alert: category name, subscription count, potential savings
  - "Review" link to subscription list filtered by category
  - If no redundancies: "No redundancies detected" success message
- Renewal Timeline: next 5 upcoming renewals
  - Each: subscription name, renewal date (relative), cost, auto-renew badge
  - "View All" link to renewals calendar

**Row 4: Budget Health & Quick Actions (2 cards)**
- Budget Health: categories with budgets, mini progress bars
  - Show only categories near or over budget (> 60%)
  - If all healthy: "All categories within budget" success message
- Quick Actions:
  - "Add Subscription" button
  - "Export to Spreadsheet" button
  - "Export CSV" button
  - "View Renewals Calendar" button

**Dashboard implementation:**
```typescript
// src/pages/Dashboard.tsx

interface DashboardMetrics {
  totalMonthlySpend: number;
  totalAnnualSpend: number;
  upcomingRenewals: number;
  averageCostPerSubscription: number;
  activeSubscriptionCount: number;
  monthOverMonthChange: number;  // percentage
}

function calculateDashboardMetrics(subscriptions: Subscription[]): DashboardMetrics {
  const activeSubscriptions = subscriptions.filter(s => s.status === 'active');

  const totalMonthlySpend = activeSubscriptions.reduce(
    (sum, s) => sum + toMonthlyAmount(s.cost_amount, s.billing_cycle), 0
  );

  const upcomingRenewals = activeSubscriptions.filter(s => {
    const daysUntil = daysBetween(new Date(), new Date(s.next_renewal_date));
    return daysUntil >= 0 && daysUntil <= 30;
  }).length;

  return {
    totalMonthlySpend,
    totalAnnualSpend: totalMonthlySpend * 12,
    upcomingRenewals,
    averageCostPerSubscription: activeSubscriptions.length > 0
      ? totalMonthlySpend / activeSubscriptions.length
      : 0,
    activeSubscriptionCount: activeSubscriptions.length,
    monthOverMonthChange: 0,  // Calculated from cost history
  };
}
```

### 2. Redundancy Alerts (Enhanced)

Upgrade redundancy detection from Phase 3 with actionable alerts on the dashboard.

**Alert severity levels:**
- **High** — 3+ subscriptions in the same category with at least one having < 30% seat utilisation
- **Medium** — 2 subscriptions in the same category with similar names or overlapping features
- **Low** — 2+ subscriptions in the same category (informational — may be legitimate)

**Alert card design:**
```
┌─────────────────────────────────────────────┐
│ ⚠ Design Tools — 3 subscriptions            │
│                                             │
│  Figma        $15/seat  90% utilisation     │
│  Sketch       $9/seat   20% utilisation  ←  │
│  Adobe XD     $12/seat  5% utilisation   ←  │
│                                             │
│  Potential savings: $21/month               │
│  [Review Subscriptions]                     │
└─────────────────────────────────────────────┘
```

**Low utilisation indicator:**
- Arrow (←) next to subscriptions with < 50% seat utilisation
- These are candidates for consolidation or cancellation

### 3. CSV Export

Export all subscriptions or a filtered subset as a CSV file.

**File:** `src/lib/export.ts`

```typescript
export function exportSubscriptionsToCSV(
  subscriptions: Subscription[],
  categories: Category[],
  filename?: string,
) {
  const headers = [
    'Name',
    'Vendor',
    'Category',
    'Status',
    'Cost',
    'Currency',
    'Billing Cycle',
    'Monthly Equivalent',
    'Annual Equivalent',
    'Next Renewal',
    'Start Date',
    'End Date',
    'Auto Renew',
    'Owner',
    'Seats Purchased',
    'Seats Used',
    'Utilisation %',
    'Login URL',
    'Tags',
    'Notes',
  ];

  const rows = subscriptions.map(s => {
    const category = categories.find(c => c.id === s.category_id);
    const monthlyEquiv = toMonthlyAmount(s.cost_amount, s.billing_cycle);
    const annualEquiv = toAnnualAmount(s.cost_amount, s.billing_cycle);
    const utilisation = seatUtilisation(s.seats_used, s.seats_purchased);

    return [
      s.name,
      s.vendor || '',
      category?.name || 'Uncategorised',
      s.status,
      s.cost_amount,
      s.cost_currency,
      s.billing_cycle,
      monthlyEquiv.toFixed(2),
      annualEquiv.toFixed(2),
      s.next_renewal_date,
      s.start_date,
      s.end_date || '',
      s.auto_renew ? 'Yes' : 'No',
      s.owner_id || 'Unassigned',
      s.seats_purchased || '',
      s.seats_used || '',
      utilisation !== null ? `${utilisation}%` : '',
      s.login_url || '',
      s.tags?.join(', ') || '',
      s.notes || '',
    ];
  });

  const csvContent = [
    headers.join(','),
    ...rows.map(row => row.map(cell =>
      typeof cell === 'string' && cell.includes(',') ? `"${cell}"` : cell
    ).join(',')),
  ].join('\n');

  const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' });
  const url = URL.createObjectURL(blob);
  const link = document.createElement('a');
  link.href = url;
  link.download = filename || `subscriptions-${new Date().toISOString().split('T')[0]}.csv`;
  link.click();
  URL.revokeObjectURL(url);
}
```

**Export options:**
- "Export All" — all subscriptions regardless of filters
- "Export Filtered" — only subscriptions matching current filters
- "Export Active Only" — only active subscriptions

**Toast notification:** "Exported X subscriptions to CSV"

### 4. Mobile Responsive Layouts

Ensure all views work well on mobile devices.

**Subscription list (mobile):**
- Card view instead of table on screens < 768px
- Each card:
  - Header: subscription name + status badge
  - Body: vendor, cost (prominent), billing cycle, next renewal (relative time)
  - Footer: category badge, owner, seat utilisation mini bar
- Floating action button: "+" to add subscription
- Pull-to-refresh for subscription list

**Dashboard (mobile):**
- Single column layout
- Key metrics: 2x2 grid
- Charts: full-width, stacked vertically
- Alerts: collapsible accordion

**Calendar (mobile):**
- Month view: compact dots (no text on day cells)
- Tap a day → full-screen renewal list for that date
- Week view: scrollable horizontal strip

**Add subscription modal (mobile):**
- Full-screen modal
- Sections as collapsible accordions
- Large touch targets for all inputs

**Category management (mobile):**
- Card view instead of table
- Swipe to edit/deactivate

**Breakpoints:**
```css
/* Mobile first */
@media (min-width: 768px) { /* Tablet */ }
@media (min-width: 1024px) { /* Desktop */ }
@media (min-width: 1280px) { /* Wide desktop */ }
```

### 5. Quick Actions

Add quick action buttons for common subscription lifecycle operations.

**Subscription row actions:**
- **Renew** — Advances next_renewal_date by one billing cycle period
  - Monthly: +1 month
  - Quarterly: +3 months
  - Annual: +1 year
  - Creates a new calendar event for the new date
  - Shows confirmation: "Renewal date updated to [new date]"

- **Pause** — Sets status to `paused`
  - Confirmation dialog: "Pause [name]? You can resume it later."
  - Paused subscriptions excluded from spend calculations
  - Calendar event removed while paused

- **Cancel** — Sets status to `cancelled`, sets end_date to today
  - Confirmation dialog: "Cancel [name]? This will mark it as cancelled."
  - Calendar event removed
  - Cost history entry: "Cancelled"

- **Resume** (if paused) — Sets status back to `active`
  - Re-creates calendar event

**Quick action implementation:**
```typescript
// src/services/subscriptions.ts

export async function renewSubscription(subscription: Subscription) {
  const newRenewalDate = advanceDate(
    subscription.next_renewal_date,
    subscription.billing_cycle,
  );

  await updateSubscription(subscription.id, {
    next_renewal_date: newRenewalDate,
  });

  // Update or create calendar event
  await syncRenewalToCalendar({
    ...subscription,
    next_renewal_date: newRenewalDate,
  });
}

export async function pauseSubscription(id: string) {
  await updateSubscription(id, { status: 'paused' });
  // Remove calendar event while paused
}

export async function cancelSubscription(id: string) {
  await updateSubscription(id, {
    status: 'cancelled',
    end_date: new Date().toISOString().split('T')[0],
  });
  // Remove calendar event
  // Add cost history entry
}

export async function resumeSubscription(subscription: Subscription) {
  await updateSubscription(subscription.id, { status: 'active' });
  // Re-create calendar event
  await syncRenewalToCalendar(subscription);
}

function advanceDate(dateStr: string, billingCycle: string): string {
  const date = new Date(dateStr);
  switch (billingCycle) {
    case 'monthly': date.setMonth(date.getMonth() + 1); break;
    case 'quarterly': date.setMonth(date.getMonth() + 3); break;
    case 'annual': date.setFullYear(date.getFullYear() + 1); break;
  }
  return date.toISOString().split('T')[0];
}
```

### 6. waymaker.config.ts Manifest

Create the Host Schema manifest declaring what tables and services this app uses.

**File:** `waymaker.config.ts`

```typescript
import { defineConfig } from '@waymakerone/sdk';

export default defineConfig({
  name: 'Subscription Tracker',
  slug: 'subscription-tracker',
  type: 'ex',  // Internal app
  version: '1.0.0',

  tables: [
    {
      name: 'st_subscriptions',
      description: 'SaaS subscription records with cost, billing, and renewal data',
      access: 'read-write',
    },
    {
      name: 'st_categories',
      description: 'Subscription categories with optional monthly budgets',
      access: 'read-write',
    },
    {
      name: 'st_cost_history',
      description: 'Historical cost changes per subscription for trend analysis',
      access: 'read-write',
    },
  ],

  services: [
    { name: 'commander-table-operations', access: 'read-write' },
    { name: 'commander-table-ddl', access: 'write' },
    { name: 'commander-calendar', access: 'read-write' },
    { name: 'commander-sheet-operations', access: 'write' },
    { name: 'commander-goals-operations', access: 'write' },
    { name: 'journeys-send-email', access: 'write' },
    { name: 'commander-automation-operations', access: 'write' },
  ],
});
```

### 7. Final Polish

- [ ] Loading skeletons for all data-fetching views (subscription list, dashboard, calendar, analysis)
- [ ] Error boundaries with friendly error messages
- [ ] Toast notifications for all actions (added, updated, deleted, paused, cancelled, resumed, exported)
- [ ] URL-based filters (subscription list filters in query params for shareable links)
- [ ] Keyboard shortcuts: Cmd+N (new subscription), Cmd+E (export CSV)
- [ ] Confirmation dialogs for destructive actions (delete, cancel)
- [ ] Deploy-ready configuration (Vite build, environment variables)
- [ ] Verify all CRUD operations work end-to-end
- [ ] Test lifecycle: active → pause → resume → cancel
- [ ] Test calendar sync: create → update renewal → cancel (event removed)
- [ ] Test cost history: change cost → prompted for reason → history entry created

---

## Acceptance Criteria

- [ ] Dashboard loads with 4 key metric cards (monthly spend, annual spend, renewals count, avg cost)
- [ ] Month-over-month change calculated and displayed with up/down indicator
- [ ] Monthly spend trend chart shows last 6 months on dashboard
- [ ] Category breakdown donut chart shows current month on dashboard
- [ ] Redundancy alerts section shows categories with potential overlap
- [ ] Redundancy alert cards show subscription names, costs, utilisation, and potential savings
- [ ] Renewal timeline shows next 5 upcoming renewals with relative dates
- [ ] Budget health section shows categories near or over budget
- [ ] Quick actions section with 4 action buttons
- [ ] CSV export downloads correct file with all subscription data
- [ ] CSV opens correctly in Excel/Google Sheets with proper formatting
- [ ] Export filtered option respects current subscription list filters
- [ ] Mobile: subscription list shows as cards on small screens
- [ ] Mobile: dashboard single column with 2x2 metric grid
- [ ] Mobile: calendar compact view with tap-to-expand
- [ ] Mobile: full-screen add subscription modal
- [ ] Quick action: Renew advances date by correct billing cycle period
- [ ] Quick action: Pause sets status and removes calendar event
- [ ] Quick action: Cancel sets status, end_date, and removes calendar event
- [ ] Quick action: Resume restores active status and re-creates calendar event
- [ ] waymaker.config.ts declares all 3 tables and 7 services
- [ ] Loading skeletons on all data-fetching views
- [ ] Toast notifications on all user actions
- [ ] Error boundaries catch and display errors gracefully
- [ ] Deploy succeeds to Waymaker Host

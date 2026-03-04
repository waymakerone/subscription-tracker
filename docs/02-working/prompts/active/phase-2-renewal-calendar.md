---
sync:
  type: doc
  layer: Subscription Tracker
build:
  status: todo
  phase: 2
  priority: P0
  depends_on: ["phase-1-subscription-register"]
  started_at: null
  completed_at: null
---

# Phase 2: Renewal Calendar & Alerts

**Goal:** Calendar view of upcoming renewals synced to Commander Calendar, upcoming renewal panels, email reminder automation at 30/7/1 days before renewal, cancellation deadline alerts, and auto-renew indicators.

**PRD Reference:** `docs/01-planning/product-requirements/subscription-tracker-prd.md` — Phase 2

---

## What to Build

### 1. Calendar View Component

Build a calendar view showing subscription renewal dates.

**File:** `src/components/RenewalCalendar.tsx`

**Month view:**
- Full month calendar grid
- Each day cell shows coloured dots for subscriptions renewing on that date
- Dot colour matches the subscription's category colour
- Click a date → show renewal details in a side panel
- Navigate between months with arrow buttons
- "Today" button to jump to current month

**Week view:**
- 7-day strip with more detail per day
- Each renewal shown as a card: subscription name, cost, auto-renew badge
- Click a card → opens subscription detail

**View toggle:**
- Month | Week toggle at top of calendar
- Default to month view

**Implementation approach:**
```typescript
// Build a simple calendar grid — no heavy calendar library needed
interface CalendarDay {
  date: string;  // ISO date
  isCurrentMonth: boolean;
  isToday: boolean;
  renewals: Subscription[];
}

function generateCalendarDays(year: number, month: number, subscriptions: Subscription[]): CalendarDay[] {
  // Generate 6 weeks of days (42 cells)
  // Match subscriptions by next_renewal_date
  // Return array of CalendarDay objects
}
```

### 2. Renewal Detail Panel

Side panel that appears when clicking a date or subscription on the calendar.

**Panel contents:**
- Date header (e.g. "March 15, 2026")
- List of subscriptions renewing on that date, each showing:
  - Subscription name + vendor
  - Cost amount + billing cycle
  - Category badge (coloured)
  - Auto-renew indicator (green checkmark or red X)
  - Cancellation deadline (if cancellation_notice_days set: "Cancel by [date] to avoid renewal")
  - Owner name
  - Quick actions: Edit, View Details, Snooze Reminder

### 3. Upcoming Renewals Panels

**"Renewing This Week" panel:**
- Shows on `/renewals` page sidebar and dashboard
- List of subscriptions with next_renewal_date within the next 7 days
- Each item: name, cost, date, auto-renew badge
- Sorted by renewal date ascending
- Count badge in header: "Renewing This Week (3)"

**"Renewing This Month" panel:**
- Same format but for the current calendar month
- Count badge: "Renewing This Month (12)"
- Total cost of renewals this month shown at bottom

**Implementation:**
```typescript
// src/lib/renewals.ts

export interface RenewalGroup {
  label: string;
  subscriptions: Subscription[];
  totalCost: number;
}

export function getRenewalsThisWeek(subscriptions: Subscription[]): RenewalGroup {
  const today = new Date();
  const weekEnd = new Date(today);
  weekEnd.setDate(today.getDate() + 7);

  const matching = subscriptions.filter(s => {
    const renewal = new Date(s.next_renewal_date);
    return renewal >= today && renewal <= weekEnd && s.status === 'active';
  });

  return {
    label: 'Renewing This Week',
    subscriptions: matching.sort((a, b) =>
      new Date(a.next_renewal_date).getTime() - new Date(b.next_renewal_date).getTime()
    ),
    totalCost: matching.reduce((sum, s) => sum + s.cost_amount, 0),
  };
}

export function getRenewalsThisMonth(subscriptions: Subscription[]): RenewalGroup {
  const today = new Date();
  const monthEnd = new Date(today.getFullYear(), today.getMonth() + 1, 0);

  const matching = subscriptions.filter(s => {
    const renewal = new Date(s.next_renewal_date);
    return renewal >= today && renewal <= monthEnd && s.status === 'active';
  });

  return {
    label: 'Renewing This Month',
    subscriptions: matching.sort((a, b) =>
      new Date(a.next_renewal_date).getTime() - new Date(b.next_renewal_date).getTime()
    ),
    totalCost: matching.reduce((sum, s) => sum + s.cost_amount, 0),
  };
}
```

### 4. Commander Calendar Sync

Sync renewal events to Commander Calendar so they appear in the org-wide calendar.

**Create calendar event when subscription is created/updated:**
```typescript
// src/services/calendar.ts

export async function syncRenewalToCalendar(subscription: Subscription) {
  const token = await getToken();

  return fetch(`${SUPABASE_URL}/functions/v1/commander-calendar`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
    },
    body: JSON.stringify({
      action: 'create',
      data: {
        title: `${subscription.name} renewal — ${formatCurrency(subscription.cost_amount, subscription.cost_currency)}`,
        description: `${subscription.billing_cycle} subscription${subscription.auto_renew ? ' (auto-renew)' : ''}. ${subscription.vendor ? 'Vendor: ' + subscription.vendor : ''}`,
        start_date: subscription.next_renewal_date,
        end_date: subscription.next_renewal_date,
        all_day: true,
        color: '#0EA5E9',  // Sky blue — subscription tracker brand
        metadata: {
          source: 'subscription-tracker',
          subscription_id: subscription.id,
          cost_amount: subscription.cost_amount,
          billing_cycle: subscription.billing_cycle,
        },
      },
    }),
  });
}

export async function updateCalendarEvent(eventId: string, subscription: Subscription) {
  // Update existing calendar event when renewal date or cost changes
  const token = await getToken();

  return fetch(`${SUPABASE_URL}/functions/v1/commander-calendar`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
    },
    body: JSON.stringify({
      action: 'update',
      data: {
        id: eventId,
        title: `${subscription.name} renewal — ${formatCurrency(subscription.cost_amount, subscription.cost_currency)}`,
        start_date: subscription.next_renewal_date,
        end_date: subscription.next_renewal_date,
      },
    }),
  });
}

export async function deleteCalendarEvent(eventId: string) {
  const token = await getToken();

  return fetch(`${SUPABASE_URL}/functions/v1/commander-calendar`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
    },
    body: JSON.stringify({
      action: 'delete',
      data: { id: eventId },
    }),
  });
}
```

**Sync rules:**
- When a subscription is created → create a calendar event for the next_renewal_date
- When next_renewal_date changes → update the calendar event
- When a subscription is cancelled or deleted → delete the calendar event
- Store the calendar event ID on the subscription (add a `calendar_event_id` field or use metadata lookup)

### 5. Email Reminder Automation

Set up automated email reminders before subscription renewals.

**Reminder schedule:**
- 30 days before renewal → "Upcoming renewal" email
- 7 days before renewal → "Renewal next week" email
- 1 day before renewal → "Renewal tomorrow" email

**Email via Commander Journeys:**
```typescript
// src/services/reminders.ts

export async function sendRenewalReminder(
  subscription: Subscription,
  daysUntilRenewal: number,
  recipientEmail: string,
) {
  const token = await getToken();

  const subject = daysUntilRenewal === 1
    ? `${subscription.name} renews tomorrow`
    : daysUntilRenewal === 7
    ? `${subscription.name} renews next week`
    : `${subscription.name} renews in 30 days`;

  const body = `
    <h2>${subscription.name} Subscription Renewal</h2>
    <p><strong>Renewal Date:</strong> ${subscription.next_renewal_date}</p>
    <p><strong>Cost:</strong> ${formatCurrency(subscription.cost_amount, subscription.cost_currency)} (${subscription.billing_cycle})</p>
    <p><strong>Auto-renew:</strong> ${subscription.auto_renew ? 'Yes' : 'No'}</p>
    ${subscription.cancellation_notice_days
      ? `<p><strong>Cancellation deadline:</strong> ${getCancellationDeadline(subscription)}</p>`
      : ''
    }
    <p>Review this subscription in the <a href="${APP_URL}">Subscription Tracker</a>.</p>
  `;

  return fetch(`${SUPABASE_URL}/functions/v1/journeys-send-email`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
    },
    body: JSON.stringify({
      action: 'send',
      data: {
        to: recipientEmail,
        subject,
        html: body,
        metadata: {
          source: 'subscription-tracker',
          subscription_id: subscription.id,
          reminder_type: `${daysUntilRenewal}-day`,
        },
      },
    }),
  });
}
```

**Reminder check logic:**
```typescript
// Run on app load or dashboard visit
export function getSubscriptionsNeedingReminders(subscriptions: Subscription[]): {
  thirtyDay: Subscription[];
  sevenDay: Subscription[];
  oneDay: Subscription[];
} {
  const today = new Date();

  return {
    thirtyDay: subscriptions.filter(s => {
      const daysUntil = daysBetween(today, new Date(s.next_renewal_date));
      return daysUntil === 30 && s.status === 'active';
    }),
    sevenDay: subscriptions.filter(s => {
      const daysUntil = daysBetween(today, new Date(s.next_renewal_date));
      return daysUntil === 7 && s.status === 'active';
    }),
    oneDay: subscriptions.filter(s => {
      const daysUntil = daysBetween(today, new Date(s.next_renewal_date));
      return daysUntil === 1 && s.status === 'active';
    }),
  };
}
```

**Automation setup via Commander:**
```typescript
// Set up a scheduled automation to check for renewals daily
export async function setupRenewalAutomation() {
  const token = await getToken();

  return fetch(`${SUPABASE_URL}/functions/v1/commander-automation-operations`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
    },
    body: JSON.stringify({
      action: 'create',
      data: {
        name: 'Subscription Renewal Reminders',
        trigger: 'schedule',
        schedule: '0 9 * * *',  // Daily at 9 AM
        action_type: 'check_renewals',
        metadata: {
          source: 'subscription-tracker',
          reminder_days: [30, 7, 1],
        },
      },
    }),
  });
}
```

### 6. Cancellation Deadline Alerts

For subscriptions with `cancellation_notice_days` set, calculate and display the cancellation deadline.

**Cancellation deadline logic:**
```typescript
// src/lib/renewals.ts

export function getCancellationDeadline(subscription: Subscription): string | null {
  if (!subscription.cancellation_notice_days) return null;

  const renewalDate = new Date(subscription.next_renewal_date);
  const deadline = new Date(renewalDate);
  deadline.setDate(deadline.getDate() - subscription.cancellation_notice_days);

  return deadline.toISOString().split('T')[0];
}

export function isCancellationDeadlineSoon(subscription: Subscription): boolean {
  const deadline = getCancellationDeadline(subscription);
  if (!deadline) return false;

  const today = new Date();
  const deadlineDate = new Date(deadline);
  const daysUntil = daysBetween(today, deadlineDate);

  return daysUntil <= 7 && daysUntil >= 0;
}

export function isCancellationDeadlinePassed(subscription: Subscription): boolean {
  const deadline = getCancellationDeadline(subscription);
  if (!deadline) return false;

  return new Date(deadline) < new Date();
}
```

**Where cancellation alerts appear:**
1. Calendar — red warning badge on the cancellation deadline date
2. Renewal detail panel — "Cancel by [date] to avoid renewal" warning
3. Subscription list — warning icon on rows where cancellation deadline is within 7 days
4. Dashboard — dedicated "Cancellation Deadlines" section

### 7. Auto-Renew Indicator

Display auto-renew status clearly throughout the app.

**Visual indicators:**
- Calendar events: green loop icon for auto-renew, red X for manual renewal
- Subscription list: small icon in the renewal date column
- Renewal detail panel: "Auto-renew: On" with green badge or "Manual renewal required" with amber badge
- Add/edit form: toggle switch for auto_renew field

---

## Acceptance Criteria

- [ ] Calendar month view shows renewal dates with coloured dots
- [ ] Calendar week view shows renewal cards with subscription details
- [ ] Month/week toggle works correctly
- [ ] Navigate between months with arrow buttons
- [ ] Click a date shows renewal detail panel with all subscription info
- [ ] "Renewing This Week" panel shows correct subscriptions with count and total cost
- [ ] "Renewing This Month" panel shows correct subscriptions with count and total cost
- [ ] Renewal events sync to Commander Calendar on subscription create
- [ ] Calendar events update when next_renewal_date changes
- [ ] Calendar events deleted when subscription is cancelled/deleted
- [ ] Email reminder logic identifies subscriptions at 30/7/1 day thresholds
- [ ] Email content includes subscription name, cost, billing cycle, auto-renew status
- [ ] Cancellation deadline calculated correctly from next_renewal_date minus cancellation_notice_days
- [ ] Cancellation deadline warning shown when deadline is within 7 days
- [ ] Cancellation deadline passed state shown with red indicator
- [ ] Auto-renew indicator displays correctly (green loop vs red X)
- [ ] Auto-renew toggle works in add/edit subscription form
- [ ] Calendar events include auto-renew status in description
- [ ] Empty state for calendar with no renewals this month
- [ ] Loading state while fetching subscription data for calendar

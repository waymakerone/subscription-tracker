---
sync:
  type: doc
  layer: Subscription Tracker
build:
  status: todo
  phase: 3
  priority: P0
  depends_on: ["phase-2-renewal-calendar"]
  started_at: null
  completed_at: null
---

# Phase 3: Cost Analysis & Spreadsheets

**Goal:** Spend analytics with category breakdown charts, monthly cost trend lines, annual projections, cost history tracking, Commander Spreadsheet export, budget vs actual per category, redundancy detection, and goal integration.

**PRD Reference:** `docs/01-planning/product-requirements/subscription-tracker-prd.md` — Phase 3

---

## What to Build

### 1. Cost Analysis Page (/analysis)

A dedicated analytics page with charts and breakdowns.

**Layout — three sections stacked:**

**Section 1: Summary Cards (top row)**
- Total Monthly Spend (normalised — all subscriptions to monthly equivalent)
- Total Annual Spend (normalised — all subscriptions to annual equivalent)
- Average Cost Per Subscription (monthly equivalent)
- Year-over-Year Change (percentage, based on cost history — green if down, red if up)

**Section 2: Charts (two columns)**
- Left: Spend by Category donut chart
- Right: Monthly Cost Trend line chart

**Section 3: Detail Tables**
- Budget vs Actual per category
- Cost History timeline

### 2. Spend by Category Donut Chart

**File:** `src/components/charts/CategorySpendDonut.tsx`

A recharts donut chart showing current monthly spend broken down by category.

```typescript
import { PieChart, Pie, Cell, ResponsiveContainer, Tooltip, Legend } from 'recharts';

interface CategorySpend {
  categoryId: string;
  categoryName: string;
  categoryColour: string;
  totalMonthly: number;
  percentage: number;
  subscriptionCount: number;
}

function calculateCategorySpend(
  subscriptions: Subscription[],
  categories: Category[],
): CategorySpend[] {
  // Group active subscriptions by category
  // Normalise each to monthly cost
  // Calculate percentage of total
  // Sort by totalMonthly descending
}
```

**Chart features:**
- Each slice coloured by category.colour
- Hover tooltip: category name, monthly amount, percentage, subscription count
- Center text: total monthly spend
- Legend below chart: category name + amount
- Click a slice → filter the subscription list to that category (via URL param or state)

### 3. Monthly Cost Trend Line Chart

**File:** `src/components/charts/MonthlyCostTrend.tsx`

A recharts line chart showing total monthly spend over the last 12 months.

```typescript
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer } from 'recharts';

interface MonthlyTotal {
  month: string;      // "Jan 2026", "Feb 2026"
  monthKey: string;    // "2026-01", "2026-02"
  total: number;       // Total monthly-normalised spend
  breakdown: {
    categoryName: string;
    categoryColour: string;
    amount: number;
  }[];
}

function calculateMonthlyTotals(
  subscriptions: Subscription[],
  costHistory: CostHistory[],
  months: number = 12,
): MonthlyTotal[] {
  // For each of the last N months:
  // 1. Find subscriptions that were active during that month
  //    (start_date <= month_end AND (end_date >= month_start OR end_date is null) AND status != 'cancelled' before that month)
  // 2. Use cost_history to determine the cost at that point in time
  //    (most recent cost_history entry with effective_date <= month_end)
  // 3. Normalise to monthly equivalent
  // 4. Sum by category for breakdown
}
```

**Chart features:**
- X-axis: month labels (Jan, Feb, Mar...)
- Y-axis: dollar amount (formatted currency)
- Single line for total spend
- Hover tooltip: month, total amount, top 3 categories
- Optional: area fill below the line for visual weight

### 4. Annual Projection

**File:** `src/components/charts/AnnualProjection.tsx`

Project annual spend based on current active subscriptions.

```typescript
interface AnnualProjection {
  committedAnnual: number;     // Sum of all active subscriptions, annualised
  projectedAnnual: number;     // Committed + estimated new additions (optional)
  monthlyBreakdown: {
    month: string;
    committed: number;         // Known costs (active subscriptions)
    projected: number;         // Estimated based on trend
  }[];
  byCategory: {
    categoryName: string;
    categoryColour: string;
    annualAmount: number;
    percentage: number;
  }[];
}

function calculateAnnualProjection(subscriptions: Subscription[]): AnnualProjection {
  // Sum all active subscriptions, annualised via toAnnualAmount()
  // Break down by category
  // Generate 12-month forward projection
}
```

**Display:**
- Large number: "Projected Annual Spend: $148,800"
- Stacked bar chart: committed (solid) vs projected (striped/lighter) by month
- Category breakdown table below

### 5. Cost History Tracking

Track price changes over time per subscription using the `st_cost_history` table.

**File:** `src/components/CostHistoryTimeline.tsx`

**When to create cost history records:**
- When a subscription is first created → initial cost entry
- When cost_amount is updated (edit subscription) → new cost history entry with change_reason
- User can manually add historical cost entries

**Cost history display (on subscription detail page):**
- Timeline view: vertical list of cost changes
- Each entry: date, amount, change (arrow up/down with percentage), reason
- "Add Cost Change" button → modal: amount, effective_date, change_reason

**Cost change detection:**
```typescript
// When updating a subscription, detect cost change and prompt for reason
export async function updateSubscriptionWithCostTracking(
  id: string,
  updates: Partial<Subscription>,
  currentSubscription: Subscription,
) {
  // If cost_amount changed:
  if (updates.cost_amount && updates.cost_amount !== currentSubscription.cost_amount) {
    // Prompt user for change_reason (via modal)
    // Create cost_history record
    await createCostHistory({
      subscription_id: id,
      amount: updates.cost_amount,
      currency: updates.cost_currency || currentSubscription.cost_currency,
      effective_date: new Date().toISOString().split('T')[0],
      change_reason: promptedReason,  // From modal
    });
  }

  // Update subscription
  return updateSubscription(id, updates);
}
```

### 6. Commander Spreadsheet Export

Export a monthly spend summary to Commander Spreadsheets.

```typescript
// src/services/spreadsheets.ts

export async function exportToSpreadsheet(
  subscriptions: Subscription[],
  categories: Category[],
  month: string,  // "2026-03"
) {
  const token = await getToken();
  const categorySpend = calculateCategorySpend(subscriptions, categories);

  // Create a new sheet with monthly summary data
  return fetch(`${SUPABASE_URL}/functions/v1/commander-sheet-operations`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
    },
    body: JSON.stringify({
      action: 'create',
      data: {
        title: `Subscription Spend — ${month}`,
        columns: [
          { name: 'Category', type: 'text' },
          { name: 'Subscription Count', type: 'number' },
          { name: 'Monthly Spend', type: 'currency' },
          { name: 'Annual Equivalent', type: 'currency' },
          { name: 'Budget', type: 'currency' },
          { name: 'Variance', type: 'currency' },
          { name: '% of Total', type: 'percentage' },
        ],
        rows: categorySpend.map(cs => ({
          'Category': cs.categoryName,
          'Subscription Count': cs.subscriptionCount,
          'Monthly Spend': cs.totalMonthly,
          'Annual Equivalent': cs.totalMonthly * 12,
          'Budget': categories.find(c => c.id === cs.categoryId)?.budget_monthly || 0,
          'Variance': (categories.find(c => c.id === cs.categoryId)?.budget_monthly || 0) - cs.totalMonthly,
          '% of Total': cs.percentage,
        })),
      },
    }),
  });
}
```

**Export button:** "Export to Spreadsheet" on the Cost Analysis page → creates a Commander Sheet and shows a success toast with a link to the sheet.

### 7. Budget vs Actual Per Category

**File:** `src/components/BudgetVsActual.tsx`

Show budget utilisation for each category that has a budget_monthly set.

```typescript
interface BudgetStatus {
  categoryId: string;
  categoryName: string;
  categoryColour: string;
  budgetMonthly: number;
  actualMonthly: number;
  variance: number;           // budget - actual (positive = under, negative = over)
  utilisationPercent: number; // (actual / budget) * 100
  level: 'healthy' | 'warning' | 'danger' | 'exceeded';
}

function calculateBudgetStatus(
  categories: Category[],
  subscriptions: Subscription[],
): BudgetStatus[] {
  return categories
    .filter(c => c.budget_monthly !== null && c.budget_monthly > 0)
    .map(category => {
      const categorySubscriptions = subscriptions.filter(
        s => s.category_id === category.id && s.status === 'active'
      );
      const actualMonthly = categorySubscriptions.reduce(
        (sum, s) => sum + toMonthlyAmount(s.cost_amount, s.billing_cycle), 0
      );
      const utilisationPercent = (actualMonthly / category.budget_monthly!) * 100;

      return {
        categoryId: category.id,
        categoryName: category.name,
        categoryColour: category.colour || '#6B7280',
        budgetMonthly: category.budget_monthly!,
        actualMonthly,
        variance: category.budget_monthly! - actualMonthly,
        utilisationPercent,
        level: utilisationPercent > 100 ? 'exceeded'
             : utilisationPercent >= 80 ? 'danger'
             : utilisationPercent >= 60 ? 'warning'
             : 'healthy',
      };
    })
    .sort((a, b) => b.utilisationPercent - a.utilisationPercent);
}
```

**Display:**
- Card per category with budget set
- Progress bar: actual / budget, coloured by level
- Text: "$X of $Y (Z%)" — e.g. "$4,200 of $5,000 (84%)"
- Variance: "+$800 under budget" (green) or "-$300 over budget" (red)
- Sorted by utilisation descending (most at-risk first)

### 8. Redundancy Detection

Flag potential redundancies — multiple subscriptions in the same category with low seat utilisation.

**File:** `src/lib/redundancy.ts`

```typescript
interface RedundancyAlert {
  categoryId: string;
  categoryName: string;
  subscriptions: {
    id: string;
    name: string;
    costMonthly: number;
    seatUtilisation: number | null;  // percentage or null if not tracked
  }[];
  potentialSavings: number;  // Cost of lowest-utilised subscription
  reason: string;            // Human-readable explanation
}

function detectRedundancies(
  subscriptions: Subscription[],
  categories: Category[],
): RedundancyAlert[] {
  // Group active subscriptions by category
  // For categories with 2+ subscriptions:
  //   - Flag if any subscription has seat utilisation < 50%
  //   - Flag if category has 3+ subscriptions (likely overlap)
  // Calculate potential savings (cost of least-used subscription)
  // Return sorted by potentialSavings descending
}
```

**Display on Cost Analysis page:**
- "Redundancy Alerts" section
- Each alert as a card:
  - Category name
  - List of overlapping subscriptions with their costs and utilisation
  - "Potential savings: $X/month" in green
  - "Review" button → navigates to subscription list filtered to that category

### 9. Goal Integration

Link spend targets to Commander Goals.

```typescript
// src/services/goals.ts

export async function createSpendReductionGoal(
  title: string,
  targetReduction: number,
  currentSpend: number,
) {
  const token = await getToken();

  return fetch(`${SUPABASE_URL}/functions/v1/commander-goals-operations`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
    },
    body: JSON.stringify({
      action: 'create',
      data: {
        title,
        description: `Reduce monthly SaaS spend from ${formatCurrency(currentSpend)} to ${formatCurrency(currentSpend - targetReduction)}`,
        target_value: targetReduction,
        current_value: 0,
        unit: 'USD',
        metadata: {
          source: 'subscription-tracker',
          baseline_spend: currentSpend,
          target_spend: currentSpend - targetReduction,
        },
      },
    }),
  });
}
```

**"Set Spend Target" button** on the Cost Analysis page:
- Modal: "Set a spend reduction goal"
- Current monthly spend (read-only)
- Target reduction amount (number input)
- Goal title (auto-generated: "Reduce SaaS spend by X%")
- Creates a Commander Goal linked to the subscription tracker

---

## Acceptance Criteria

- [ ] Cost Analysis page loads with summary cards (total monthly, annual, avg per subscription, YoY change)
- [ ] Spend by category donut chart renders with correct category colours
- [ ] Donut chart hover shows category name, amount, percentage, subscription count
- [ ] Donut center shows total monthly spend
- [ ] Click donut slice filters subscription list to that category
- [ ] Monthly cost trend line chart shows last 12 months
- [ ] Line chart hover shows month total and top categories
- [ ] Annual projection calculates correctly from active subscriptions
- [ ] Annual projection shows committed vs projected breakdown
- [ ] Cost history record created when subscription cost changes
- [ ] Cost change prompts for change_reason via modal
- [ ] Cost history timeline displays on subscription detail page
- [ ] Manual cost history entry works via "Add Cost Change" button
- [ ] Export to Commander Spreadsheet creates a sheet with category summary
- [ ] Spreadsheet includes: category, count, monthly spend, annual, budget, variance, % of total
- [ ] Budget vs actual progress bars show correct utilisation
- [ ] Budget bars: green under 60%, yellow 60-80%, amber 80-100%, red over 100%
- [ ] Variance displayed as under/over budget with appropriate colour
- [ ] Redundancy detection flags categories with 2+ subscriptions and low utilisation
- [ ] Redundancy alert shows potential savings amount
- [ ] Goal integration creates a Commander Goal with spend reduction target
- [ ] All charts responsive on smaller screens

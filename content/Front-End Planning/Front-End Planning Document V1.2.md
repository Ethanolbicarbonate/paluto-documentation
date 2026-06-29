# Paluto Affiliate Tracking System — Front-End Planning Document

> **Version:** 1.2 | **Stack:** Next.js 14+ (App Router), TypeScript, Tailwind CSS, shadcn/ui, React Query, Zustand **Audience:** Front-end development team. This document is the authoritative implementation roadmap.
> 
> **Changelog — v1.1 Addendum:**
> 
> - **§4.2** Commission formula replaced: percentage-based → tiered (`Math.floor(revenue / 1000) * 50`). Commission field on form is now editable (manager override supported).
> - **§4.15 (new)** Field Independence Rules: `referral_success`, commission amount, and `free_meal_claimed` are fully independent. No field gates another.
> - **§4.9 + §4.16 (new)** Driver ID normalization: all reads/writes go through `.toUpperCase()`. Case-insensitive matching enforced at both the API and DB layer.
> - **§3 TransactionForm** Commission preview replaced with a live-editable pre-filled input. `commission_rate` column removed from affiliate auto-populate display.
> 
> **Changelog — v1.2 Schema Alignment:**
> 
> - **§1 File Tree** `driver_code` → `driver_id` across all component comments and type files. Added `admin/branches/new/` page for dynamic branch creation. Added `transactions/food-stamp-types/` API route.
> - **§2 Routing** `/admin/branches` updated to include Admin branch CRUD (create/edit/delete). Branch names are now free-text `varchar` — no enum constraint.
> - **§3 Components** `BranchFilter` options are now fetched dynamically from the DB, not hardcoded. `FoodStampConditional` changed from a `<Select>` with fixed enum options to a free-text `<Input>` with `<datalist>` suggestions.
> - **§4 Data Logic** All SQL queries updated: `affiliates.driver_id` replaces `driver_id`. Branch name series in charts are now dynamic. `food_stamp_type` treated as open `varchar` — no enum validation.
> - **Dropped enums:** `branch_names` (branches.name is now `varchar`) and the implicit food stamp type enum (`food_stamp_type` is now `varchar`). These columns accept any string — no DB migration required for new values.

---

## Table of Contents

1. [File Tree / Folder Hierarchy](#1-file-tree--folder-hierarchy)
2. [Layouts, Pages, and Routing Strategy](#2-layouts-pages-and-routing-strategy)
3. [Component Architecture & Detailed Breakdowns](#3-component-architecture--detailed-breakdowns)
4. [Data Calculation & API Logic Strategy](#4-data-calculation--api-logic-strategy)

---

## 1. File Tree / Folder Hierarchy

```
paluto-affiliate-tracking/
│
├── app/                                  # Next.js App Router root
│   │
│   ├── (auth)/                           # Route group — no shared layout
│   │   └── login/
│   │       └── page.tsx                  # /login — Universal login page
│   │
│   ├── (admin)/                          # Route group — AdminLayout wraps these
│   │   ├── layout.tsx                    # AdminLayout: sidebar + topbar
│   │   └── admin/
│   │       ├── dashboard/
│   │       │   └── page.tsx              # /admin/dashboard
│   │       ├── branches/
│   │       │   ├── page.tsx              # /admin/branches (list + create button)
│   │       │   ├── new/
│   │       │   │   └── page.tsx          # /admin/branches/new (create branch form)
│   │       │   └── [branchId]/
│   │       │       └── page.tsx          # /admin/branches/[id] (detail + edit)
│   │       ├── affiliates/
│   │       │   ├── page.tsx              # /admin/affiliates (all-branch table)
│   │       │   └── [affiliateId]/
│   │       │       └── page.tsx          # /admin/affiliates/[id] (read-only detail)
│   │       ├── commissions/
│   │       │   └── page.tsx              # /admin/commissions (global view)
│   │       ├── leaderboard/
│   │       │   └── page.tsx              # /admin/leaderboard (cross-branch)
│   │       ├── reports/
│   │       │   └── page.tsx              # /admin/reports (data match + revenue)
│   │       └── managers/
│   │           └── page.tsx              # /admin/managers (manage AM accounts)
│   │
│   ├── (manager)/                        # Route group — ManagerLayout wraps these
│   │   ├── layout.tsx                    # ManagerLayout: sidebar + topbar
│   │   └── manager/
│   │       ├── dashboard/
│   │       │   └── page.tsx              # /manager/dashboard
│   │       ├── transactions/
│   │       │   ├── page.tsx              # /manager/transactions (list + tabs)
│   │       │   ├── new/
│   │       │   │   └── page.tsx          # /manager/transactions/new (form)
│   │       │   └── [transactionId]/
│   │       │       └── page.tsx          # /manager/transactions/[id] (detail modal page)
│   │       ├── affiliates/
│   │       │   ├── page.tsx              # /manager/affiliates (branch list)
│   │       │   ├── new/
│   │       │   │   └── page.tsx          # /manager/affiliates/new (onboard form)
│   │       │   └── [affiliateId]/
│   │       │       └── page.tsx          # /manager/affiliates/[id] (detail/edit)
│   │       ├── commissions/
│   │       │   ├── page.tsx              # /manager/commissions (redirects to /pending)
│   │       │   ├── pending/
│   │       │   │   └── page.tsx          # /manager/commissions/pending
│   │       │   └── claimed/
│   │       │       └── page.tsx          # /manager/commissions/claimed
│   │       ├── meals/
│   │       │   └── page.tsx              # /manager/meals (banked meals dashboard)
│   │       └── leaderboard/
│   │           └── page.tsx              # /manager/leaderboard (branch leaderboard)
│   │
│   ├── (affiliate)/                      # Route group — AffiliateMobileLayout
│   │   ├── layout.tsx                    # Mobile layout: bottom nav + header
│   │   └── affiliate/
│   │       ├── dashboard/
│   │       │   └── page.tsx              # /affiliate/dashboard
│   │       ├── earnings/
│   │       │   └── page.tsx              # /affiliate/earnings (history list)
│   │       ├── meals/
│   │       │   └── page.tsx              # /affiliate/meals (my banked meals)
│   │       └── leaderboard/
│   │           └── page.tsx              # /affiliate/leaderboard (rank view)
│   │
│   ├── api/                              # API Route Handlers (backend-for-frontend)
│   │   ├── auth/
│   │   │   └── [...nextauth]/
│   │   │       └── route.ts              # NextAuth credential provider
│   │   ├── admin/
│   │   │   ├── stats/route.ts            # Global KPI aggregates
│   │   │   ├── branches/route.ts         # Branch CRUD
│   │   │   ├── branches/[id]/route.ts
│   │   │   ├── affiliates/route.ts       # All-branch affiliate queries
│   │   │   ├── affiliates/[id]/route.ts
│   │   │   ├── commissions/route.ts      # Global commission data
│   │   │   ├── leaderboard/route.ts      # Cross-branch leaderboard
│   │   │   ├── reports/route.ts          # Data match + revenue report
│   │   │   └── managers/route.ts         # Manager account CRUD
│   │   ├── manager/
│   │   │   ├── stats/route.ts            # Branch KPI aggregates
│   │   │   ├── transactions/route.ts     # Transaction CRUD
│   │   │   ├── transactions/[id]/route.ts
│   │   │   ├── transactions/food-stamp-types/route.ts  # GET distinct food_stamp_type values for datalist
│   │   │   ├── affiliates/route.ts       # Branch affiliate queries
│   │   │   ├── affiliates/[id]/route.ts
│   │   │   ├── affiliates/search/route.ts# Combobox search endpoint
│   │   │   ├── commissions/route.ts
│   │   │   ├── commissions/[id]/claim/route.ts  # Mark commission claimed
│   │   │   ├── meals/route.ts            # Banked meals list
│   │   │   └── meals/[txId]/claim/route.ts      # Mark meal claimed
│   │   └── affiliate/
│   │       ├── me/route.ts               # Current affiliate profile
│   │       ├── earnings/route.ts         # Personal commission history
│   │       ├── meals/route.ts            # Personal meal history
│   │       └── leaderboard/route.ts      # Rank in own branch
│   │
│   ├── layout.tsx                        # Root layout (fonts, providers)
│   └── page.tsx                          # / → redirects based on session role
│
├── components/
│   ├── ui/                               # Primitive / atomic components (shadcn/ui base)
│   │   ├── Button.tsx
│   │   ├── Input.tsx
│   │   ├── Select.tsx
│   │   ├── Badge.tsx
│   │   ├── Modal.tsx                     # Dialog wrapper with portal
│   │   ├── Drawer.tsx                    # Side-sheet for detail views
│   │   ├── Table.tsx                     # Reusable table with sort headers
│   │   ├── Card.tsx
│   │   ├── Avatar.tsx                    # Initials-based avatar
│   │   ├── Skeleton.tsx                  # Loading placeholders
│   │   ├── Toast.tsx                     # Notification system
│   │   ├── Tooltip.tsx
│   │   ├── Toggle.tsx                    # Boolean switch (for free_meal, food_stamp)
│   │   ├── Tabs.tsx                      # Tab navigation (Pending/Claimed)
│   │   └── DateRangePicker.tsx           # Calendar-based custom range
│   │
│   ├── layout/
│   │   ├── AdminSidebar.tsx              # Desktop left nav for Admin role
│   │   ├── ManagerSidebar.tsx            # Desktop left nav for Manager role
│   │   ├── AffiliateMobileNav.tsx        # Bottom tab bar for Affiliate role
│   │   ├── TopBar.tsx                    # Universal top bar (user info, logout)
│   │   └── PageHeader.tsx                # Title + breadcrumb + action slot
│   │
│   ├── charts/                           # Chart components (Recharts)
│   │   ├── RevenueLineChart.tsx          # Revenue trend over time
│   │   ├── CommissionBarChart.tsx        # Commission earned per affiliate/group
│   │   ├── BranchComparisonChart.tsx     # Side-by-side branch bars
│   │   ├── AffiliationGroupPieChart.tsx  # Headcount share by affiliation_group
│   │   └── HeadcountTrendChart.tsx       # Headcount over DWM period
│   │
│   ├── shared/                           # Domain-agnostic reusable pieces
│   │   ├── StatCard.tsx                  # KPI number card (icon + label + value)
│   │   ├── DWMFilter.tsx                 # Daily/Weekly/Monthly toggle + custom range
│   │   ├── SearchBar.tsx                 # Debounced search input
│   │   ├── BranchFilter.tsx              # Dropdown: All | [branches fetched from API — dynamic]
│   │   ├── AffiliateTypeFilter.tsx       # Driver | Tour Guide | All
│   │   ├── EmptyState.tsx                # Illustration + message for empty data
│   │   ├── LoadingSpinner.tsx
│   │   └── RoleBadge.tsx                 # Colored badge for affiliate_type
│   │
│   ├── admin/
│   │   ├── dashboard/
│   │   │   ├── GlobalStatsRow.tsx        # Top-level KPI cards
│   │   │   ├── BranchComparisonTable.tsx # Side-by-side branch metrics table
│   │   │   └── TopPerformersPreview.tsx  # Top 5 affiliates cross-branch
│   │   ├── branches/
│   │   │   └── BranchCard.tsx            # Card with branch quick-stats
│   │   ├── affiliates/
│   │   │   └── GlobalAffiliateTable.tsx  # All affiliates with branch column
│   │   ├── commissions/
│   │   │   └── GlobalCommissionTable.tsx # All commissions with branch column
│   │   ├── reports/
│   │   │   ├── DataMatchReport.tsx       # Headcount + revenue by affiliation_group
│   │   │   └── ExportButton.tsx          # CSV/PDF export
│   │   └── managers/
│   │       └── ManagerTable.tsx          # CRUD table for affiliate_managers
│   │
│   ├── manager/
│   │   ├── dashboard/
│   │   │   ├── BranchStatsRow.tsx        # Branch-scoped KPI cards
│   │   │   ├── RecentTransactionsFeed.tsx# Last 10 transactions as list
│   │   │   └── BankedMealsAlert.tsx      # Banner/card alerting unclaimed meals
│   │   ├── transactions/
│   │   │   ├── TransactionTable.tsx      # Paginated sortable transaction list
│   │   │   ├── TransactionForm.tsx       # Full create/edit form
│   │   │   ├── AffiliateSearchCombobox.tsx # Live-search affiliate by name/code
│   │   │   ├── FoodStampConditional.tsx  # Conditional food_stamp_type field
│   │   │   └── TransactionDetailModal.tsx# Read-only detail view in modal
│   │   ├── affiliates/
│   │   │   ├── AffiliateTable.tsx        # Branch affiliate list table
│   │   │   ├── AffiliateForm.tsx         # Onboard / edit affiliate form
│   │   │   └── AffiliateDetailDrawer.tsx # Side drawer with full affiliate stats
│   │   ├── commissions/
│   │   │   ├── CommissionTable.tsx       # Commission rows with status badge
│   │   │   └── BulkClaimModal.tsx        # Confirm-and-mark-claimed modal
│   │   ├── meals/
│   │   │   ├── BankedMealsList.tsx       # List of affiliates with banked meals
│   │   │   └── ClaimMealModal.tsx        # Select which trip meals to mark claimed
│   │   └── leaderboard/
│   │       └── LeaderboardTable.tsx      # Ranked table with medals
│   │
│   └── affiliate/
│       ├── dashboard/
│       │   ├── EarningsSummaryCard.tsx   # Total/Pending/Claimed in mobile cards
│       │   ├── MealStatusCard.tsx        # Banked meals count prominently
│       │   └── RecentActivityFeed.tsx    # Last 5 trips as a card feed
│       ├── earnings/
│       │   └── EarningsHistoryList.tsx   # Scrollable list of commission entries
│       └── leaderboard/
│           └── RankCard.tsx              # Current rank, points, visual bar
│
├── hooks/
│   ├── useAuth.ts                        # Session state + role extraction
│   ├── useRole.ts                        # Role-guard helper hook
│   ├── useDWMFilter.ts                   # Encapsulates DWM date range state
│   ├── useTransactionForm.ts             # All form state + validation logic
│   ├── useAffiliateSearch.ts             # Debounced affiliate lookup
│   └── useDebounce.ts                    # Generic debounce utility hook
│
├── lib/
│   ├── auth.ts                           # NextAuth config, session callbacks
│   ├── db.ts                             # Database client (e.g., Drizzle ORM / Prisma)
│   ├── utils.ts                          # cn(), clamp(), generateCode()
│   ├── formatters.ts                     # Currency, date, percentage formatters
│   └── validators.ts                     # Zod schemas for all forms
│
├── store/
│   ├── authStore.ts                      # Zustand: session, role, branch context
│   └── filterStore.ts                    # Zustand: global DWM filter state
│
├── types/
│   ├── auth.types.ts                     # User, Session, Role enums
│   ├── affiliate.types.ts                # Affiliate, AffiliateType, AffiliationGroup
│   ├── transaction.types.ts              # Transaction, food_stamp_type is plain string (varchar)
│   ├── commission.types.ts               # CommissionWallet, CommissionStatus
│   └── branch.types.ts                   # Branch, BranchName
│
├── middleware.ts                          # Route protection by role (NextAuth middleware)
├── next.config.ts
├── tailwind.config.ts
└── tsconfig.json
```

---

## 2. Layouts, Pages, and Routing Strategy

### 2.1 Middleware & Route Guard Strategy

`middleware.ts` intercepts every request. It reads the NextAuth JWT session token and enforces access rules before the page ever renders:

|Role|Allowed Path Prefixes|Redirect If Not Authorized|
|---|---|---|
|`admin`|`/admin/*`|`/login`|
|`affiliate_manager`|`/manager/*`|`/login`|
|`affiliate`|`/affiliate/*`|`/login`|
|Unauthenticated|`/login`|—|

The root page (`/`) reads the session and performs a **server-side redirect** to the role's default dashboard immediately.

### 2.2 Auth Routes

|Route|Component|Access|Description|
|---|---|---|---|
|`/login`|`(auth)/login/page.tsx`|Public|Single login form for all roles. After authentication, NextAuth session contains `role`, `userId`, and `branchId`. Server action redirects to the correct dashboard.|

### 2.3 Admin Routes (`/admin/*`)

**Layout:** `AdminLayout` — Fixed left sidebar (240px), sticky top bar, scrollable main content area. Desktop-only (`min-width: 1024px` warning shown on small screens).

|Route|Page File|Access|Description|
|---|---|---|---|
|`/admin/dashboard`|`admin/dashboard/page.tsx`|Admin only|HQ overview. Global KPIs, cross-branch comparison, top performers.|
|`/admin/branches`|`admin/branches/page.tsx`|Admin only|List of all branches as cards with quick metrics. Branch names are dynamic `varchar` — Admins can create new branches from this page via the "+ Create Branch" button.|
|`/admin/branches/new`|`admin/branches/new/page.tsx`|Admin only|Branch creation form. Fields: Name (varchar, required), Location (varchar). On submit: `POST /api/admin/branches`. New branch is immediately available system-wide in all dropdowns and filters.|
|`/admin/branches/[branchId]`|`admin/branches/[branchId]/page.tsx`|Admin only|Drill-down into a single branch: its transactions, affiliates, and commission summary. Includes Edit and (soft-)Delete actions.|
|`/admin/affiliates`|`admin/affiliates/page.tsx`|Admin only|Full affiliate roster across all branches. Filterable by branch, type, affiliation group.|
|`/admin/affiliates/[affiliateId]`|`admin/affiliates/[affiliateId]/page.tsx`|Admin only|Read-only affiliate profile: full history, total earnings, meal history.|
|`/admin/commissions`|`admin/commissions/page.tsx`|Admin only|Global commission ledger. All Pending and Claimed entries across all branches.|
|`/admin/leaderboard`|`admin/leaderboard/page.tsx`|Admin only|Cross-branch leaderboard. Top earners by total commission. Filterable by DWM and branch.|
|`/admin/reports`|`admin/reports/page.tsx`|Admin only|Data Match Report (affiliation group breakdown) + Revenue trend charts. Export to CSV/PDF.|
|`/admin/managers`|`admin/managers/page.tsx`|Admin only|Manage Affiliate Manager accounts: view, create, deactivate. Assign to branch.|

### 2.4 Manager Routes (`/manager/*`)

**Layout:** `ManagerLayout` — Similar to Admin but sidebar nav links are branch-scoped. Branch name displayed prominently in sidebar header. Desktop-optimized.

|Route|Page File|Access|Description|
|---|---|---|---|
|`/manager/dashboard`|`manager/dashboard/page.tsx`|Manager (own branch)|Branch KPIs, recent transactions feed, banked meals alert.|
|`/manager/transactions`|`manager/transactions/page.tsx`|Manager (own branch)|Full transaction history for branch. Searchable, sortable, filterable by DWM.|
|`/manager/transactions/new`|`manager/transactions/new/page.tsx`|Manager (own branch)|Transaction creation form. The most critical form in the application.|
|`/manager/transactions/[id]`|`manager/transactions/[id]/page.tsx`|Manager (own branch)|Read-only detail view of a single transaction with commission breakdown.|
|`/manager/affiliates`|`manager/affiliates/page.tsx`|Manager (own branch)|All affiliates onboarded at this branch. Search, filter by type/group.|
|`/manager/affiliates/new`|`manager/affiliates/new/page.tsx`|Manager (own branch)|Affiliate onboarding form.|
|`/manager/affiliates/[id]`|`manager/affiliates/[id]/page.tsx`|Manager (own branch)|Affiliate detail drawer/page: stats, transaction history, banked meals, commission history.|
|`/manager/commissions/pending`|`manager/commissions/pending/page.tsx`|Manager (own branch)|List of all Pending commission entries. Manager can mark as Claimed here.|
|`/manager/commissions/claimed`|`manager/commissions/claimed/page.tsx`|Manager (own branch)|Historical log of all Claimed commissions with payout dates.|
|`/manager/meals`|`manager/meals/page.tsx`|Manager (own branch)|**Banked Free Meals Hub.** Shows all affiliates with unclaimed meals. Manager claims meals here.|
|`/manager/leaderboard`|`manager/leaderboard/page.tsx`|Manager (own branch)|Branch-scoped leaderboard. Ranked by total commission earnings.|

**Nested Tab Navigation — Commissions:** The `/manager/commissions` page renders a `<Tabs>` component at the top. The two tabs link to `/manager/commissions/pending` and `/manager/commissions/claimed`. The active tab is derived from the current URL pathname.

### 2.5 Affiliate Routes (`/affiliate/*`)

**Layout:** `AffiliateMobileLayout` — Full-height mobile viewport. Fixed bottom navigation bar (4 icons: Dashboard, Earnings, Meals, Leaderboard). Fixed top header with greeting and driver ID. No sidebar. Optimized for 375px–430px viewports.

|Route|Page File|Access|Description|
|---|---|---|---|
|`/affiliate/dashboard`|`affiliate/dashboard/page.tsx`|Own affiliate only|Personal overview. Total pending, total claimed, banked meals count, last trip date.|
|`/affiliate/earnings`|`affiliate/earnings/page.tsx`|Own affiliate only|Full chronological commission history. Filterable by status.|
|`/affiliate/meals`|`affiliate/meals/page.tsx`|Own affiliate only|Shows all trips where `free_meal_claimed = false` (their banked meals). Read-only.|
|`/affiliate/leaderboard`|`affiliate/leaderboard/page.tsx`|Own affiliate only|Shows full leaderboard for their branch. Highlights the affiliate's own row.|

---

## 3. Component Architecture & Detailed Breakdowns

### 3.1 `/login` — Universal Login Page

**Visual:** Centered card on a branded background (restaurant/food-themed). Card is ~400px wide. Contains logo at top.

**UI Elements:**

- Logo / app name `"Paluto"` at top of card
- `<Input>` — Username
- `<Input type="password">` — Password
- `<Button type="submit">` — "Sign In"
- Error message area (inline, below button)

**Functionality:**

1. On submit, call NextAuth `signIn('credentials', { username, password })`.
2. On success, NextAuth session is populated with `{ role, userId, branchId, fullName }`.
3. Server-side `redirect()` to:
    - `role === 'admin'` → `/admin/dashboard`
    - `role === 'affiliate_manager'` → `/manager/dashboard`
    - `role === 'affiliate'` → `/affiliate/dashboard`
4. On error, display: `"Invalid username or password."` inline — **never** disclose which field is wrong.

---

### 3.2 Admin Pages

---

#### `/admin/dashboard`

**Visual:** Full-width desktop page. Row of stat cards at top. Two-column section below: left = branch comparison table, right = top performers list. Bottom = revenue line chart spanning full width.

**Components:**

**`GlobalStatsRow`**

- 4 x `<StatCard>` in a responsive 4-column grid.
- Cards: `Total Revenue (All Branches)`, `Total Commissions Paid Out`, `Total Pending Commissions`, `Total Active Affiliates`.
- Each card: colored icon (left), large number (center), label (bottom), subtle percentage change from previous period (below number, green/red).
- Includes `<DWMFilter>` above the row — changing the filter re-fetches all cards via React Query.

**`DWMFilter` (shared component)**

- A segmented button group: `[Day] [Week] [Month] [Custom]`
- Selecting `Custom` reveals a `<DateRangePicker>` with Start Date and End Date inputs.
- State managed by `useDWMFilter` hook. The computed `{ startDate, endDate }` object is passed to all data-fetching hooks on the page.

**`BranchComparisonTable`**

- Standard HTML table. Rows are generated dynamically from the `branches` table — no fixed row count is assumed. A new branch created by an Admin will automatically appear here.
- Columns: `Branch | Total Trips | Total Headcount | Total Revenue | Total Commissions | Pending | Claimed | Active Affiliates`.
- Each branch row is clickable → navigates to `/admin/branches/[branchId]`.
- Last row is a bold `"Totals"` summary row.

**`TopPerformersPreview`**

- A vertical list of 5 items. Each item: `Rank Medal Icon | Avatar (initials) | Name + Driver ID | Branch Badge (dynamic branch name from DB) | Total Earnings`.
- Footer: `"View Full Leaderboard →"` link to `/admin/leaderboard`.

**`RevenueLineChart`**

- Recharts `<LineChart>` with one line per branch, each a different color. Lines are rendered dynamically based on the branch list returned by the API — no branch is hardcoded.
- X-axis: date buckets based on DWM filter (daily = hours, weekly = days, monthly = weeks).
- Y-axis: revenue in Philippine Peso (₱).
- Tooltip on hover shows all branch values.
- Legend below chart (branch names pulled from API response, not hardcoded).

---

#### `/admin/branches/[branchId]`

**Visual:** Page header with branch name + location. Row of branch-specific stat cards. Below: two tabs — `Transactions` and `Affiliates`.

**Components:**

- `PageHeader`: Branch name (e.g., "Bitoon Branch" — displayed verbatim from `branches.name` varchar, whatever the Admin named it), location string, `<DWMFilter>`.
- 4 x `<StatCard>`: Revenue, Trips, Active Affiliates, Pending Commissions — all scoped to this branch.
- `<Tabs>`: `Transactions | Affiliates`
    - Transactions tab: `<TransactionTable>` (read-only, same as manager view but no action buttons).
    - Affiliates tab: `<GlobalAffiliateTable>` filtered to this branch.

---

#### `/admin/branches` — Branch List & Creation

**Visual:** Page header with `"Branches"` title + `"+ Create Branch"` button in the action slot. Below: a grid of `<BranchCard>` components, one per branch, loaded from `GET /api/admin/branches`.

**`BranchCard`** — Each card shows:

- Branch name (varchar, whatever the Admin entered)
- Location string
- Quick stats: Total Affiliates | Total Revenue (current month) | Active Today
- `[View Details]` → navigates to `/admin/branches/[branchId]`
- `[Edit]` → inline name/location edit via a popover form (PATCH)
- `[Delete]` → only enabled if the branch has 0 affiliates and 0 transactions (to prevent orphaned records). Shows a confirmation modal.

**`"+ Create Branch"` button flow:**

1. Navigates to `/admin/branches/new`.
2. Form fields: `Name` (varchar, required, max 100 chars) and `Location` (varchar, optional).
3. On submit: `POST /api/admin/branches` with `{ name, location }`.
4. On success: redirect to `/admin/branches/[newId]`. The new branch is **immediately available** in every `<BranchFilter>` dropdown system-wide because all filters call `GET /api/admin/branches` at render time — no cache invalidation needed beyond React Query's default stale-time.

> **Design note:** Because `branches.name` is a plain `varchar`, the Admin can name branches anything: `"Bitoon"`, `"Passi HQ"`, `"Cebu Pilot"`, etc. There is no enum constraint. The `<BranchFilter>` component must never hardcode branch names — always fetch from the API.

---

#### `/admin/affiliates`

**Visual:** Full-width table page with filter bar at top.

**Filter Bar Components:**

- `<SearchBar>` — search by name or driver ID
- `<BranchFilter>` — dropdown populated dynamically from `GET /api/admin/branches` (returns `id` + `name` for each branch). Options: `All | [branch name 1] | [branch name 2] | ...`. Adding a new branch via `/admin/branches/new` immediately appears here without any code change.
- `<AffiliateTypeFilter>` — All | Driver | Tour Guide
- Affiliation Group Filter — dropdown: All | TAXI | GRAB | MAXIM | VAN | BTODA | (others from DB)

**`GlobalAffiliateTable`**

- Sortable columns: `Name | Driver ID | Type | Affiliation Group | Branch | Manager | Total Earned | Status`.
- Note: `commission_rate` is intentionally omitted from this table — the global tier formula is used, and the DB field is a legacy placeholder only.
- Each row is clickable → `/admin/affiliates/[id]`.
- `<RoleBadge>` component in the Type column (Driver = blue, Tour Guide = orange).
- Pagination: 25 rows per page, with page controls at bottom.

---

#### `/admin/leaderboard`

**Visual:** Podium display for top 3, then a ranked table below for ranks 4–N.

**Filter Bar:**

- `<DWMFilter>` — controls the earnings calculation period.
- `<BranchFilter>` — filter to see one branch or all.
- `<AffiliateTypeFilter>` — filter drivers only or guides only.

**Podium (Top 3):**

- Three cards rendered in 2-1-3 visual order (1st in center, elevated). Each shows: Rank Medal, Avatar, Name, Driver ID, Total Earned, Trip Count.

**`LeaderboardTable` (Ranks 4+):**

- Columns: `Rank | Name | Code | Branch | Group | Trips | Total Headcount | Total Earned`.
- Current user's row highlighted (not applicable for admin, but component supports it for reuse).
- Paginated: 20 rows per page.

---

#### `/admin/reports`

**Visual:** Two major report sections on one page, each with its own DWM filter. Export button at top-right of each section.

**`DataMatchReport`**

- Table: Rows = Affiliation Groups (TAXI, GRAB, MAXIM, VAN, BTODA, etc.). Columns = `Group | # Affiliates | Total Trips | Total Headcount | Total Revenue | Total Commission | Avg Revenue/Trip`.
- `<BranchFilter>` to scope or aggregate across branches.
- Footer row: Totals.
- `<ExportButton>` — generates CSV client-side using `Papa.unparse()` or triggers a `/api/admin/reports?format=csv` download.

**Revenue Trend Section**

- `<RevenueLineChart>` (reused component).
- `<HeadcountTrendChart>` (bar chart).

---

### 3.3 Manager Pages

---

#### `/manager/dashboard`

**Visual:** Two-column layout. Left column (wider): stat cards + recent transactions feed. Right column (narrower): banked meals alert + quick-action buttons.

**`BranchStatsRow`**

- 4 x `<StatCard>` in a 4-column grid.
- Cards: `Today's Trips`, `Today's Revenue`, `Pending Commissions (Branch)`, `Banked Meals (Branch)`.
- `<DWMFilter>` above row (Today's Trip and Revenue cards respond to this; Pending and Banked are always current totals).

**`RecentTransactionsFeed`**

- A scrollable list of the last 10 transactions.
- Each item: `[Avatar] [Affiliate Name · Code] [₱ Revenue] [Date] [Status Badge]`.
- Status badge: green `"Claimed"`, yellow `"Pending"`.
- "View All Transactions →" link at bottom.

**`BankedMealsAlert`**

- Attention-grabbing card (amber/orange border).
- Headline: `"🍴 X Affiliates have unclaimed banked meals"`
- Sub-text: "Don't forget to log meals when drivers return."
- `<Button>` → navigates to `/manager/meals`.
- If count is 0, the card is not shown (or shown in a muted "all clear" state).

**Quick Actions (right column):**

- `<Button>` — "+ Log New Transaction" → `/manager/transactions/new`
- `<Button>` — "Onboard New Affiliate" → `/manager/affiliates/new`
- `<Button>` — "Manage Meals" → `/manager/meals`

---

#### `/manager/transactions/new` — Transaction Creation Form

This is the **most critical and complex page** in the application. It must be bulletproof.

**Visual:** Single-column centered form (~640px max-width) with a `PageHeader` ("Log New Transaction"). Form divided into labeled sections.

**`TransactionForm` — Full Field Breakdown:**

**Section 1: Affiliate**

- **`AffiliateSearchCombobox`** — A combobox input (type-to-search). User types a name or driver ID (e.g., "D-0055"). Calls `GET /api/manager/affiliates/search?q=` (debounced 300ms). Dropdown shows matching results: `[Driver ID] — [Full Name] — [Group Badge]`. On selection:
    - Locks the field to show selected affiliate.
    - Auto-populates a read-only display below: `Contact Number: [phone]`, `Affiliation Group: [group]`.
    - **Note:** Commission Rate is no longer a per-affiliate field driving the calculation. The global tier formula (`₱50 per ₱1,000 revenue`) applies automatically — see the Commission field in Section 2 below.
    - A small `[✕ Clear]` button allows re-selection.

**Section 2: Trip Details**

- **Headcount** — `<Input type="number">` (min: 1). Label: "Number of Customers". Required.
- **Total Revenue (Trip Amount)** — `<Input type="number">` with `₱` prefix. Required.
- **Commission Amount** — `<Input type="number">` with `₱` prefix. **This field is pre-filled automatically using the tier formula as the manager types revenue, but remains fully editable so the manager can manually override the final value before submitting.**
    - Auto-fill behavior: `onChange` on the Revenue field → `Math.floor(revenue / 1000) * 50` → written into the Commission Amount field via `setValue('commissionAmount', calculated)` (React Hook Form).
    - The field is NOT read-only. The manager can click into it and change the number at any time.
    - Visual treatment: a small `"Auto-calculated"` ghost label or pill sits to the right of the input when the value matches the formula result. If the manager edits it manually, the pill changes to `"Manually overridden"` (amber color) as a soft warning — not blocking.
    - Help text below field: `"Default: ₱50 per ₱1,000 of revenue. Edit to override."`.
    - The value submitted to the API is whatever is in this field at time of submit — the server does NOT recalculate it.
- **Transaction Date** — `<DateTimePicker>`. Defaults to `now()`. Manager can backdate if needed (within 7-day limit, enforced by validator).
- **Referral Success** — `<Toggle>`. Default: ON (true). Label: "Customer was referred by this affiliate." **This toggle is informational only — it does NOT affect whether a commission or free meal is granted. All three fields (referral_success, commission, free_meal_claimed) are fully independent.**
- **Notes** — `<Textarea>`. Optional. Placeholder: `"e.g., First Referral, customer party of 8"`.

**Section 3: Free Meal**

- **Free Meal Claimed (This Visit)** — `<Toggle>`. Default: OFF (false). Label: "Did the driver eat their free meal today?"
    - If toggled ON: a subtle green confirmation indicator appears. No sub-fields needed.
    - If left OFF: meal is automatically "banked" (will appear in unclaimed count).

**Section 4: Food Stamp (Conditional)**

- **Was a Food Stamp/Voucher Given?** — `<Toggle>`. Default: OFF (false).
    - **If toggled ON:** A conditional field `<FoodStampConditional>` slides in below (CSS `transition-all` for smooth reveal):
        - **Food Stamp Type** — Free-text `<Input>` with an HTML `<datalist>` for suggestions. Because `food_stamp_type` is a plain `varchar` column (no enum), managers can type any value freely. The `<datalist>` is populated by `GET /api/manager/transactions/food-stamp-types` which returns previously used type strings for that branch, enabling consistency without restricting future types.
        - **API for suggestions:** `SELECT DISTINCT food_stamp_type FROM transactions WHERE branch_id = :branchId AND food_stamp_type IS NOT NULL ORDER BY food_stamp_type ASC`. Results are cached by React Query — no extra DB hit per keystroke.
        - Example values that may appear as suggestions: `"Standard"`, `"Unli Paluto & Sugba Converted"`. These are not hardcoded in the UI — they are read from whatever values already exist in the DB.
        - This field is required (non-empty string) if `food_stamp = true`. Validated on submit via Zod: `z.string().min(1, 'Food stamp type is required').max(100)`.

**Form Actions:**

- `<Button type="submit" variant="primary">` — "Log Transaction"
- `<Button type="button" variant="ghost">` — "Cancel" → navigates back

**Submission Flow:**

1. Client-side validation via Zod schema (all required fields).
2. `POST /api/manager/transactions` with body: `{ affiliateId, branchId, headcount, totalRevenue, commissionAmount, transactionDate, referralSuccess, freeMealClaimed, foodStamp, foodStampType, notes }`.
    - `commissionAmount` is whatever value the manager left in the Commission Amount field — auto-calculated or manually overridden. The API uses this value directly and does **not** recalculate it.
3. API creates the `transactions` row, then immediately creates the `commission_wallet` row using `commissionAmount` from the request body.
4. On success: Toast notification "Transaction logged successfully!", redirect to `/manager/transactions`.
5. On error: Inline error messages under each invalid field.

---

#### `/manager/transactions` — Transaction List

**Visual:** Full-width table with filter bar at top. Prominent "+ Log Transaction" button in PageHeader action slot.

**Filter Bar:**

- `<SearchBar>` — search by affiliate name or driver ID.
- `<DWMFilter>` — filter by date range.
- `<AffiliateTypeFilter>` — All | Driver | Tour Guide.
- Status Filter (quick toggle): `All | Has Banked Meal | Has Food Stamp`.

**`TransactionTable`**

- Sortable columns: `Date | Affiliate (Code) | Headcount | Revenue | Commission | Free Meal | Food Stamp | Notes | Actions`.
- **Free Meal column:** Green checkmark if `free_meal_claimed = true`. Orange icon `"Banked"` if false.
- **Food Stamp column:** Badge `"[Type]"` if `food_stamp = true`, dash otherwise.
- **Actions column:** `[View]` button → opens `TransactionDetailModal`.
- Pagination: 25 rows per page.
- **Row click behavior:** Opens `TransactionDetailModal`.

**`TransactionDetailModal`**

- A centered `<Modal>` with all transaction fields displayed read-only.
- Shows: Affiliate info (name, code, type, group), all transaction fields, commission amount, commission status.
- Action button at bottom: If commission status is Pending → `"Mark Commission Claimed"` button (calls `/api/manager/commissions/[id]/claim`). If Claimed → shows payout date.
- Also shows: Free Meal status. If banked → `"Mark Meal Claimed"` button.

---

#### `/manager/affiliates` — Affiliate List

**Visual:** Table with filter bar and `"Onboard New Affiliate"` button.

**`AffiliateTable`** — Columns: `Driver ID | Name | Type | Group | Contact | Total Earned | Banked Meals | Actions`.

- **Banked Meals column:** Number badge (orange if > 0, gray if 0).
- **Actions:** `[View]` → opens `AffiliateDetailDrawer`.
- Clicking anywhere on row also opens drawer.

**`AffiliateDetailDrawer`** — Slides in from the right (480px wide).

- **Header:** Avatar (initials), full name, driver ID, `<RoleBadge>`, affiliation group, contact number.
- **Stats Row:** Total Trips | Total Revenue Generated | Total Earned | Pending Commission | Banked Meals.
- **Tabs inside Drawer:** `Transactions | Commissions | Meals`
    - `Transactions` tab: Compact list of last 20 transactions.
    - `Commissions` tab: Commission wallet entries with status badges.
    - `Meals` tab: List of trips where meal is banked (unclaimed). Each item has a `"Mark Claimed"` button.
- **Footer Actions:** `"Edit Profile"` → `/manager/affiliates/[id]`, `"View Full Page"` link.

---

#### `/manager/affiliates/new` — Onboard Affiliate Form

**`AffiliateForm`** — Fields:

- First Name, Last Name (text inputs, required).
- Contact Number (tel input).
- Type — `<Select>`: `Driver | Tour Guide` (required).
- Affiliation Group — `<Select>` + allow custom text: `TAXI | GRAB | MAXIM | VAN | BTODA | [Other...]`. If "Other" selected: free-text input appears.
- Driver ID — Auto-suggested by system (`D-XXXX` for drivers, `G-XXXX` for guides, sequentially). Manager can override. Shown as editable input with generated suggestion. **Value is always saved as uppercase** (see §4.16).
- Notes (optional textarea).

> **Note on `commission_rate` DB field:** The `commission_rate` column exists on the `affiliates` table in the schema but is **not used for commission auto-calculation**, which now follows the global tier formula (`₱50 per ₱1,000`). The field is retained in the DB for potential future per-affiliate overrides but should **not** appear on the onboarding form to avoid confusion. Do not expose it as a user-facing input.

---

#### `/manager/commissions/pending` — Pending Commissions

**Visual:** Table with filter bar. Prominent total at top: `"₱ XX,XXX.XX in Pending Commissions"` (big bold number).

**Filter Bar:** `<SearchBar>` (affiliate name/code), `<DWMFilter>`.

**`CommissionTable`** — Columns: `Affiliate | Driver ID | Trip Date | Revenue | Commission | Status | Actions`.

- Status column: `<Badge variant="warning">Pending</Badge>`.
- Actions: `[Mark Claimed]` button per row.

**`[Mark Claimed]` Flow (Single Row):**

1. Button click → opens a small inline confirmation tooltip/popover: `"Mark this ₱XXX.XX commission as claimed? This sets payout_date to now."`.
2. `[Confirm]` → `PATCH /api/manager/commissions/[walletId]/claim` → sets `status = 'Claimed'`, `payout_date = now()`.
3. Row visually transitions (fades out + moves to Claimed tab).

**Bulk Actions:**

- Checkbox column on each row.
- "Select All" checkbox in header.
- When ≥1 checked: sticky bottom action bar appears: `"X items selected — [Mark All Claimed]"`.
- `[Mark All Claimed]` → opens `BulkClaimModal`.

**`BulkClaimModal`:**

- Lists the selected affiliates and total commission amount being marked.
- `"Confirm — Mark ₱XX,XXX.XX as Claimed"` button.
- `PATCH /api/manager/commissions/bulk-claim` with array of walletIds.

---

#### `/manager/meals` — Banked Free Meals Hub

This page is the **primary UX surface for the Free Meal management workflow.**

**Visual:** Page header: `"Banked Meals — [Branch Name]"`. Stats row at top. Below: searchable list of affiliates with banked meals.

**Stats Row:**

- `StatCard`: "Total Banked Meals" (total count across all affiliates at branch).
- `StatCard`: "Affiliates with Pending Meals" (count of distinct affiliates).

**`BankedMealsList`:**

- Each item is a card representing one affiliate who has ≥1 banked meal.
- Card content: `[Avatar] [Name · Code · Group] [N Meals Banked]` + `[Claim Meals]` button.
- Cards sorted by most banked meals first.
- `<SearchBar>` at top to filter by affiliate name/code.
- `<EmptyState>` if all meals are claimed: "✅ All meals have been claimed! Great job."

**`[Claim Meals]` Button Flow:**

This is the most nuanced interaction. Here's the exact UX:

1. Manager clicks `"Claim Meals"` on an affiliate's card.
2. `ClaimMealModal` opens. Title: `"Claim Banked Meals for [Name]"`.
3. **Modal body:** A checklist of ALL transactions where `free_meal_claimed = false` for that affiliate. Each list item shows:
    - `☐ Trip on [Date] — [Headcount] pax — ₱[Revenue]` and optional notes (e.g., "Will claim next visit").
4. Manager checks the box(es) for the trip(s) the driver is eating their meal for _on this visit_. They can claim one or multiple banked meals at once.
5. `[Confirm Claim (X meals)]` button at bottom.
6. On confirm: `PATCH /api/manager/meals/bulk-claim` with `{ transactionIds: [id1, id2, ...] }`. Updates `free_meal_claimed = true` for those transaction IDs.
7. Toast: `"X meals marked as claimed for [Name]."`.
8. Modal closes. Card updates: if all meals now claimed, card is removed from the list (with a smooth fade-out animation). If some remain, the count updates.

---

#### `/manager/leaderboard`

**Visual:** Same structure as Admin leaderboard but branch-scoped.

**Additions for Manager view:**

- No `<BranchFilter>` (always scoped to manager's branch).
- Affiliation Group filter to compare e.g. "Who is the top GRAB driver?".
- `<DWMFilter>` for period selection.

**`LeaderboardTable`:**

- Columns: `Rank | Name | Code | Group | Trips | Headcount | Total Earned | Pending | Claimed`.
- Top 3 rows highlighted with gold/silver/bronze left border.
- Each row is clickable → opens `AffiliateDetailDrawer`.

---

### 3.4 Affiliate Pages (Mobile-Optimized)

All affiliate pages share the `AffiliateMobileLayout`: fixed header showing `"Hi, [First Name] 👋"` and `"[Driver ID]"`, and a bottom tab bar with icons for: Home, Earnings, Meals, Leaderboard.

---

#### `/affiliate/dashboard`

**Visual:** Stacked vertical cards. No horizontal overflow. All cards full-width. Clean, minimal, easy to read in sunlight.

**`EarningsSummaryCard`:**

- Large card, top of page.
- Three stat blocks side-by-side (mobile 3-column mini-grid):
    - `Total Earned` (lifetime sum of all commission_wallet.amount).
    - `Pending` (sum where status = 'Pending').
    - `Claimed` (sum where status = 'Claimed').
- DWM filter toggle BELOW the card (small pill buttons): `Day | Week | Month` — updates the three stats.
- Currency formatted: `₱ 2,450.00`.

**`MealStatusCard`:**

- Prominent card with fork-and-knife icon.
- Large number: `"3 Meals Banked"`.
- Sub-label: `"Remember to claim these on your next visit!"`.
- If 0: `"No meals banked. You're all caught up! 🎉"` in a green tint.
- Tapping card → navigates to `/affiliate/meals`.

**`RecentActivityFeed`:**

- Section header: `"Recent Trips"`.
- Vertical list of last 5 transaction-linked commission entries.
- Each item: `[Date] | ₱[Revenue] trip → ₱[Commission] earned | [Status Badge]`.
- `"View All Earnings →"` link at bottom.

---

#### `/affiliate/earnings`

**Visual:** Full-width scrollable list. Filter controls at top (sticky).

**Filter Controls:**

- `<DWMFilter>` (Day/Week/Month/Custom) — period filter.
- Status filter: `All | Pending | Claimed` — pill tabs.

**`EarningsHistoryList`:**

- Each item is a card:
    - Top row: `[Date]` (left) + `₱[commission amount]` (right, bold).
    - Bottom row: `Trip ₱[total_revenue] · [Headcount] pax` (left) + `[Status Badge]` (right).
    - If `free_meal_claimed = false` for that transaction: small `"🍴 Meal Banked"` tag inside the card.
- Infinite scroll or "Load More" button (20 items per page).
- Summary at top: `"Showing XX entries · Total: ₱XX,XXX.XX"`.

---

#### `/affiliate/meals`

**Visual:** Simple list of "owed meals" — a read-only view of their banked meal receipts.

**Header Card:** `"You have X banked meals"`. Explanation text: `"Each trip earns you one free meal. The list below shows trips where you haven't eaten your meal yet. Ask your manager to mark it when you do!"`

**Meals List:**

- Each item: `[Trip Date] — [Headcount] pax — ₱[Revenue]` + `"🍴 Meal Not Yet Claimed"` badge.
- If notes exist: shown in smaller italic text below.
- Sorted: most recent trip first.
- `<EmptyState>` if no unclaimed meals.

---

#### `/affiliate/leaderboard`

**Visual:** Full-width list. Affiliate's own row pinned/highlighted.

**`RankCard`** at top of page:

- Displays the affiliate's current rank prominently. e.g., `"You are #4 this month"`.
- Visual rank bar: a horizontal progress bar showing their position relative to #1.
- `"₱[total_earned] earned · [trip_count] trips"`.

**Leaderboard List below:**

- Shows the full ranked list for their branch.
- Their own row: highlighted with a distinct background color.
- Each row: `[Rank] [Avatar] [Name · Code] [Group] [Trips] [Total Earned]`.
- Tapping other rows: no action (read-only for affiliates).
- `<DWMFilter>` at top: Day | Week | Month (so they can see who's leading this week vs. overall).

---

## 4. Data Calculation & API Logic Strategy

This section defines the exact logic for every computed value in the system. Use these as the specification for your API route handlers.

---

### 4.1 Date Range Helper

Every query that accepts a DWM filter uses this shared date resolver. Implement in `lib/utils.ts`:

```ts
// lib/utils.ts
export type DWMPeriod = 'day' | 'week' | 'month' | 'custom';

export function resolveDateRange(period: DWMPeriod, startDate?: Date, endDate?: Date): { start: Date; end: Date } {
  const now = new Date();
  switch (period) {
    case 'day':
      return {
        start: startOfDay(now),
        end: endOfDay(now),
      };
    case 'week':
      return {
        start: startOfWeek(now, { weekStartsOn: 1 }), // Monday
        end: endOfWeek(now, { weekStartsOn: 1 }),
      };
    case 'month':
      return {
        start: startOfMonth(now),
        end: endOfMonth(now),
      };
    case 'custom':
      return { start: startDate!, end: endDate! };
  }
}
```

All API routes accept `?period=week` or `?startDate=&endDate=` query params and call this function internally.

---

### 4.2 Commission Calculation Logic

**The commission amount is computed using Paluto's tier system — not a percentage rate.**

**Rule:** For every ₱1,000 of `total_revenue`, the affiliate earns ₱50. Partial thousands are dropped (floor division).

**Formula:**

```
commission_wallet.amount = Math.floor(total_revenue / 1000) * 50
```

**Examples:**

|Total Revenue|Calculation|Commission Earned|
|---|---|---|
|₱800|`Math.floor(800 / 1000) * 50` = `0 * 50`|₱0.00|
|₱1,000|`Math.floor(1000 / 1000) * 50` = `1 * 50`|₱50.00|
|₱2,500|`Math.floor(2500 / 1000) * 50` = `2 * 50`|₱100.00|
|₱5,999|`Math.floor(5999 / 1000) * 50` = `5 * 50`|₱250.00|
|₱12,300|`Math.floor(12300 / 1000) * 50` = `12 * 50`|₱600.00|

**Shared utility function — implement once in `lib/utils.ts`, import everywhere:**

```ts
// lib/utils.ts
export function calculateTieredCommission(totalRevenue: number): number {
  return Math.floor(totalRevenue / 1000) * 50;
}
```

**Live Auto-fill on the Front-End (`TransactionForm`):**

The Commission Amount field is pre-filled using this function as the manager types, but remains a fully editable `<Input>`. This is not a read-only preview — the manager can always override the value.

```ts
// hooks/useTransactionForm.ts
const { watch, setValue } = useFormContext();
const totalRevenue = watch('totalRevenue');

useEffect(() => {
  const revenue = parseFloat(totalRevenue);
  if (!isNaN(revenue) && revenue >= 0) {
    // Only auto-fill if the manager hasn't manually overridden
    if (!isManuallyOverridden) {
      setValue('commissionAmount', calculateTieredCommission(revenue));
    }
  }
}, [totalRevenue]);

// Track whether the manager has manually touched the commission field
const handleCommissionManualChange = () => {
  setIsManuallyOverridden(true);
};

// Reset override flag if revenue changes significantly (optional UX choice)
const handleRevenueChange = () => {
  setIsManuallyOverridden(false); // Re-run auto-fill on new revenue input
};
```

**Visual state of the Commission Amount field:**

|Condition|Field Appearance|
|---|---|
|Value matches tier formula result|Ghost pill: `"Auto-calculated"` (gray, subtle)|
|Manager has manually edited value|Pill changes to `"Manually overridden"` (amber) — informational only, does NOT block submit|
|Revenue is 0 or empty|Field is empty, no pill shown|

**Server-side API Logic (`POST /api/manager/transactions`):**

The API **does NOT recalculate** the commission. It trusts the `commissionAmount` sent from the form (whether auto-filled or manually overridden by the manager). This is intentional — the manager's override is the source of truth.

```ts
// api/manager/transactions/route.ts — POST handler

const {
  affiliateId, branchId, headcount, totalRevenue, commissionAmount,
  transactionDate, referralSuccess, freeMealClaimed,
  foodStamp, foodStampType, notes
} = body;

// Validate commissionAmount is a non-negative number
if (typeof commissionAmount !== 'number' || commissionAmount < 0) {
  return NextResponse.json({ error: 'Invalid commission amount' }, { status: 400 });
}

// INDEPENDENCE RULE: referralSuccess, commissionAmount, and freeMealClaimed
// are all stored independently. Do NOT gate commission or meal on referralSuccess.
// Save whatever values the manager submitted.

// Insert transaction row
const transaction = await db.transactions.create({
  data: {
    affiliateId,
    branchId,
    headcount,
    totalRevenue,
    transactionDate,
    referralSuccess,   // Stored as-is, does not affect commission or meal
    freeMealClaimed,   // Stored as-is, does not depend on referralSuccess
    foodStamp,
    foodStampType: foodStamp ? foodStampType : null,
    notes,
  }
});

// Insert commission_wallet using manager-submitted amount (not recalculated)
await db.commissionWallet.create({
  data: {
    transactionId: transaction.id,
    amount: commissionAmount,  // From form — may be auto-calculated or manually overridden
    status: 'Pending',
  }
});

return NextResponse.json({ transaction, success: true });
```

> **Why the server doesn't recalculate:** The tier formula is simple and deterministic, but managers have the authority to grant different amounts (e.g., bonuses, corrections for special cases). Trusting the submitted `commissionAmount` makes this explicit and auditable. If needed in the future, an audit log comparing submitted vs. tier-expected amounts can be added without changing this architecture.

---

### 4.3 Banked Free Meals — Query & Update Logic

**Calculating how many meals an affiliate has banked:**

```sql
-- Count of banked (unclaimed) meals for a specific affiliate
SELECT COUNT(*) AS banked_meal_count
FROM transactions
WHERE affiliate_id = :affiliateId
  AND free_meal_claimed = FALSE;
```

**Fetching the list of banked meal "receipts" for the ClaimMealModal:**

```sql
-- Get the specific transaction records for the modal checklist
SELECT
  t.id,
  t.transaction_date,
  t.headcount,
  t.total_revenue,
  t.notes
FROM transactions t
WHERE t.affiliate_id = :affiliateId
  AND t.free_meal_claimed = FALSE
ORDER BY t.transaction_date DESC;
```

**Branch-wide banked meals count (for Manager Dashboard `BankedMealsAlert`):**

```sql
-- Count of all unclaimed meals at a branch
SELECT COUNT(*) AS total_banked
FROM transactions
WHERE branch_id = :branchId
  AND free_meal_claimed = FALSE;

-- Count of distinct affiliates who have at least 1 banked meal
SELECT COUNT(DISTINCT affiliate_id) AS affiliates_with_banked
FROM transactions
WHERE branch_id = :branchId
  AND free_meal_claimed = FALSE;
```

**`BankedMealsList` (Manager Meals Page) — Affiliates grouped with their banked count:**

```sql
-- Affiliates with banked meal counts, sorted by most banked first
SELECT
  a.id,
  a.first_name,
  a.last_name,
  a.driver_id,
  a.affiliation_group,
  a.type,
  COUNT(t.id) AS banked_meal_count
FROM affiliates a
INNER JOIN transactions t ON t.affiliate_id = a.id
WHERE t.branch_id = :branchId
  AND t.free_meal_claimed = FALSE
GROUP BY a.id, a.first_name, a.last_name, a.driver_id, a.affiliation_group, a.type
ORDER BY banked_meal_count DESC;
```

**Marking meals as claimed (`PATCH /api/manager/meals/bulk-claim`):**

```ts
// Request body: { transactionIds: number[] }
// Update all specified transaction IDs at once
await db.transactions.updateMany({
  where: {
    id: { in: transactionIds },
    branch_id: managerBranchId, // SECURITY: Only manager's own branch
    free_meal_claimed: false,   // Idempotency: don't re-claim
  },
  data: { free_meal_claimed: true },
});
```

**Security note:** Always include `branch_id = MANAGER_BRANCH` in the WHERE clause for all Manager write operations to prevent cross-branch manipulation.

---

### 4.4 Commission Status — Pending vs. Claimed

**Total Pending Commissions (Manager Dashboard StatCard):**

```sql
SELECT COALESCE(SUM(cw.amount), 0) AS total_pending
FROM commission_wallet cw
INNER JOIN transactions t ON t.id = cw.transaction_id
WHERE cw.status = 'Pending'
  AND t.branch_id = :branchId
  AND t.transaction_date BETWEEN :startDate AND :endDate;
```

**Total Claimed Commissions (same structure, filter `status = 'Claimed'`):**

```sql
SELECT COALESCE(SUM(cw.amount), 0) AS total_claimed,
       cw.payout_date
FROM commission_wallet cw
INNER JOIN transactions t ON t.id = cw.transaction_id
WHERE cw.status = 'Claimed'
  AND t.branch_id = :branchId
  AND t.transaction_date BETWEEN :startDate AND :endDate;
```

**Marking a Commission as Claimed (`PATCH /api/manager/commissions/[walletId]/claim`):**

```ts
await db.commissionWallet.update({
  where: {
    id: walletId,
    // Verify this commission belongs to the manager's branch (join through transaction)
    transaction: { branch_id: managerBranchId },
    status: 'Pending', // Idempotency: can't re-claim
  },
  data: {
    status: 'Claimed',
    payout_date: new Date(),
  },
});
```

**Bulk Claim (`PATCH /api/manager/commissions/bulk-claim`):**

```ts
// body: { walletIds: number[] }
await db.commissionWallet.updateMany({
  where: {
    id: { in: walletIds },
    transaction: { branch_id: managerBranchId },
    status: 'Pending',
  },
  data: { status: 'Claimed', payout_date: new Date() },
});
```

---

### 4.5 Leaderboard Ranking Query

**Manager Leaderboard (Branch-scoped):**

```sql
SELECT
  a.id                            AS affiliate_id,
  a.first_name,
  a.last_name,
  a.driver_id,
  a.affiliation_group,
  a.type,
  COUNT(t.id)                     AS trip_count,
  SUM(t.headcount)                AS total_headcount,
  COALESCE(SUM(cw.amount), 0)    AS total_earned,
  RANK() OVER (ORDER BY COALESCE(SUM(cw.amount), 0) DESC) AS rank
FROM affiliates a
LEFT JOIN transactions t
  ON t.affiliate_id = a.id
  AND t.branch_id = :branchId
  AND t.transaction_date BETWEEN :startDate AND :endDate
LEFT JOIN commission_wallet cw
  ON cw.transaction_id = t.id
WHERE a.branch_id = :branchId
  -- Optional type filter:
  AND (:affiliateType IS NULL OR a.type = :affiliateType)
  -- Optional affiliation group filter:
  AND (:affiliationGroup IS NULL OR a.affiliation_group = :affiliationGroup)
GROUP BY a.id, a.first_name, a.last_name, a.driver_id, a.affiliation_group, a.type
ORDER BY total_earned DESC;
```

**Admin Leaderboard (Cross-branch):**

```sql
-- Same as above but with optional branch filter
WHERE ('' = :branchId OR a.branch_id = :branchId)
  AND t.branch_id = a.branch_id  -- trip must be at their primary branch
```

**Affiliate's own rank (for `/affiliate/leaderboard` `RankCard`):**

- Run the full branch leaderboard query.
- Find the row where `affiliate_id = SESSION_AFFILIATE_ID`.
- Return the `rank` and `total_earned` values alongside the full list.

---

### 4.6 Admin Global KPI Stats

**`GET /api/admin/stats?period=week`**

Returns all of these in a single response object to minimize round-trips:

```sql
-- 1. Total Revenue (All Branches)
SELECT COALESCE(SUM(total_revenue), 0) AS total_revenue_global
FROM transactions
WHERE transaction_date BETWEEN :startDate AND :endDate;

-- 2. Total Commissions Paid Out (status = Claimed)
SELECT COALESCE(SUM(cw.amount), 0) AS total_paid_out
FROM commission_wallet cw
INNER JOIN transactions t ON t.id = cw.transaction_id
WHERE cw.status = 'Claimed'
  AND t.transaction_date BETWEEN :startDate AND :endDate;

-- 3. Total Pending Commissions
SELECT COALESCE(SUM(cw.amount), 0) AS total_pending
FROM commission_wallet cw
INNER JOIN transactions t ON t.id = cw.transaction_id
WHERE cw.status = 'Pending'
  AND t.transaction_date BETWEEN :startDate AND :endDate;

-- 4. Total Active Affiliates
-- "Active" = has at least 1 transaction in the chosen period
SELECT COUNT(DISTINCT affiliate_id) AS active_affiliates
FROM transactions
WHERE transaction_date BETWEEN :startDate AND :endDate;

-- 5. Previous period values (for % change cards)
-- Run the same queries with the PREVIOUS equivalent period
-- (e.g., for 'week', previous period = 7 days before start)
```

The API computes percentage change:

```ts
const revenueChange = ((currentRevenue - previousRevenue) / previousRevenue) * 100;
// Returns: { value: 45000, change: +12.5 }
```

---

### 4.7 Branch Comparison Table (Admin Dashboard)

**`GET /api/admin/branches/comparison?period=week`**

```sql
SELECT
  b.id,
  b.name                          AS branch_name,
  b.location,
  COUNT(DISTINCT t.id)            AS total_trips,
  COALESCE(SUM(t.headcount), 0)  AS total_headcount,
  COALESCE(SUM(t.total_revenue), 0) AS total_revenue,
  COALESCE(SUM(cw.amount), 0)   AS total_commission,
  COALESCE(SUM(CASE WHEN cw.status = 'Pending' THEN cw.amount END), 0) AS pending_commission,
  COALESCE(SUM(CASE WHEN cw.status = 'Claimed' THEN cw.amount END), 0) AS claimed_commission,
  COUNT(DISTINCT a.id)           AS active_affiliates
FROM branches b
LEFT JOIN transactions t
  ON t.branch_id = b.id
  AND t.transaction_date BETWEEN :startDate AND :endDate
LEFT JOIN commission_wallet cw
  ON cw.transaction_id = t.id
LEFT JOIN affiliates a
  ON a.branch_id = b.id
GROUP BY b.id, b.name, b.location
ORDER BY b.id;
```

---

### 4.8 Data Match Report — Affiliation Group Breakdown

**`GET /api/admin/reports/data-match?period=month&branchId=all`**

This is the core analytics report. Groups all performance metrics by `affiliation_group`.

```sql
SELECT
  a.affiliation_group,
  COUNT(DISTINCT a.id)             AS affiliate_count,
  COUNT(t.id)                      AS trip_count,
  COALESCE(SUM(t.headcount), 0)   AS total_headcount,
  COALESCE(SUM(t.total_revenue), 0) AS total_revenue,
  COALESCE(SUM(cw.amount), 0)    AS total_commission,
  CASE
    WHEN COUNT(t.id) > 0
    THEN ROUND(SUM(t.total_revenue) / COUNT(t.id), 2)
    ELSE 0
  END                              AS avg_revenue_per_trip,
  CASE
    WHEN COUNT(t.id) > 0
    THEN ROUND(SUM(t.headcount) / COUNT(t.id), 1)
    ELSE 0
  END                              AS avg_headcount_per_trip
FROM affiliates a
LEFT JOIN transactions t
  ON t.affiliate_id = a.id
  AND t.transaction_date BETWEEN :startDate AND :endDate
  AND (:branchId = 'all' OR t.branch_id = :branchId)
LEFT JOIN commission_wallet cw
  ON cw.transaction_id = t.id
WHERE (:branchId = 'all' OR a.branch_id = :branchId)
GROUP BY a.affiliation_group
ORDER BY total_revenue DESC;
```

**Totals row** is computed client-side by summing each column from the returned rows.

---

### 4.9 Affiliate Search Combobox (`AffiliateSearchCombobox`)

**`GET /api/manager/affiliates/search?q=D-005&branchId=2`**

**Driver ID normalization is mandatory here.** The input `"d-0055"` and `"D-0055"` must resolve to the same affiliate. Normalize both the search query and the stored DB value.

**Client-side (before sending the API request):**

```ts
// hooks/useAffiliateSearch.ts
const normalizedQuery = query.trim().toUpperCase(); // "d-0055" → "D-0055"
// Send normalizedQuery as the `q` param
```

**API route — normalize the incoming `q` param:**

```ts
// api/manager/affiliates/search/route.ts
const rawQuery = searchParams.get('q') ?? '';
const q = rawQuery.trim().toUpperCase(); // Always normalize before querying
const queryParam = `%${q}%`;
```

**SQL query (uses `UPPER()` on the DB column for safe case-insensitive match):**

```sql
SELECT
  a.id,
  a.first_name,
  a.last_name,
  a.driver_id,
  a.contact_number,
  a.affiliation_group,
  a.type
FROM affiliates a
WHERE a.branch_id = :branchId
  AND (
    UPPER(a.driver_id) LIKE :queryParam        -- e.g., '%D-005%'
    OR UPPER(a.first_name) LIKE :queryParam
    OR UPPER(a.last_name) LIKE :queryParam
    OR UPPER(CONCAT(a.first_name, ' ', a.last_name)) LIKE :queryParam
  )
ORDER BY a.driver_id ASC
LIMIT 10;
```

> **Note:** `commission_rate` is no longer returned in this response — it is not needed since the commission formula is global (`calculateTieredCommission()`), not per-affiliate.

- Query param `:queryParam` is the normalized, `%`-wrapped search string.
- Response populates the combobox dropdown. On selection, `contact_number` and `affiliation_group` auto-populate the read-only display fields.

---

### 4.10 Affiliate Personal Dashboard Stats

**`GET /api/affiliate/me/stats?period=month`** (Session identifies the affiliate)

```sql
-- 1. Personal earnings summary
SELECT
  COALESCE(SUM(cw.amount), 0) AS total_earned,
  COALESCE(SUM(CASE WHEN cw.status = 'Pending' THEN cw.amount END), 0) AS pending,
  COALESCE(SUM(CASE WHEN cw.status = 'Claimed' THEN cw.amount END), 0) AS claimed,
  COUNT(t.id) AS trip_count
FROM transactions t
INNER JOIN commission_wallet cw ON cw.transaction_id = t.id
WHERE t.affiliate_id = :affiliateId
  AND t.transaction_date BETWEEN :startDate AND :endDate;

-- 2. Banked meals count
SELECT COUNT(*) AS banked_meals
FROM transactions
WHERE affiliate_id = :affiliateId
  AND free_meal_claimed = FALSE;

-- 3. Last trip date
SELECT MAX(transaction_date) AS last_trip_date
FROM transactions
WHERE affiliate_id = :affiliateId;
```

---

### 4.11 Revenue Trend Chart Data

**`GET /api/admin/reports/revenue-trend?period=month&branchId=all`**

Returns time-bucketed revenue data for the line chart.

```sql
-- Monthly period: bucket by day
SELECT
  DATE_TRUNC('day', t.transaction_date) AS bucket,
  b.name AS branch_name,
  COALESCE(SUM(t.total_revenue), 0) AS revenue
FROM transactions t
INNER JOIN branches b ON b.id = t.branch_id
WHERE t.transaction_date BETWEEN :startDate AND :endDate
GROUP BY DATE_TRUNC('day', t.transaction_date), b.name
ORDER BY bucket ASC;
```

The front-end transforms this flat array into a Recharts-compatible structure:

```ts
// Transform: [{ bucket, branch_name, revenue }] →
// { dates: ['2024-01-01', ...], series: { [branchName]: [revenue, ...] } }
// Branch names come from the DB — do NOT hardcode them as keys.
// Use Object.entries(series) to render each line dynamically.
```

---

### 4.12 Driver ID Auto-Generation

When the Manager opens the `AffiliateForm` to onboard a new affiliate, the system suggests the next sequential driver ID:

**`GET /api/manager/affiliates/next-id?type=driver`**

```sql
-- Get the highest existing numeric suffix for the given type prefix
SELECT driver_id
FROM affiliates
WHERE UPPER(driver_id) LIKE 'D-%'   -- or 'G-%' for tour_guide
ORDER BY driver_id DESC
LIMIT 1;
-- Result: "D-0054"
-- Front-end displays suggestion: "D-0055" (increment by 1, zero-padded to 4 digits)
```

---

### 4.13 State Management Summary

|State|Tool|Location|Notes|
|---|---|---|---|
|Auth session & role|NextAuth + Zustand|`authStore.ts`|Role, userId, branchId, fullName|
|DWM filter date range|Zustand|`filterStore.ts`|Shared across dashboard widgets on same page|
|Server data (API fetches)|TanStack React Query|Component-level `useQuery`|Cached, refetched on focus|
|Transaction form state|React Hook Form + Zod|`useTransactionForm.ts`|Validation + live commission preview|
|Modal/Drawer open state|React `useState`|Local to parent component|Never lifted to global store|
|Combobox search input|React `useState` + `useDebounce`|`AffiliateSearchCombobox.tsx`|300ms debounce before API call|

---

### 4.14 Security & Access Control Checklist

Every API route must enforce these guards. No exceptions.

|Guard|Implementation|
|---|---|
|Authentication|Check `getServerSession()` at the top of every route handler. 401 if no session.|
|Role enforcement|Check `session.user.role`. 403 if wrong role for that endpoint.|
|Branch scoping (Manager)|All queries include `WHERE branch_id = session.user.branchId`. Never trust client-sent branchId.|
|Affiliate scoping|All `/api/affiliate/*` routes include `WHERE affiliate_id = session.user.affiliateId`.|
|Cross-branch meal/commission claim|Include branch join in WHERE clause for all PATCH/UPDATE operations.|
|Input validation|Zod schema validation at the API route entry point (before any DB call).|
|SQL injection prevention|Use parameterized queries only (Prisma/Drizzle handles this automatically).|

### 4.15 Field Independence Rules

**`referral_success`, `commissionAmount`, and `free_meal_claimed` are fully independent fields. No field gates, blocks, or conditions another — ever.**

This is a hard architectural rule. The following patterns are explicitly forbidden:

```ts
// ❌ WRONG — never do this
if (!referralSuccess) {
  commissionAmount = 0; // FORBIDDEN
}

// ❌ WRONG — never do this
if (!referralSuccess) {
  freeMealClaimed = false; // FORBIDDEN
}

// ❌ WRONG — never do this
if (!referralSuccess) {
  return NextResponse.json({ error: 'No commission for failed referrals' }, { status: 400 }); // FORBIDDEN
}
```

**Why this matters:** Managers use `referral_success` as an informational tag (e.g., "the customer didn't actually mention the driver's name"). However, Paluto's business rules still allow them to grant meals and commissions at their discretion regardless of that flag. The toggle captures ground-truth data without enforcing punitive logic.

**Correct pattern:**

```ts
// ✅ CORRECT — all three fields stored and processed independently
await db.transactions.create({
  data: {
    referralSuccess,    // Stored for reporting — does not affect commission or meal
    freeMealClaimed,    // Stored as-is — does not depend on referralSuccess
  }
});

await db.commissionWallet.create({
  data: {
    amount: commissionAmount,  // Stored as-is — does not depend on referralSuccess or freeMealClaimed
    status: 'Pending',
  }
});
```

**UI implications:** The `referral_success` toggle on the TransactionForm must have no visual coupling to the Commission Amount field or the Free Meal toggle. Toggling it on/off should not disable, gray out, or modify any other field. Its only effect is the value stored in `transactions.referral_success`.

---

### 4.16 Driver ID Normalization — Full Standard

Driver IDs must be stored, searched, and displayed in a **consistent uppercase format** system-wide. A driver using `"d-0055"` on a physical card, a manager searching `"D-0055"`, and the record in the DB reading `"D-0055"` must all be treated as identical.

**Implement the normalizer once in `lib/utils.ts`:**

```ts
// lib/utils.ts
export function normalizeDriverId(raw: string): string {
  return raw.trim().toUpperCase();
}

// Usage examples:
normalizeDriverId('d-0055')  // → 'D-0055'
normalizeDriverId('  G-0012 ') // → 'G-0012'
normalizeDriverId('g0012')   // → 'G0012' (valid if that's the format)
```

**Apply at every write boundary:**

|Location|Where to Apply|
|---|---|
|`AffiliateForm` (onboard)|`normalizeDriverId(formData.driverId)` before `POST /api/manager/affiliates`|
|`AffiliateForm` (edit)|Same — normalize before `PATCH` request|
|API `POST /api/manager/affiliates`|`driver_id: normalizeDriverId(body.driverId)` before DB insert|
|DB `INSERT` / `UPDATE`|Always write the normalized value|

**Apply at every read/search boundary:**

|Location|Where to Apply|
|---|---|
|`AffiliateSearchCombobox` (client)|`normalizeDriverId(inputValue)` before debounced API call|
|`GET /api/manager/affiliates/search`|`normalizeDriverId(searchParams.get('q'))` before SQL query|
|SQL `WHERE` clause|`UPPER(a.driver_id) LIKE :param` — never rely on exact-case match in DB|
|`TransactionTable` display|Display `a.driver_id` as-is (already normalized in DB)|

**Auto-generation of new IDs (reviewed from §4.12):**

```ts
// api/manager/affiliates/next-id/route.ts
const prefix = affiliateType === 'driver' ? 'D' : 'G';

// Query normalized values — UPPER() ensures consistent sort order
const latest = await db.$queryRaw`
  SELECT driver_id FROM affiliates
  WHERE UPPER(driver_id) LIKE ${prefix + '-%'}
  ORDER BY driver_id DESC
  LIMIT 1
`;

// Parse the numeric suffix and increment
const latestId = latest[0]?.driver_id ?? `${prefix}-0000`;
const currentNum = parseInt(latestId.split('-')[1], 10);
const nextId = `${prefix}-${String(currentNum + 1).padStart(4, '0')}`;
// e.g., "D-0055" → "D-0056"

return NextResponse.json({ suggestedId: nextId });
```

**Zod validator for driver ID format:**

```ts
// lib/validators.ts
export const driverIdSchema = z
  .string()
  .trim()
  .toUpperCase()
  .regex(/^[DG]-\d{4}$/, 'Driver ID must be in format D-0001 or G-0001');
```

---

_End of Paluto Affiliate Tracking System — Front-End Planning Document v1.2_
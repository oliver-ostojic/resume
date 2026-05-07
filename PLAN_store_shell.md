# Store Shell Refactor Plan

**Goal:** Eliminate 10-second fairness health latency by keeping the outer store card mounted and prefetching dashboard data immediately on store open — instead of waiting for the user to navigate to `/fairness-dashboard`.

**Root cause:** `useAvailableDates` + `useDashboardData` only fire when `/fairness-dashboard` page mounts (sequential waterfall, triggered too late).

**Fix:** Single `StoreShell` component stays mounted across all top-nav tabs. Data hooks live in the shell and fire immediately. Tab switches are state changes, not route navigations.

---

## Phase 1 — Modularize `home/page.tsx`

**Goal:** Reduce 3453-line monolith to ~300-line orchestrator. Required before Phase 2 so views can be lifted into the shell.

### 1a. Extract utility functions
- [ ] Create `apps/web/app/stores/[storeId]/home/utils.ts`
- [ ] Move: `parseLocalDate`, `formatLogbookDate`, `formatShortDate`, `capitalizeStatus`, `formatRelativeTime`, `getActivityDisplayType`, `formatActivityDate`, `groupRulesByType`
- [ ] Update imports in `page.tsx`
- [ ] Build — fix errors

### 1b. Extract inline UI components
- [ ] Create `apps/web/app/stores/[storeId]/home/components/ListRowItemLight.tsx`
- [ ] Create `apps/web/app/stores/[storeId]/home/components/RoleRuleCard.tsx`
- [ ] Create `apps/web/app/stores/[storeId]/home/components/SentenceBubbleItem.tsx`
- [ ] Add to `components/index.ts` barrel
- [ ] Update imports in `page.tsx`
- [ ] Build — fix errors

### 1c. Extract data fetching hook
- [ ] Create `apps/web/app/stores/[storeId]/home/useHomeData.ts`
- [ ] Move: all `useState` for `apiStore`, `apiCrew`, `apiRoles`, `apiRoleFamilies`, `apiPreferences`, `apiRuns`, `apiLogbooks`, `dataLoaded`
- [ ] Move: `fetchData` / `refreshData` logic
- [ ] Move: activity log fetch + `activityLogs`, `activityFilter`, `activityUserFilter`, `activityPage` state
- [ ] Return everything the page needs
- [ ] Update `page.tsx` to use the hook
- [ ] Build — fix errors

### 1d. Extract view components
Each view gets its own file in `components/`. Each receives data + callbacks as props (no direct API calls inside).

- [ ] Create `HomeView.tsx` — overview cards + activity log (lines ~2267–2600 in current page.tsx)
- [ ] Create `CrewView.tsx` — list + right detail panel for crew
- [ ] Create `RolesView.tsx` — list + right detail panel for roles
- [ ] Create `PreferencesView.tsx` — list + right detail panel for preferences
- [ ] Create `LogbooksView.tsx` — list + right detail panel for logbooks
- [ ] Add all five to `components/index.ts`
- [ ] Reduce `page.tsx` to pure orchestration: state + hook calls + conditional render
- [ ] Build — fix errors

**Checkpoint:** `home/page.tsx` is ≤400 lines. All five views render correctly. No regressions.

---

## Phase 2 — Create `StoreShell`

**Goal:** Single mounted wrapper that holds data hooks + switches inner content via state instead of route navigation.

### 2a. Modify `TopNavHeader` to support callback mode
- [ ] Add optional prop `onNavChange?: (tab: 'home' | 'dashboard' | 'settings') => void`
- [ ] When `onNavChange` is provided, call it instead of `router.push()`
- [ ] When not provided, keep existing `router.push()` behavior (backwards compat)
- [ ] Build — fix errors

### 2b. Create `FairnessContent.tsx`
- [ ] Create `apps/web/app/stores/[storeId]/components/FairnessContent.tsx`
- [ ] Extract render body from `fairness-dashboard/page.tsx` (everything below the data hooks)
- [ ] Props: `storeId`, `availableDatesData`, `dashboardQueryData`, `isDashboardLoading` — data passed in from shell, not fetched internally
- [ ] Keep all local UI state (`activeView`, `selectedCrew`, `selectedRole`, pagination, search, etc.) inside this component
- [ ] Keep `useDashboardComputed` inside this component (it's computation, not fetching)
- [ ] Keep tutorial effects inside this component
- [ ] Build — fix errors

### 2c. Create `SettingsContent.tsx`
- [ ] Create `apps/web/app/stores/[storeId]/components/SettingsContent.tsx`
- [ ] Extract render body from `settings/page.tsx`
- [ ] Props: `storeId`
- [ ] Keep all settings-specific state + fetch inside this component (settings data is small, no need to prefetch)
- [ ] Build — fix errors

### 2d. Create `StoreShell.tsx`
- [ ] Create `apps/web/app/stores/[storeId]/components/StoreShell.tsx`
- [ ] Props: `initialTab: 'home' | 'dashboard' | 'settings'`
- [ ] State: `activeTopNav`
- [ ] Data hooks (fire immediately on shell mount):
  ```ts
  const { data: availableDatesData } = useAvailableDates(storeId)
  // datesToFetch = most recent available dates up to 30, derived once availableDatesData resolves
  const datesToFetch = useMemo(() =>
    availableDatesData?.dates.slice(-30) ?? [], [availableDatesData])
  const { data: dashboardQueryData, isLoading } = useDashboardData(storeId, datesToFetch)
  ```
  Both hooks start at shell mount (store open). `useDashboardData` fires as soon as `useAvailableDates` resolves — the sequential wait starts ~1s after store open instead of only after tab click. User typically spends 10-30s on home view, enough time for both to complete in the background.
- [ ] Outer glass card + `TopNavHeader` with `onNavChange={setActiveTopNav}` always rendered
- [ ] Conditional inner content:
  ```tsx
  {activeTopNav === 'home' && <HomePageContent storeId={storeId} />}
  {activeTopNav === 'dashboard' && <FairnessContent ... />}
  {activeTopNav === 'settings' && <SettingsContent storeId={storeId} />}
  ```
- [ ] Build — fix errors

**Checkpoint:** Switching between Home / Fairness Health / Settings tabs is instant. Network tab confirms `dashboard/dates` fires within 1s of store page load and `dashboard/logbooks` fires immediately after it resolves — both well before any tab click.

---

## Phase 3 — Wire routes

**Goal:** All three routes render `StoreShell`. Direct links still work. Old routes do not 404.

- [ ] Replace `home/page.tsx` body with `<StoreShell storeId={storeId} initialTab="home" />`
- [ ] Replace `fairness-dashboard/page.tsx` body with `<StoreShell storeId={storeId} initialTab="dashboard" />`
- [ ] Replace `settings/page.tsx` body with `<StoreShell storeId={storeId} initialTab="settings" />`
- [ ] Verify all three routes render correctly
- [ ] Verify browser back/forward still works as expected
- [ ] Build — fix errors

**Final checkpoint:** 
- Navigate to a store → open Network tab → confirm `dashboard/dates` and `dashboard/logbooks` requests fire within 2s
- Click "System Health" → content renders immediately (no spinner)
- Direct link to `/fairness-dashboard` still works
- No console errors

---

## File Map (end state)

```
apps/web/app/stores/[storeId]/
├── components/
│   ├── StoreShell.tsx          ← NEW: outer shell, data hooks, tab state
│   ├── FairnessContent.tsx     ← NEW: extracted from fairness-dashboard/page.tsx
│   └── SettingsContent.tsx     ← NEW: extracted from settings/page.tsx
├── home/
│   ├── page.tsx                ← SLIM: just mounts StoreShell
│   ├── useHomeData.ts          ← NEW: extracted data fetching
│   ├── utils.ts                ← NEW: extracted utility functions
│   └── components/
│       ├── HomeView.tsx        ← NEW: home tab content
│       ├── CrewView.tsx        ← NEW: crew list + detail
│       ├── RolesView.tsx       ← NEW: roles list + detail
│       ├── PreferencesView.tsx ← NEW: preferences list + detail
│       ├── LogbooksView.tsx    ← NEW: logbooks list + detail
│       ├── ListRowItemLight.tsx ← NEW: extracted inline component
│       ├── RoleRuleCard.tsx    ← NEW: extracted inline component
│       ├── SentenceBubbleItem.tsx ← NEW: extracted inline component
│       └── ... (existing components unchanged)
├── fairness-dashboard/
│   └── page.tsx                ← SLIM: just mounts StoreShell with initialTab="dashboard"
└── settings/
    └── page.tsx                ← SLIM: just mounts StoreShell with initialTab="settings"
```

---

## Commit cadence

- After 1a+1b: `[date-T1] refactor(home): extract utils and inline components`
- After 1c: `[date-T2] refactor(home): extract useHomeData hook`
- After 1d: `[date-T3] refactor(home): extract view components, slim page.tsx`
- After 2a-2d: `[date-T4] feat(shell): add StoreShell with fairness data prefetch`
- After 3: `[date-T5] feat(shell): wire all three routes through StoreShell`

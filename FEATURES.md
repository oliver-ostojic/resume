# Logbook Writer — Feature Bank

> **Purpose:** Resume ammunition. Each feature entry has enough technical and contextual detail to generate tailored bullets for different job types.
> Filter by `Tags` to pull relevant features per application, then ask Claude to synthesize into polished bullets.
>
> **Job type tags:** `[backend]` `[optimization]` `[ml-adjacent]` `[data-modeling]` `[systems]` `[frontend]` `[fullstack]` `[devops]` `[algorithms]`

---

## 1. Solver & Optimization

---

### CP-SAT Constraint Programming Model

Builds and solves a mixed-integer constraint program that assigns crew members to roles across a discretized daily time grid. Decision variables are `(crew, slot, role, task_slots)` BoolVars; time is discretized by computing the GCD of all role `taskLength` values (default 30-min slots). The solver maximizes a weighted objective while satisfying hard and soft constraints.

**Key files:** `apps/solver-python/logbook_solver_v2/solver_v2.py`, `variables.py`, `time_grid.py`
**Tech:** Python, Google OR-Tools CP-SAT, `cp_model.CpModel`, parallel portfolio search (`num_search_workers = os.cpu_count()`), Large Neighborhood Search (LNS enabled)
**Tags:** `[optimization]` `[algorithms]` `[backend]` `[systems]`

---

### Hard Constraint Enforcement

Translates scheduling business rules into CP-SAT constraints that the solver cannot violate. Constraint types include:

- **One task per slot** — for each `(crew, slot)` pair, sum of all covering variables ≤ 1
- **Coverage window enforcement** — slot-level crew demand must satisfy `MIN`/`MAX`/`EXACTLY` mode per `RoleCoverageWindow`
- **Consecutive REQUIRED** — triplet no-gap constraint: if no middle slot variable exists, adjacent pair variables ≤ 1
- **Role family min/max** — summed minutes per crew across a role family bounded by `[minMinutes, maxMinutes]`; gated by minimum shift length
- **Timing deadlines** — `ASSIGN_BEFORE/AFTER_SHIFT_MIN_X` forces variables outside the constraint window to 0

**Key files:** `apps/solver-python/logbook_solver_v2/constraints.py`
**Tech:** OR-Tools `model.Add()`, `AddMaxEquality()`, `AddImplication()`, `AddBoolAnd()`
**Tags:** `[optimization]` `[algorithms]` `[backend]`

---

### Multi-Term Objective Function

Maximization objective that balances 10+ reward/penalty signals simultaneously:

- **Coverage reward** — flat 100 pts/assignment to incentivize full slot coverage
- **Half-block penalty** — −70 pts for gap-filling half-sized blocks (net 30 vs. 100)
- **Hour-alignment bonus** — +15 for tasks ≥ 60 min starting on hour boundaries
- **Consecutive bonus/gap penalty** — +10/adjacent pair for `PREFERRED` consecutive roles; −200/gap via XOR auxiliary BoolVars
- **Timing gradient** — EARLY/LATE/MIDDLE preferences mapped to position ratios within the shift window, scaled by rule weight
- **Hour preference** — `LIKE/DISLIKE_ROLE_FOR_HOUR_X` adds/subtracts per-hour objective weight
- **Z-score fairness terms** — under-assigned crew boosted by `|z| * fairnessBoost * task_slots`; over-assigned penalized symmetrically
- **Tiered rotation boost** — `BASE_BOOST * normalized_mph` per variable, where `normalized_mph = 1 − (mph − min) / (max − min)`
- **Intra-day spread penalty** — −2000/extra block per tracked role per crew per day
- **Quota shortfall penalty** — −1000/minute of unmet quota via `shortfall_var`

**Key files:** `apps/solver-python/logbook_solver_v2/objective.py`
**Tech:** OR-Tools integer linear expressions, `model.Maximize()`, auxiliary `BoolVar`/`IntVar`, optional quadratic penalty mode via `AddMultiplicationEquality`
**Tags:** `[optimization]` `[algorithms]` `[ml-adjacent]`

---

### Role Rule Constraint System

16 named rule types (`RoleRuleType` enum) stored as Postgres records and translated into CP-SAT constraints or objective terms at solve time. Includes a priority cascade where crew-level rules override store defaults for the same `roleId:type:targetRoleId` base key.

Notable rule implementations:
- **FORBID_ROLE** — forces all `(crew, slot, role)` vars to 0
- **MIN_CONSECUTIVE_MINUTES** — auxiliary `starts_block` vars enforce minimum run length via `AddImplication`
- **MAX_CONSECUTIVE_MINUTES (HARD)** — sliding window: every window of `max_slots + 1` slots sums to ≤ `max_slots`
- **MAX_CONSECUTIVE_MINUTES (SOFT)** — BoolVar violation counter, 10,000 pts/violation
- **CANNOT_BE_ASSIGNED_BEFORE/AFTER** — adjacency check: `role_end_slot == target_start_slot` triggers hard or soft penalty

**Key files:** `apps/solver-python/logbook_solver_v2/role_rules.py`, `apps/api/src/solver2/builder.ts`
**Tech:** OR-Tools, `AddImplication`, `AddMaxEquality`, priority resolution in TypeScript before payload is sent to Python
**Tags:** `[optimization]` `[backend]` `[algorithms]` `[data-modeling]`

---

### Portfolio Tuning Engine

Multi-region parallel search across 12 seeds with a ladder-style time budget (3 shots × 15 s/region = 45 s/region). After all regions solve, selects the result that best balances objective score and fairness index. `BASE_BOOST=740` was empirically determined across a 4-phase sweep validated on 31-day simulations, achieving Gini coefficient=0.203 and 64.1% average preference satisfaction.

**Key files:** `apps/api/src/config/solver.config.ts`, `apps/api/src/routes/solver2.ts`
**Tech:** Node.js `child_process.spawn()`, Python subprocess IPC, OR-Tools portfolio search
**Tags:** `[optimization]` `[backend]` `[systems]`

---

### Warm-Start Solution Hints

Accepts a previous solve result as initial variable hints to speed up re-solves after constraint changes. For each `(crewId, roleId, startMin)` triple in the hint set, calls `model.AddHint(var, 1)`; all other vars receive `AddHint(var, 0)`.

**Key files:** `apps/solver-python/logbook_solver_v2/solver_v2.py`
**Tech:** OR-Tools `model.AddHint()`
**Tags:** `[optimization]` `[backend]`

---

### Force-Soft Diagnosis Mode

Re-solves with all HARD constraints demoted to SOFT (high-weight penalties) to diagnose infeasible schedules. Reports which constraints were violated and by how much, enabling operators to identify and fix root-cause conflicts before re-running.

**Key files:** `apps/solver-python/logbook_solver_v2/solver_v2.py`
**Tech:** Python, OR-Tools, conditional constraint generation on `force_soft_mode=True`
**Tags:** `[optimization]` `[backend]` `[systems]`

---

### Quota Divisibility Pre-Validation

Pre-solve check that detects quotas where `requiredMin` is not divisible by `taskLength` (or `taskLength/2` for half-block roles) and where crew shift overlap with the quota window is insufficient. Violations are returned in `metadata.quotaDivisibilityErrors` without invoking the solver.

**Key files:** `apps/solver-python/logbook_solver_v2/constraints.py`
**Tech:** Python, arithmetic validation
**Tags:** `[optimization]` `[backend]`

---

## 2. API & Data Layer

---

### Fastify REST API

REST API exposing all scheduling operations across 14 route modules. RBAC middleware enforces a 4-tier role hierarchy (CREW < MATE < CAPTAIN < ADMIN) on all mutating endpoints. JWT stored in HTTP-only cookies. The solver endpoint spawns a Python subprocess, pipes JSON to stdin, and parses the result from stdout.

**Key files:** `apps/api/src/index.ts`, `apps/api/src/routes/`, `apps/api/src/middleware/rbac.ts`
**Tech:** Fastify, Prisma ORM, `@fastify/jwt`, `@fastify/cookie`, Node.js `child_process.spawn()`
**Tags:** `[backend]` `[systems]`

---

### Solver Input Builder

Assembles the full `SolverInputV2` payload from Postgres in a single parallel-fetch pass (11 concurrent Prisma queries). Resolves rule priority via a 4-tier cascade (crew priority > store priority > crew non-priority > store fallback). Handles shift overrides, per-role lookback windows for fairness history, and crew eligibility filtering.

**Key files:** `apps/api/src/solver2/builder.ts`
**Tech:** TypeScript, Prisma, `Promise.all`, PostgreSQL
**Tags:** `[backend]` `[data-modeling]` `[optimization]`

---

### Relational Data Model

PostgreSQL schema capturing the full scheduling domain: stores, crew, roles, shifts, coverage windows, quotas, logbooks, assignments, fairness history, preference satisfaction metadata, runs, activity logs, and invite codes. Key design decisions:

- Crew IDs are fixed-length `CHAR(7)` (e.g., `TCRW001`)
- All times stored as minutes-since-midnight integers `[0, 1440)`
- `@db.Date` columns for daily dates to prevent timezone drift
- `Logbook.status` enum (`DRAFT`/`PUBLISHED`/`SUPERSEDED`) with `supersededById` chain for history
- `Assignment.origin` enum (`ENGINE`/`MANUAL`/`ML_ADJUSTED`) for provenance tracking
- `LogPreferenceMetadata` JSON blob with full per-rule/per-crew/per-type satisfaction breakdowns

**Key files:** `apps/api/prisma/schema.prisma`
**Tech:** Prisma ORM, PostgreSQL, UUID, enum types, JSON columns
**Tags:** `[data-modeling]` `[backend]`

---

### Logbook Manager

Persists solver output atomically: builds `LogbookMetadata` JSON (solver stats, constraint counts, preference satisfaction summary), computes a 16-char `sha256` input hash for deduplication, upserts the `Logbook` record, deletes and recreates `Assignment` and `LogPreferenceMetadata` rows, creates a `Run` audit record, and triggers downstream fairness history updates — all within a single workflow.

**Key files:** `apps/api/src/services/logbook-manager.ts`
**Tech:** TypeScript, Prisma, `crypto.createHash('sha256')`, atomic multi-table writes
**Tags:** `[backend]` `[data-modeling]`

---

### Post-Solve Constraint Analyzer

Independent validation layer that re-checks solver output against all hard constraints without re-invoking the solver. Builds lookup maps (`crewMinutesByRole`, `crewAssignmentsByRole`) from the raw assignments, runs all constraint checks, and returns a categorized `ConstraintViolation[]` with severity levels (error/warning/info).

**Key files:** `apps/api/src/services/constraint-analyzer.ts`
**Tech:** TypeScript, pure in-process computation
**Tags:** `[backend]` `[optimization]`

---

### Auth & RBAC with Invite Codes

JWT-cookie authentication with a 4-tier role hierarchy and invite-code-based onboarding. Invite codes are 8-char alphanumeric (excluding ambiguous chars `I/O/0/1`), one-time-use, with enforced expiry. Store-scoped endpoints check `user.storeId` against the requested store unless the user is `ADMIN`.

**Key files:** `apps/api/src/routes/auth.ts`, `apps/api/src/config/auth.config.ts`
**Tech:** `@fastify/jwt`, `@fastify/cookie`, bcrypt, HTTP-only cookies, `sameSite: 'lax'`
**Tags:** `[backend]` `[systems]`

---

### Activity Log & Audit Trail

Full audit trail of user actions written to Postgres: 18 action types including `solver_run`, `logbook_publish`, `assignment_edit`, `comment`, `login`. Exposed as a filterable activity feed on the home page.

**Key files:** `apps/api/src/services/activity-logger.ts`
**Tech:** TypeScript, Prisma, `ActivityLog` table
**Tags:** `[backend]` `[data-modeling]`

---

### Shift Segmentation

Splits each crew shift into PRODUCT segments (outside the store's register window) and FLEX segments (inside register hours, allocated by the solver). Three intervals computed: left PRODUCT, FLEX, right PRODUCT. Used to pre-assign PRODUCT time before invoking the solver.

**Key files:** `apps/api/src/services/segmentation.ts`
**Tech:** TypeScript, pure arithmetic, minutes-since-midnight time representation
**Tags:** `[backend]` `[algorithms]`

---

## 3. Preference System

---

### Multi-Layer Preference Framework

Per-crew scheduling preferences encoded using the same 16-rule-type taxonomy as hard constraints, but flagged as `constraintType = SOFT`. Each `CrewRoleRule` carries an `isPriority` flag that controls whether it overrides store-level defaults for the same rule key. Preference types include: timing gradients (EARLY/MIDDLE/LATE), shift-relative hour bonuses (`LIKE/DISLIKE_ROLE_FOR_HOUR_X`), adjacency preferences (`CANNOT_BE_ASSIGNED_AFTER`), preferred block duration (`MAX_CONSECUTIVE_MINUTES`), role family balance (`DISTRIBUTION_BETWEEN_ROLE_X`), and role avoidance (`FORBID_ROLE`).

**Key files:** `apps/api/src/solver2/builder.ts`, `apps/solver-python/logbook_solver_v2/objective.py`
**Tech:** TypeScript, Python, PostgreSQL, Prisma
**Tags:** `[backend]` `[optimization]` `[data-modeling]`

---

### Preference Satisfaction Scoring

After each solve, scores how well each `CrewRoleRule` preference was satisfied, producing per-rule, per-crew, and per-rule-type breakdowns. Computes:

- `satisfaction: 0.0–1.0` per rule (type-specific scorer)
- `eligiblePreferences`: count of applicable rules (skips rules where crew didn't work or role wasn't assigned)
- `preferencesMet`: rules with satisfaction ≥ 0.5
- `avgSatisfaction`: mean over eligible (0–100 scale)
- `fairnessIndex`: Gini-based equality across per-crew satisfaction scores
- Letter grade (A+–F) on a scheduling-aware scale (A+ ≥ 94%, F < 28%)

JSON breakdowns stored in `LogPreferenceMetadata` for dashboard display.

**Key files:** `apps/api/src/services/crew-rule-satisfaction.ts`
**Tech:** TypeScript, Gini coefficient, letter grade mapping, Prisma
**Tags:** `[backend]` `[algorithms]` `[data-modeling]` `[ml-adjacent]`

---

### Preference Config & Weight Scaling

Centralized, env-var-driven config for the preference weighting strategy. Supports three scaling modes (linear/exponential/logarithmic), type-specific multipliers (first-hour 1.5×, break 1.2×), adaptive boosts for under-satisfied crew (1.3×), dampening for over-satisfied (0.7×), conflict rotation cycles, and preference banking with age-boost factor. All parameters are overridable at runtime without code changes.

**Key files:** `apps/api/src/config/preferences.ts`
**Tech:** TypeScript, environment variables (`BANKING_WEIGHT_DIVISOR`, `BANKING_MAX_WEIGHT_BOOST`, `BANKING_AGE_BOOST_FACTOR`, `BANKING_CARRYOVER_DAYS`)
**Tags:** `[backend]` `[ml-adjacent]` `[optimization]`

---

## 4. Fairness Engine

---

### Role-Level Fairness Tracking

Tracks daily `minutesAssigned` per crew per role in `CrewRoleFairnessHistory`. After each logbook save, computes Gini coefficient–based fairness metrics per tracked role and stores a `RoleFairnessSnapshot` (Gini coefficient, fairness index, letter grade, avg minutes/day, std deviation). Normalization uses `minutesPerDay = roleMinutes / daysWorked` so part-time crew aren't unfairly penalized.

**Key files:** `apps/api/src/services/role-fairness.service.ts`
**Tech:** TypeScript, Prisma, Gini coefficient (`O(n²)` pairwise: `gini = Σ|xi−xj| / (2n·Σxi)`)
**Tags:** `[backend]` `[algorithms]` `[data-modeling]`

---

### Tiered Rotation Boost (Intra-Schedule Fairness)

At solve time, computes `mph = (role_minutes / shift_minutes) × 60` for each eligible crew from historical assignment data, normalizes to `[0, 1]`, and adds a scaled boost `BASE_BOOST × normalized_mph` to each crew's assignment variables in the objective. Crew with the fewest historical minutes per hour worked receive the highest boost, naturally rotating underserved crew into preferred roles. `BASE_BOOST=740` was empirically tuned across a 31-day simulation achieving Gini=0.203 and stabilization by day 11.

**Key files:** `apps/solver-python/logbook_solver_v2/constraints.py` (`_intra_schedule_fairness_constraints`)
**Tech:** Python, OR-Tools objective terms, historical lookup from `fairnessHistory` + `shiftHistory` payloads
**Tags:** `[optimization]` `[algorithms]` `[ml-adjacent]`

---

### Intra-Day Distribution Penalty

Prevents the same crew member from receiving multiple blocks of a fairness-tracked role within a single day. For each tracked role and crew with >1 possible assignment, creates `extra_var ≥ sum(crew_vars) − 1` and subtracts `extra_var × 2000` from the objective. Spreads high-demand roles across eligible crew each day.

**Key files:** `apps/solver-python/logbook_solver_v2/constraints.py` (`_intra_day_distribution_penalty`)
**Tech:** Python, OR-Tools `NewIntVar`, objective penalty terms
**Tags:** `[optimization]` `[algorithms]`

---

### Z-Score Fairness Objective Terms

Symmetric fairness signal based on statistical deviation from the mean. Computes `z_score = (mph − mean_mph) / std_dev` per crew for tracked roles. Under-assigned crew (negative z) receive a reward proportional to `|z| × fairnessBoost × task_slots`; over-assigned crew receive a symmetric penalty. Only activates when `std_dev > fairnessMinStd`. Supports A/B testing via `skipFairnessWeights=true` on individual solve requests.

**Key files:** `apps/solver-python/logbook_solver_v2/objective.py` (`_fairness_objective_terms`)
**Tech:** Python, statistics, OR-Tools objective terms
**Tags:** `[optimization]` `[algorithms]` `[ml-adjacent]`

---

## 5. Frontend & UX

---

### Multi-Step Logbook Creation Wizard

4-step guided flow for MATE/CAPTAIN users: shifts entry → constraints setup → solver preview → publish. Each step is a full-page route component with a shared `ProgressBar` and AI glass card wrappers. Steps navigate programmatically; the preview step shows the solver result before the user commits to publishing.

**Key files:** `apps/web/app/stores/[storeId]/logbook/create/`
**Tech:** Next.js 13 App Router, React 18, Tailwind CSS, AI glass design system
**Tags:** `[frontend]` `[fullstack]`

---

### Fairness Dashboard

Rich analytics dashboard with three views (Overview, Roles, Crew) built from custom SVG chart components. Charts include:

- **SatisfactionLineGraph** — SVG line graph with seeded-random gradient fills and interactive hover/selection
- **RoleHeatmap** — week × day-of-week grid with opacity-scaled cells (theme-red, 0.2–0.85 range)
- **CrewFairnessTable** — sortable table of crew ranked by total/per-day minutes and fairness index
- **BoxPlotGraph**, **StackedPillBarGraph** — statistical distribution visualizations

Data comes from `GET /api/stores/:storeId/dashboard/logbooks` and is transformed by `buildDashboardSnapshot` before rendering.

**Key files:** `apps/web/app/stores/[storeId]/fairness-dashboard/`, `apps/web/src/dashboard/`
**Tech:** React 18, Next.js, SVG, Tailwind CSS, AI glass design system, Heroicons
**Tags:** `[frontend]` `[fullstack]` `[data-modeling]`

---

### Store Home Hub (Management Dashboard)

Multi-tab management hub with list views and inline CRUD forms for crew, roles, logbooks, and preferences, plus a real-time activity feed. 5 tabs (Activity, Crew, Logbooks, Roles, Preferences) backed by 8 extracted view components, a shared `useHomeData` hook, and a custom `DetailPanelContent` component that handles 10+ entity types. Role rules rendered via a display-name registry (`ROLE_RULE_TYPE_LABELS`). Logbooks list supports drag-select multi-delete. Auth via Zustand, route-level RBAC at the component level.

**Key files:** `apps/web/app/stores/[storeId]/home/`
**Tech:** React 18, Next.js App Router, Zustand, Headless UI, Heroicons, Tailwind CSS
**Tags:** `[frontend]` `[fullstack]`

---

### SPA Tab Navigation with History API

Three top-nav tabs (Home, System Health, Settings) switch without route navigation. A `StoreShell` component manages tab state; clicks call `history.pushState` so the URL updates and the browser back/forward buttons work, but no Next.js route transition fires — no unmount/remount, no scroll reset, no transition flash. A `popstate` listener restores the active tab on back/forward for shell-owned history entries; non-shell entries fall through to Next.js routing. Each route (`/home`, `/fairness-dashboard`, `/settings`) still exists as a thin server-component wrapper that passes `initialTab` to `StoreShell`, so deep links and direct navigation still work.

**Key files:** `apps/web/app/stores/[storeId]/components/StoreShell.tsx`
**Tech:** React 18, Next.js App Router, browser History API (`pushState`/`popstate`), TypeScript
**Tags:** `[frontend]` `[fullstack]`

---

### Dashboard Data Prefetch via React Query Cache

Eliminated ~10s perceived latency on the "System Health" tab by prefetching at the store layout level instead of on tab click. `useAvailableDates` and `useDashboardData` fire as side effects in `StoreLayout` (a Next.js layout that stays mounted across all store routes). When the user later opens the fairness dashboard, both hooks return from the React Query cache instantly — no network round-trip. Cache hit relies on React Query's deep-key equality: both the layout and the fairness page derive `datesToFetch` from the same `useAvailableDates` cache entry, guaranteeing identical query keys.

**Key files:** `apps/web/app/stores/[storeId]/layout.tsx`, `apps/web/lib/hooks/useDashboardData.ts`
**Tech:** React Query (`@tanstack/react-query`), Next.js App Router layout persistence, TypeScript
**Tags:** `[frontend]` `[fullstack]` `[systems]`

---

### AI Glass Design System

Custom design system of translucent card components with runtime-tunable CSS variables. Light and dark mode variants use `backdropFilter: blur(var(--glass-blur, 7px))` and configurable `--glass-bg-opacity`, `--border-opacity`, `--border-color`. Includes a `GlassDevPanel` for real-time CSS variable tweaking during development. Card hierarchy: `CardTiny` → `CardLarge`, `GlassPillCard`, `GlassPillButton`.

**Key files:** `apps/web/components/ui/ai-glass/`
**Tech:** React, TypeScript, CSS custom properties, `backdropFilter`, Tailwind CSS
**Tags:** `[frontend]`

---

### Route-Aware Tutorial System

Step-by-step overlay tutorial with route-aware navigation, target element highlighting, and positioned bubble components. Tutorial steps are authored as typed `TutorialStep[]` arrays with `target` (DOM selector), `route`, `onEnter` callback, and `bubble: { title, body, position }`. State managed in Zustand. Covers home, crew, and fairness dashboard pages with pre-seeded example data navigation.

**Key files:** `apps/web/app/tutorial/steps/`, `apps/web/lib/tutorialStore.ts`
**Tech:** React 18, Next.js `useRouter`, Zustand, TypeScript
**Tags:** `[frontend]`

---

## 6. Tooling & Ops

---

### Turborepo Monorepo

pnpm workspaces with a Turborepo pipeline DAG (`build` depends on `^build`, shared with `test` and `lint`). Shared TypeScript base config extended per package. Three packages: `packages/domain` (constraint logic + validators), `packages/shared-types` (interfaces shared across API and web), and two apps (`apps/api`, `apps/web`) plus the Python solver.

**Key files:** `turbo.json`, `pnpm-workspace.yaml`, `tsconfig.base.json`
**Tech:** Turborepo, pnpm, TypeScript
**Tags:** `[devops]` `[systems]`

---

### Domain Package (Shared Constraint Logic)

Isolated TypeScript package with constraint validators, date/shift normalizers, solver utilities, and preference scorers (firstHour, favorite, timing, consecutive). Decoupled from the API layer for unit testing without database dependencies.

**Key files:** `packages/domain/src/`
**Tech:** TypeScript, Vitest
**Tags:** `[backend]` `[algorithms]` `[devops]`

---

### Python Solver IPC Architecture

The Python solver is invoked as a subprocess by the Node.js API via stdin/stdout JSON — no HTTP overhead, no network boundary. Node.js locates the Python binary at `.venv/bin/python` (falls back to `python3`), spawns `python -m logbook_solver_v2.cli`, pipes the serialized `SolverInputV2`, and parses the stdout result. The tuning engine uses a separate module invocation.

**Key files:** `apps/api/src/routes/solver2.ts`, `apps/solver-python/logbook_solver_v2/cli.py`
**Tech:** Node.js `child_process.spawn()`, Python venv, OR-Tools, stdin/stdout IPC
**Tags:** `[systems]` `[backend]` `[devops]`

---

### Solver Parameter Tuning Infrastructure

Empirically validated solver configuration documented with experimental results. `TUNING_CONFIG` captures tested parameter tables (fairness boost sweep across 4 phases: coarse 10k→100, fine 1000→500, ultra-fine 750→700, ultra-fine 2). `PRODUCTION_SOLVER_SETTINGS` is the live config object merged into every solve request. `TRACKED_ROLE_IDS` maps business roles (PARKING_HELMS, WINE_DEMO, FOOD_DEMO) to database IDs. Supports `skipFairnessWeights=true` for A/B testing fairness vs. non-fairness runs.

**Key files:** `apps/api/src/config/solver.config.ts`
**Tech:** TypeScript constants, environment variable overrides
**Tags:** `[optimization]` `[ml-adjacent]` `[devops]`

---

### ML Integration Scaffolding

Scaffolding for a machine-learning schedule tuning engine. Includes `solver_training_data.jsonl` (training examples), `apps/ml/` directory, `pnpm weights:auto-tune` / `pnpm weights:mixed-mode:apply` scripts for automated weight optimization, and `AssignmentOrigin.ML_ADJUSTED` enum value for provenance tracking of ML-generated assignments.

**Key files:** `apps/api/scripts/`, `apps/ml/`, `apps/api/prisma/schema.prisma` (`AssignmentOrigin`)
**Tech:** Node.js scripts, JSONL, Prisma enums
**Tags:** `[ml-adjacent]` `[devops]` `[backend]`

---

### Test Infrastructure

Integration tests covering the full solver flow (`solver.integration.test.ts`, `e2e.api.test.ts`) and unit tests for individual constraint scorers (`packages/domain/test/constraints/`), CRUD operations, preference calculations, segmentation, and coverage wizard. Python unit tests for specific constraint types. Test stores use self-cleanup utilities (`cleanup-test-stores.ts`) so they don't pollute production data.

**Key files:** `apps/api/test/`, `packages/domain/test/`, `apps/solver-python/logbook_solver_v2/test_cannot_assign_during_hour.py`
**Tech:** Vitest, Python `unittest`, OR-Tools test utilities
**Tags:** `[backend]` `[devops]`

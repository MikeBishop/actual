# Schedule Forecast Smoothing — Design

## Goal

When a category has multiple schedule-type Budget Automation (Budget
Templates) rules, today's engine can hold significantly more balance in
the category than is ever actually required at one time, even though
it arrives at a technically-correct monthly average. This introduces a
new, more accurate smoothing algorithm — a 60-month forward projection
with an iterative correction — as an alternative to today's
payMonthOf/sinking heuristic, gated behind a build-time switch so it
can be perf-tested before deciding rollout scope.

## Background

Current implementation (`packages/loot-core/src/server/budget/schedule-template.ts`,
`runSchedule` + `createScheduleList`) buckets each schedule template
into "due this month" (`payMonthOf`, paid in full) vs "future"
(`sinking`, smoothed via a per-schedule monthly-base-contribution
heuristic that doesn't look at the category's actual projected
balance trajectory). Multiple sinking schedules are summed
independently rather than jointly optimized against the category's
real balance curve, which is what causes overholding.

"Budget Templates" and "Budget Automation" are the same feature/code
path — no fork exists (confirmed by exploring
`goal-template.pegjs`, `category-template-context.ts`, and
`schedule-template.ts`).

## Non-Goals

- No user-facing toggle or per-category setting in this iteration —
  gating is a build-time constant, to be promoted to a real setting
  (or made the unconditional default) only after perf data.
- No change to non-schedule template types (`simple`, `refill`,
  `copy`, `periodic`, `spend`, `percentage`, `by`, `average`).
- No change to `full`-flagged schedule behavior — it stays exactly as
  today (paid in full when due, unconditionally added on top).
- No faster-convergence optimization (minimum-increment stepping,
  local-minima walking) — noted as a possible future improvement, not
  in scope now.

## Data Assembly

**Inputs** (mirrors `runSchedule`'s existing call site in
`category-template-context.ts:209-231`, same values already computed
there):
- `template_lines: Template[]` — all templates at this priority level.
- `current_month: string`
- `startingBalance = fromLastMonth + toBudgetSoFar` — the exact
  expression the engine already passes as `balance` today. No new
  balance-reservation logic for other automation types: other
  template types' contributions already pool into this same shared
  category balance (all templates fund one shared bucket), and the
  existing "current month only" reservation via `toBudgetSoFar`
  (contributions from earlier-processed sibling templates in this
  same priority pass) is preserved unchanged. Sibling automations in
  other priority passes, or later in the same priority's template
  list, are not modeled — not forecastable without hypothetically
  re-running the whole engine for future months.
- `last_month_balance: number`, `to_budget: number`, `errors: string[]`,
  `category: CategoryEntity`, `currency: Currency` — passed through as
  today.

**Step 1 — split schedule templates by `full` flag.**
```
fullSchedules = scheduleTemplates.filter(t => t.full === true)
smoothSchedules = scheduleTemplates.filter(t => t.full !== true)
```
`fullSchedules` are computed via the existing `getPayMonthOfTotal`-style
logic (their target amount, when due), added straight into `to_budget`
— on top of, and outside, the new forecast. Their contribution is
subtracted from the balance pool available to the forecast:
```
forecastStartingBalance = startingBalance - fullContribution
```

**Step 2 — new shared occurrence-enumeration helper.**
No existing exported utility enumerates "all occurrences of a
recurrence rule between date A and date B." The closest matches
(`getUpcomingDates` in `server/schedules/app.ts:474-493`, `takeDates`
in `server/schedules/find-schedules.ts:16-22`) are server-private and
count-bounded, not range-bounded. `computeSchedulePreviewTransactions`
in `shared/schedules.ts:436-516` already hand-rolls the needed loop
(repeatedly calling `getNextDate`, advancing past each hit) for the
schedules list UI.

Extract that loop into a new exported function in
`packages/loot-core/src/shared/schedules.ts`:
```ts
export function getOccurrencesBetween(
  dateCond: ScheduleConditionEntity, // shape used by extractScheduleConds' `date` output
  start: string,  // YYYY-MM-DD
  end: string,    // YYYY-MM-DD, exclusive
): string[]
```
Update `computeSchedulePreviewTransactions` to call this new function
instead of duplicating the loop (removes duplicate logic; no behavior
change intended — add a regression test).

**Step 3 — build the 60-month outflow array.**
For each smooth schedule:
1. Compute its per-occurrence amount **once**, reusing
   `createScheduleList`'s existing rule-execution/adjustment/sign
   logic evaluated against its *next* occurrence (same computation
   `createScheduleList` does today for `target`). Treat this as a
   constant amount applied to every enumerated occurrence within the
   window — a BALANCE_OF-style dynamic amount can't be meaningfully
   forecast 60 months out, so freezing at the next-occurrence value is
   the accepted simplification.
2. Enumerate occurrence dates via
   `getOccurrencesBetween(dateCond, current_month_start, current_month_start + 60 months)`.
3. Bucket each occurrence into a month index `0..59` (`0` = current
   month) by calendar month difference from `current_month`, summing
   amounts per bucket into `monthlyOutflow: number[60]`.

Unlinked current/future-dated transactions already assigned to the
category (i.e. not linked to any schedule) are added into
`monthlyOutflow` at their own dated month's index, as one-time amounts
— not treated as recurring, not projected past their own dated month.
Only current-month-or-later transactions qualify (no window-bound past
transactions exist by construction).

**Perf note:** daily/weekly schedules can produce hundreds to
thousands of occurrences across a 60-month window. This cost is
exactly what the gating gives room to measure before deciding rollout
scope (see Gating below) — flagged here, not resolved in this design.

## Iteration Algorithm

Operates only on the collapsed `monthlyOutflow[0..59]` array — no
further per-occurrence work.

```
totalOutflow = sum(monthlyOutflow[0..59])
candidate = round(totalOutflow / 60)      // initial guess, nearest cent
CAP = 60

for cycle in 0..CAP-1:
  balance[-1] = forecastStartingBalance
  for i in 0..59:
    balance[i] = balance[i-1] + candidate - monthlyOutflow[i]

  if all(balance[i] >= 0 for i in 0..59):
    k = index of the lowest balance[i] (soonest index on ties)
    surplus = balance[k]
    candidate -= floor(surplus / (k+1))    // whole cents; single adjustment, no re-check
    return candidate                        // converged

  k = soonest i where balance[i] < 0
  shortfall = -balance[k]
  candidate += ceil(shortfall / (k+1))      // whole cents
  // continue loop with the new candidate

// CAP exceeded (pathological case, e.g. a schedule combination that only
// ever needs a $0.01/cycle correction): return the last computed candidate,
// not the naive initial average.
return candidate
```

Notes:
- Month index `i`'s balance includes month `i`'s own candidate
  contribution before subtracting month `i`'s outflow (fund-then-spend
  ordering) — this is what makes a same-month-due smooth schedule get
  covered immediately rather than incorrectly smoothed away.
- Convergence: raising `candidate` only ever raises every
  `balance[i]` (monotonic in `candidate`), so each shortfall-cycle
  permanently zeroes the *soonest* violation without introducing an
  earlier one. Worst case is one new violation surfacing per cycle,
  bounding cycles at 60 — `CAP = 60` is the true worst case, not a
  heuristic margin.
- The surplus branch performs its single adjustment and returns
  immediately — no re-check loop, even though in principle the
  adjustment could reveal a different soonest-lowest point.
- A future optimization (not in scope): detect a slow-converging case
  (e.g. tiny per-cycle correction) and take a larger step, or walk
  toward the local minimum directly instead of one violation at a
  time.

If `smoothSchedules` is empty, skip the forecast entirely
(`candidate = 0`).

**Output:** `to_budget += fullContribution + candidate`.

## Integration

**New function**, same file, alongside the existing one (existing
`runSchedule` is untouched, kept as the fallback/rollback path):

```ts
export async function runScheduleForecast(
  template_lines: Template[],
  current_month: string,
  balance: number,
  last_month_balance: number,
  to_budget: number,
  errors: string[],
  category: CategoryEntity,
  currency: Currency,
): Promise<{
  to_budget: number;
  errors: string[];
  perScheduleMonthly: Map<ScheduleTemplate, number>;
}>
```
Drops the `remainder` parameter `runSchedule` takes — that's
sinking-specific carried-forward state; the forecast re-derives
everything fresh from `monthlyOutflow` each cycle and has no use for
it.

**Gating** (`category-template-context.ts`, `case 'schedule':`
branch): a module-level constant, e.g.
```ts
const USE_SCHEDULE_FORECAST = false;
```
selects `runScheduleForecast` vs today's `runSchedule`. Not a
user-facing or per-category setting yet.

**Per-template UI attribution.** `redistributeBatch`
(`category-template-context.ts:997-1040`) still needs a weight per
sibling schedule template to split the batch back out for per-row
display (cosmetic only — doesn't affect the real budgeted total).
Weight = each smooth schedule's own `totalOutflow/60` share, scaled
against the final `candidate`; full-flag schedules keep their existing
exact-amount weight. Same `schedulePerTemplate` map shape returned
today; no changes needed at the `redistributeBatch` call site itself.

## Testing

Permanent tests assert the new algorithm's own correctness in absolute
terms — never by comparing its output against `runSchedule`'s. That
comparison is only meaningful while both algorithms coexist; once the
old heuristic is removed (or if it never is, but is deprioritized) a
test asserting "forecast beats old algorithm" has nothing left to
mean. The "this fixes overholding" claim is demonstrated once, by hand
or in a throwaway script, when the feature is proposed for wider
rollout — not carried in the suite.

In `packages/loot-core/src/server/budget/schedule-template.test.ts`:
- A single smooth schedule produces a `candidate` equal to its own
  monthly-equivalent amount (the degenerate case has one obviously
  correct answer, independent of what the old algorithm does).
- Two smooth schedules: assert the projected month-end balance never
  goes negative anywhere in the 60-month window, and that the returned
  `candidate` is the minimal contribution satisfying that (i.e.
  raising it by any smaller increment than the algorithm's own
  adjustment step would have left some month negative, or the surplus
  step didn't over-reduce it either) — an absolute correctness
  assertion on the iteration's own postcondition, not a comparison to
  another algorithm.
- A same-month-due smooth schedule forces immediate full coverage that
  month (fund-then-spend ordering check).
- A `full`-flagged schedule stays separate/on top of the forecast,
  unaffected by smoothing.
- An unlinked future-dated transaction in the category reduces the
  forecasted candidate need in its own month, with no effect on other
  months.
- Convergence terminates within a small number of cycles for realistic
  multi-schedule inputs.
- `CAP` fallback path: a contrived pathological case (e.g. engineered
  $0.01/cycle correction) returns the last computed candidate rather
  than the naive initial average.

In `packages/loot-core/src/shared/` (co-located with
`schedules.ts`'s existing tests): a regression test confirming
`computeSchedulePreviewTransactions` produces identical output before
and after being refactored to call the new
`getOccurrencesBetween`.

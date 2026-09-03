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
- No iterative-approximation solver — the minimum safe contribution
  has a closed form (see Iteration Algorithm below), so there is
  nothing left to optimize the convergence of.

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
  start: string, // YYYY-MM-DD
  end: string, // YYYY-MM-DD, exclusive
): string[];
```

Update `computeSchedulePreviewTransactions` to call this new function
instead of duplicating the loop (removes duplicate logic; no behavior
change intended — add a regression test).

**Step 3 — build the 60-month outflow array.**
For each smooth schedule:

1. Compute its per-occurrence amount **once**, reusing
   `createScheduleList`'s existing rule-execution/adjustment/sign
   logic evaluated against its _next_ occurrence (same computation
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
further per-occurrence work. Despite the section title (kept for
continuity with the rest of this doc and the plan), this is a closed
form, not an iterative approximation — see Derivation below for why.

```
cumsum[i] = sum(monthlyOutflow[0..i])              // for i in 0..59

candidate = 0
for i in 0..59:
  threshold_i = ceil((cumsum[i] - forecastStartingBalance) / (i + 1))
  candidate = max(candidate, threshold_i)

return candidate
```

`threshold_i` is the exact whole-cent contribution that makes month
`i`'s projected balance come out to exactly zero, in isolation. The
final `candidate` — the largest of the 60 thresholds — is
simultaneously the smallest contribution that keeps every month's
balance non-negative and the smallest that keeps any month's balance
at zero without going negative: raising it further past this point
only adds unspent surplus, and any smaller value leaves the
argmax month negative. `candidate = max(0, ...)` for `smoothSchedules`
empty (all `monthlyOutflow[i] = 0`, all thresholds `<= 0`, loop is a
no-op past the `candidate = 0` initialization).

- Month index `i`'s balance (per the recurrence below) includes month
  `i`'s own candidate contribution before subtracting month `i`'s
  outflow (fund-then-spend ordering) — this is what makes a
  same-month-due smooth schedule get covered immediately rather than
  incorrectly smoothed away. This ordering is already baked into
  `threshold_i`'s `(i + 1)` divisor (month `i` receives `i + 1`
  contributions — months `0..i` — by the time its outflow is due).
- Ceiling each `threshold_i` individually, before taking the max,
  keeps every one of the 60 months at or above break-even under
  integer-cent rounding — rounding only the final max could leave the
  argmax month a cent short if a different month's un-rounded
  threshold happened to be higher pre-rounding but lower post-rounding.

### Derivation (why no iteration is needed)

Unrolling the balance recurrence `balance[i] = balance[i-1] +
candidate - monthlyOutflow[i]`, `balance[-1] = forecastStartingBalance`:

```
balance[i] = forecastStartingBalance + (i + 1) * candidate - cumsum[i]
```

Each month's balance is affine in `candidate`, and requiring
`balance[i] >= 0` for all `i` is equivalent to requiring `candidate >=
threshold_i` for all `i`, where `threshold_i = (cumsum[i] -
forecastStartingBalance) / (i + 1)`. Critically, `threshold_i` does
not depend on `candidate` — it is a fixed number derived purely from
the outflow data — so the minimal candidate satisfying every
constraint is simply `max_i(threshold_i)`, computable directly in one
pass with no guess-and-correct loop.

An earlier version of this design used a guess-and-correct loop that
repeatedly adjusted `candidate` based on `balance[i]` (the constraint
violation _scaled by_ `(i + 1)`, i.e. `(i + 1) * (candidate -
threshold_i)`) rather than the unscaled `threshold_i` itself. Because
that scaling factor grows with `i`, a month with a small threshold gap
late in the window can have a larger raw `balance[i]` than a month
with a much larger gap early in the window — so picking the extremum
of `balance[i]` (whether the earliest violation on the shortfall side,
or the smallest surplus on the correction side) does not reliably
identify `argmax(threshold_i)`. Worked failure case: months with
`threshold_0 = 100` (weight 1) and `threshold_59 = 149` (weight 60) at
a shared candidate of `150` produce `balance_0 = 50` and `balance_59 =
60` — picking the smaller raw balance (`50`, month 0) and reducing
`candidate` to `threshold_0 = 100` leaves month 59, whose actual
threshold (`149`) was never inspected, at `balance_59 = 60*100 -
60*149 = -2940`. Operating on `threshold_i` directly, as this design
now does, has no such failure mode: every threshold is computed
independently of the others and of any guess.

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
}>;
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
  goes negative anywhere in the 60-month window, and that `candidate -
1` (one cent less) _does_ leave some month negative — an absolute
  correctness assertion that the returned value is the minimal
  contribution satisfying the postcondition, not a comparison to
  another algorithm.
- A same-month-due smooth schedule forces immediate full coverage that
  month (fund-then-spend ordering check).
- A `full`-flagged schedule stays separate/on top of the forecast,
  unaffected by smoothing.
- An unlinked future-dated transaction in the category reduces the
  forecasted candidate need in its own month, with no effect on other
  months.
- A late month with a small threshold gap and an early month with a
  large one (the derivation's worked failure case, mirrored as a
  test): assert `candidate` equals the early month's threshold, not
  the late month's, and that every month's projected balance is
  non-negative — this is the regression test for the weighting bug the
  closed form replaces.

In `packages/loot-core/src/shared/` (co-located with
`schedules.ts`'s existing tests): a regression test confirming
`computeSchedulePreviewTransactions` produces identical output before
and after being refactored to call the new
`getOccurrencesBetween`.

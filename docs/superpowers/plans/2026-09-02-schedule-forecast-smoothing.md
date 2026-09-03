# Schedule Forecast Smoothing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a 60-month-projection-based smoothing algorithm
(`runScheduleForecast`) as a gated alternative to today's
payMonthOf/sinking heuristic (`runSchedule`) for schedule-type Budget
Automation templates, so a category with multiple schedules doesn't
overhold balance beyond what's actually needed.

**Architecture:** New code lives alongside the existing function in
`packages/loot-core/src/server/budget/schedule-template.ts`; nothing
existing is modified except a small type extension to carry
`dateConditions` through `createScheduleList`'s per-schedule entries,
and a one-line refactor extracting an occurrence-enumeration loop out
of `packages/loot-core/src/shared/schedules.ts` into a reusable
function. Old `runSchedule` is untouched and stays the default; the
new path is selected by a module-level constant in
`category-template-context.ts`.

**Tech Stack:** TypeScript, Vitest, existing `rschedule`-backed date
utilities in `shared/schedules.ts` and `shared/months.ts`, AQL query
builder (`#shared/query`, `#server/aql`).

**Spec:** `docs/superpowers/specs/2026-09-02-schedule-forecast-smoothing-design.md`

## Global Constraints

- No user-facing setting yet — selection between old/new algorithm is
  a build-time constant (`USE_SCHEDULE_FORECAST` in
  `category-template-context.ts`), defaulting to `false`.
- `full`-flagged schedule templates keep today's exact behavior
  unchanged (paid in full when due, added on top, never smoothed).
- All money amounts are integer cents throughout (existing convention
  in this codebase — no floating-point cent math).
- New tests assert the new algorithm's own postconditions (no negative
  projected month-end balance, minimal contribution) — never by
  comparing output against `runSchedule`'s.
- Run `yarn typecheck` and `yarn workspace @actual-app/core run test`
  (or `yarn test` from root) after each task; both must pass before
  committing.

---

### Task 1: Extract `getOccurrencesBetween` shared occurrence helper

**Files:**
- Modify: `packages/loot-core/src/shared/schedules.ts:436-516` (`computeSchedulePreviewTransactions`)
- Test: `packages/loot-core/src/shared/schedules.test.ts`

**Interfaces:**
- Produces: `export function getOccurrencesBetween(dateCond, startDay: string, endDay: string): string[]` — every occurrence date (`'YYYY-MM-DD'` strings) of the recurrence rule described by `dateCond` (the `{ op, field: 'date', value }` shape returned by `extractScheduleConds(...).date`) that falls in `[startDay, endDay)`. Non-recurring `dateCond` (a plain date, not a recur rule) returns `[]` if its date is outside the range, or `[date]` if inside — mirrors how `computeSchedulePreviewTransactions` seeds `dates` with `schedule.next_date` before iterating.
- Consumes: `getNextDate`, `scheduleIsRecurring`, `monthUtils` (already imported in `schedules.ts`).

- [ ] **Step 1: Write the failing test**

Add to `packages/loot-core/src/shared/schedules.test.ts` (check the file's existing imports/mocking pattern first — it likely already builds `dateCond`-shaped fixtures for `getNextDate` tests; reuse that pattern):

```ts
import { getOccurrencesBetween } from './schedules';

describe('getOccurrencesBetween', () => {
  it('returns every monthly occurrence within the range', () => {
    const dateCond = {
      op: 'is' as const,
      field: 'date' as const,
      value: {
        start: '2024-01-15',
        interval: 1,
        frequency: 'monthly' as const,
        patterns: [],
        skipWeekend: false,
        weekendSolveMode: 'before' as const,
        endMode: 'never' as const,
        endOccurrences: 1,
        endDate: '2099-01-01',
      },
    };
    const result = getOccurrencesBetween(dateCond, '2024-01-01', '2024-04-01');
    expect(result).toEqual(['2024-01-15', '2024-02-15', '2024-03-15']);
  });

  it('returns an empty array when the single non-repeating date is outside the range', () => {
    const dateCond = { op: 'is' as const, field: 'date' as const, value: '2023-06-01' };
    expect(getOccurrencesBetween(dateCond, '2024-01-01', '2024-04-01')).toEqual([]);
  });

  it('returns the single date when a non-repeating date is inside the range', () => {
    const dateCond = { op: 'is' as const, field: 'date' as const, value: '2024-02-01' };
    expect(getOccurrencesBetween(dateCond, '2024-01-01', '2024-04-01')).toEqual(['2024-02-01']);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @actual-app/core run test schedules.test.ts`
Expected: FAIL — `getOccurrencesBetween` is not exported yet.

- [ ] **Step 3: Implement `getOccurrencesBetween` and refactor `computeSchedulePreviewTransactions` to use it**

In `packages/loot-core/src/shared/schedules.ts`, add the new function near `computeSchedulePreviewTransactions` (after `scheduleIsRecurring`, before `isForPreview`):

```ts
export function getOccurrencesBetween(
  dateCond,
  startDay: string,
  endDay: string,
): string[] {
  const isRecurring = scheduleIsRecurring(dateCond);
  const rangeStart = d.startOfDay(monthUtils.parseDate(startDay));
  const rangeEnd = d.startOfDay(monthUtils.parseDate(endDay));

  if (!isRecurring) {
    const singleDate =
      typeof dateCond.value === 'string' ? dateCond.value : null;
    if (
      singleDate &&
      singleDate >= startDay &&
      singleDate < endDay
    ) {
      return [singleDate];
    }
    return [];
  }

  const dates: string[] = [];
  let day = rangeStart;
  while (day < rangeEnd) {
    const nextDate = getNextDate(dateCond, day);
    if (nextDate === null) break;
    if (d.startOfDay(monthUtils.parseDate(nextDate)) >= rangeEnd) break;
    if (nextDate >= startDay) dates.push(nextDate);
    day = d.startOfDay(monthUtils.parseDate(monthUtils.addDays(nextDate, 1)));
  }
  return dates;
}
```

Then replace the hand-rolled loop in `computeSchedulePreviewTransactions` (lines ~465-493) — the part building `dates` for recurring schedules — with a call to the new helper, preserving the existing `dates = [schedule.next_date]` seed and `dates.includes(nextDate)` dedup behavior:

```ts
      const dates = [schedule.next_date];
      if (isRecurring) {
        const occurrences = getOccurrencesBetween(
          dateConditions,
          monthUtils.dayFromDate(day),
          monthUtils.dayFromDate(upcomingPeriodEnd),
        );
        for (const occurrence of occurrences) {
          if (!dates.includes(occurrence)) dates.push(occurrence);
        }
      }
```

(Remove the now-unused `day` mutation loop and the `nextDate`/`break` logic that previously lived inline — `getOccurrencesBetween` owns that now.)

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @actual-app/core run test schedules.test.ts`
Expected: PASS, including all pre-existing `computeSchedulePreviewTransactions` tests (confirms the refactor is behavior-preserving).

- [ ] **Step 5: Commit**

```bash
git add packages/loot-core/src/shared/schedules.ts packages/loot-core/src/shared/schedules.test.ts
git commit -m "$(cat <<'EOF'
[AI] refactor: extract getOccurrencesBetween from schedule preview logic

computeSchedulePreviewTransactions hand-rolled a loop enumerating a
schedule's occurrences between two dates. Extract it into a reusable,
exported function so the upcoming schedule-forecast budgeting feature
can enumerate occurrences the same way.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 2: Carry `dateConditions` through `createScheduleList`'s per-schedule entries

**Files:**
- Modify: `packages/loot-core/src/server/budget/schedule-template.ts:20-31` (`ScheduleTemplateTarget` type), `:150-162` (push site)
- Test: `packages/loot-core/src/server/budget/schedule-template.test.ts`

**Interfaces:**
- Produces: `ScheduleTemplateTarget` gains a `dateConditions` field (the same object already computed at `schedule-template.ts:60` as `const { date: dateConditions, ... } = extractScheduleConds(conditions)`), so `runScheduleForecast` (Task 4) can enumerate every future occurrence per schedule without re-deriving it.
- Consumes: nothing new — `dateConditions` is already in scope at the push site.

- [ ] **Step 1: Write the failing test**

Add to `packages/loot-core/src/server/budget/schedule-template.test.ts`. `createScheduleList` isn't exported today — export it (it's currently a module-private `async function` in `schedule-template.ts`; add `export` to its declaration) so it's directly testable:

```ts
import { createScheduleList, runSchedule } from './schedule-template';

describe('createScheduleList', () => {
  beforeEach(() => {
    vi.clearAllMocks();
    vi.mocked(db.getAccounts).mockResolvedValue([]);
  });

  it('includes dateConditions on each returned entry', async () => {
    mockSingleSchedule({
      start: '2024-08-01',
      amount: -10000,
      frequency: 'monthly',
    });
    const template = {
      type: 'schedule',
      name: 'Test Schedule',
      priority: 0,
      directive: 'template',
    } as const;

    const { t } = await createScheduleList(
      [template],
      '2024-08-01',
      defaultCategory,
      defaultCurrency,
    );

    expect(t[0].dateConditions).toBeDefined();
    expect(t[0].dateConditions.value.frequency).toBe('monthly');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @actual-app/core run test schedule-template.test.ts -t "includes dateConditions"`
Expected: FAIL — `createScheduleList` not exported, or `t[0].dateConditions` is `undefined`.

- [ ] **Step 3: Implement**

In `packages/loot-core/src/server/budget/schedule-template.ts`:

1. Add `export` to `createScheduleList`'s declaration (line 33): `export async function createScheduleList(...)`.
2. Add the field to the type (line 20-31):

```ts
type ScheduleTemplateTarget = {
  template: ScheduleTemplate;
  name: string;
  target: number;
  next_date_string: string;
  target_interval: number;
  target_frequency: string | undefined;
  num_months: number;
  completed: number;
  full: boolean;
  repeat: boolean;
  dateConditions: ReturnType<typeof extractScheduleConds>['date'];
};
```
(Import `extractScheduleConds`'s return type is already imported as a function at the top of the file — this uses `ReturnType` so no new import is needed.)

3. At the push site (~line 150), add `dateConditions` to the pushed object:

```ts
      t.push({
        template,
        target,
        next_date_string,
        target_interval,
        target_frequency,
        num_months,
        completed,
        full: template.full === null ? false : template.full,
        repeat: isRepeating,
        name: displayName,
        dateConditions,
      });
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @actual-app/core run test schedule-template.test.ts`
Expected: PASS — new test passes, and all existing `runSchedule` tests still pass unchanged (this is a pure additive field).

- [ ] **Step 5: Commit**

```bash
git add packages/loot-core/src/server/budget/schedule-template.ts packages/loot-core/src/server/budget/schedule-template.test.ts
git commit -m "$(cat <<'EOF'
[AI] refactor: carry dateConditions through createScheduleList entries

The upcoming schedule-forecast algorithm needs each schedule's raw
recurrence rule to enumerate every future occurrence, not just the
next one. Export createScheduleList and add dateConditions to its
per-schedule output rather than re-deriving it from the schedule rule
a second time.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 3: Build the 60-month `monthlyOutflow` array

**Files:**
- Modify: `packages/loot-core/src/server/budget/schedule-template.ts` (new function, added after `createScheduleList`)
- Test: `packages/loot-core/src/server/budget/schedule-template.test.ts`

**Interfaces:**
- Consumes: `ScheduleTemplateTarget[]` (from Task 2's extended `createScheduleList`), `getOccurrencesBetween` (Task 1), `monthUtils`, `aqlQuery`/`q` (new imports needed in `schedule-template.ts`: `import { aqlQuery } from '#server/aql';` and `import { q } from '#shared/query';`).
- Produces: `async function buildMonthlyOutflow(smoothEntries: ScheduleTemplateTarget[], current_month: string, category: CategoryEntity): Promise<number[]>` — a 60-length array, index `0` = `current_month`, each value an integer-cent "net amount needed this month" (schedule occurrences' `target`, which is always positive, plus any unlinked category transactions' signed contribution). An unlinked transaction can be an inflow (e.g. a refund posted to an expense category) rather than an outflow, so an individual month's value is *usually* positive but is not guaranteed to be — the iteration in Task 4 must handle negative entries correctly (they only ever help the projected balance, never hurt it).

- [ ] **Step 1: Write the failing test**

```ts
describe('buildMonthlyOutflow', () => {
  beforeEach(() => {
    vi.clearAllMocks();
    vi.mocked(db.getAccounts).mockResolvedValue([]);
  });

  it('buckets each monthly schedule occurrence into its own month index', async () => {
    mockSingleSchedule({
      start: '2024-01-15',
      amount: -10000,
      frequency: 'monthly',
    });
    const template = {
      type: 'schedule',
      name: 'Rent',
      priority: 0,
      directive: 'template',
    } as const;
    const { t } = await createScheduleList(
      [template],
      '2024-01-01',
      defaultCategory,
      defaultCurrency,
    );

    vi.mocked(aqlQuery).mockResolvedValue({ data: [] } as never);

    const outflow = await buildMonthlyOutflow(t, '2024-01-01', defaultCategory);

    expect(outflow).toHaveLength(60);
    expect(outflow[0]).toBe(10000);
    expect(outflow[1]).toBe(10000);
    expect(outflow[59]).toBe(10000);
  });

  it('adds an unlinked future-dated transaction only to its own month', async () => {
    mockSingleSchedule({
      start: '2024-01-15',
      amount: -10000,
      frequency: 'monthly',
    });
    const template = {
      type: 'schedule',
      name: 'Rent',
      priority: 0,
      directive: 'template',
    } as const;
    const { t } = await createScheduleList(
      [template],
      '2024-01-01',
      defaultCategory,
      defaultCurrency,
    );

    vi.mocked(aqlQuery).mockResolvedValue({
      data: [{ amount: -5000, date: '2024-03-10' }],
    } as never);

    const outflow = await buildMonthlyOutflow(t, '2024-01-01', defaultCategory);

    expect(outflow[0]).toBe(10000);
    expect(outflow[2]).toBe(15000); // March: 10000 schedule + 5000 unlinked
    expect(outflow[3]).toBe(10000);
  });
});
```

Add `vi.mock('#server/aql');` near the top of the test file alongside the existing `vi.mock('#server/db')` etc., and import `aqlQuery` from `#server/aql`.

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @actual-app/core run test schedule-template.test.ts -t "buildMonthlyOutflow"`
Expected: FAIL — `buildMonthlyOutflow` doesn't exist.

- [ ] **Step 3: Implement**

Add near the top of `packages/loot-core/src/server/budget/schedule-template.ts`:
```ts
import { aqlQuery } from '#server/aql';
import { q } from '#shared/query';
import { getOccurrencesBetween } from '#shared/schedules';
```

Add the function after `createScheduleList`:

```ts
const FORECAST_MONTHS = 60;

export async function buildMonthlyOutflow(
  smoothEntries: ScheduleTemplateTarget[],
  current_month: string,
  category: CategoryEntity,
): Promise<number[]> {
  const monthlyOutflow = new Array(FORECAST_MONTHS).fill(0);
  const windowEnd = `${monthUtils.addMonths(current_month, FORECAST_MONTHS)}-01`;

  for (const entry of smoothEntries) {
    const occurrences = getOccurrencesBetween(
      entry.dateConditions,
      current_month,
      windowEnd,
    );
    for (const occurrenceDate of occurrences) {
      const monthIndex = monthUtils.differenceInCalendarMonths(
        occurrenceDate,
        current_month,
      );
      if (monthIndex >= 0 && monthIndex < FORECAST_MONTHS) {
        monthlyOutflow[monthIndex] += entry.target;
      }
    }
  }

  const { data: unlinkedTransactions } = await aqlQuery(
    q('transactions')
      .filter({
        category: category.id,
        schedule: null,
        date: { $gte: current_month, $lt: windowEnd },
      })
      .select(['amount', 'date']),
  );

  const sign = category.is_income ? 1 : -1;
  for (const transaction of unlinkedTransactions) {
    const monthIndex = monthUtils.differenceInCalendarMonths(
      transaction.date,
      current_month,
    );
    if (monthIndex >= 0 && monthIndex < FORECAST_MONTHS) {
      monthlyOutflow[monthIndex] += sign * transaction.amount;
    }
  }

  return monthlyOutflow;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @actual-app/core run test schedule-template.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/loot-core/src/server/budget/schedule-template.ts packages/loot-core/src/server/budget/schedule-template.test.ts
git commit -m "$(cat <<'EOF'
[AI] feat: build a 60-month per-category outflow array for schedule forecasting

Enumerates each smoothed schedule's occurrences and any current/future
transactions already assigned to the category but not linked to a
schedule, collapsing them into a single monthly array. This is the
input the forecast-smoothing iteration (next commit) runs against.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 4: Implement `runScheduleForecast`

**Files:**
- Modify: `packages/loot-core/src/server/budget/schedule-template.ts` (new function, added after `runSchedule`)
- Test: `packages/loot-core/src/server/budget/schedule-template.test.ts`

**Interfaces:**
- Consumes: `createScheduleList` (Task 2), `buildMonthlyOutflow` (Task 3).
- Produces: a pure, synchronous helper `iterateMonthlyContribution(startingBalance: number, monthlyOutflow: number[], cap = FORECAST_ITERATION_CAP): number` — separated out from `runScheduleForecast` specifically so the cap-exceeded branch can be tested directly against a hand-built `monthlyOutflow` array with an injected small `cap`, instead of trying to contrive real schedule data that takes many cycles to converge (a single bill converges in ~2 cycles: one shortfall-fix, which necessarily overshoots by at most the number of months until that bill is due, then one surplus-fix that absorbs the overshoot — genuinely forcing more cycles needs multiple interacting bills at carefully chosen months/amounts, which isn't worth constructing when the cap itself is trivially unit-testable in isolation).
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
This is the function `category-template-context.ts` will call in Task 5, in place of `runSchedule`, dropping the unused `remainder` parameter.

- [ ] **Step 1: Write the failing tests**

Add `iterateMonthlyContribution` and `runScheduleForecast` to the
`import { createScheduleList, runSchedule } from './schedule-template';`
line added in Task 2 (also add `buildMonthlyOutflow` if it isn't already
imported from Task 3).

```ts
describe('runScheduleForecast', () => {
  beforeEach(() => {
    vi.clearAllMocks();
    vi.mocked(db.getAccounts).mockResolvedValue([]);
    vi.mocked(aqlQuery).mockResolvedValue({ data: [] } as never);
  });

  it('contributes the exact monthly amount for a single monthly schedule', async () => {
    const template_lines = [
      { type: 'schedule', name: 'Rent', priority: 0, directive: 'template' } as const,
    ];
    mockSingleSchedule({ start: '2024-01-15', amount: -10000, frequency: 'monthly' });

    const result = await runScheduleForecast(
      template_lines,
      '2024-01-01',
      0,
      0,
      0,
      [],
      defaultCategory,
      defaultCurrency,
    );

    expect(result.to_budget).toBe(10000);
    expect(result.errors).toHaveLength(0);
  });

  it('never projects a negative month-end balance across two schedules sharing a category', async () => {
    const template_lines = [
      { type: 'schedule', name: 'Monthly', priority: 0, directive: 'template' } as const,
      { type: 'schedule', name: 'Yearly', priority: 0, directive: 'template' } as const,
    ];
    mockSchedulesByName({
      Monthly: { spec: { start: '2024-01-15', amount: -10000, frequency: 'monthly' } },
      Yearly: { spec: { start: '2024-12-15', amount: -60000, frequency: 'yearly' } },
    });

    const result = await runScheduleForecast(
      template_lines,
      '2024-01-01',
      0,
      0,
      0,
      [],
      defaultCategory,
      defaultCurrency,
    );

    // Re-derive the projection with the returned candidate to check the
    // postcondition directly, rather than asserting against runSchedule.
    const { t } = await createScheduleList(
      template_lines as ScheduleTemplate[],
      '2024-01-01',
      defaultCategory,
      defaultCurrency,
    );
    const outflow = await buildMonthlyOutflow(t, '2024-01-01', defaultCategory);
    let runningBalance = 0;
    let minBalance = Infinity;
    for (let i = 0; i < 60; i++) {
      runningBalance += result.to_budget - outflow[i];
      minBalance = Math.min(minBalance, runningBalance);
    }
    expect(minBalance).toBeGreaterThanOrEqual(0);
    // And it should be the *minimal* such contribution: one cent less
    // must produce a negative month somewhere.
    let runningBalanceMinusOne = 0;
    let wentNegative = false;
    for (let i = 0; i < 60; i++) {
      runningBalanceMinusOne += result.to_budget - 1 - outflow[i];
      if (runningBalanceMinusOne < 0) wentNegative = true;
    }
    expect(wentNegative).toBe(true);
  });

  it('covers a same-month-due schedule in full immediately rather than smoothing it away', async () => {
    const template_lines = [
      { type: 'schedule', name: 'DueNow', priority: 0, directive: 'template' } as const,
    ];
    mockSingleSchedule({ start: '2024-01-15', amount: -60000, frequency: 'yearly' });

    const result = await runScheduleForecast(
      template_lines,
      '2024-01-01',
      0,
      0,
      0,
      [],
      defaultCategory,
      defaultCurrency,
    );

    expect(result.to_budget).toBe(60000);
  });

  it('budgets full-flag schedules on top of, and separate from, the forecast', async () => {
    const template_lines = [
      { type: 'schedule', name: 'Full', full: true, priority: 0, directive: 'template' } as const,
      { type: 'schedule', name: 'Smooth', priority: 0, directive: 'template' } as const,
    ];
    mockSchedulesByName({
      Full: { spec: { start: '2024-12-15', amount: -60000, frequency: 'yearly' } },
      Smooth: { spec: { start: '2024-01-15', amount: -10000, frequency: 'monthly' } },
    });

    const result = await runScheduleForecast(
      template_lines,
      '2024-01-01',
      0,
      0,
      0,
      [],
      defaultCategory,
      defaultCurrency,
    );

    // Full schedule isn't due this month, so its on-top contribution is 0;
    // only the smooth monthly schedule contributes.
    expect(result.to_budget).toBe(10000);
    expect(result.perScheduleMonthly.get(template_lines[1])).toBe(10000);
  });

});

describe('iterateMonthlyContribution', () => {
  it('converges within a couple of cycles for a single one-time bill', () => {
    const monthlyOutflow = new Array(60).fill(0);
    monthlyOutflow[30] = 60000; // one bill, due in month 30
    const candidate = iterateMonthlyContribution(0, monthlyOutflow);
    // 60000 / 31 months = 1935.48 -> ceil to 1936; verify it actually
    // covers month 30 and is the minimal such whole-cent amount.
    expect(candidate * 31).toBeGreaterThanOrEqual(60000);
    expect((candidate - 1) * 31).toBeLessThan(60000);
  });

  it('falls back to the last computed (uncorrected) candidate when the cap is hit before converging', () => {
    // Same single-bill-at-month-30 data as the test above. Reaching
    // that scenario's converged answer (1936) takes two cycles: cycle 1
    // detects the shortfall and corrects to 1937 (which necessarily
    // overshoots, since ceil(29000/31) rounds up); cycle 2 sees no
    // negatives and refines the overshoot down to 1936 via the surplus
    // branch. With cap=1, the loop stops right after cycle 1's
    // correction, before that refinement runs — so it returns the
    // overshot 1937, not the fully converged 1936. This is the
    // "fall back to the last computed candidate" behavior: still a
    // safe (slightly conservative) answer, just not the provably
    // minimal one.
    const monthlyOutflow = new Array(60).fill(0);
    monthlyOutflow[30] = 60000;

    const converged = iterateMonthlyContribution(0, monthlyOutflow);
    const capped = iterateMonthlyContribution(0, monthlyOutflow, 1);

    expect(converged).toBe(1936);
    expect(capped).toBe(1937);
    expect(capped).toBeGreaterThan(converged);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @actual-app/core run test schedule-template.test.ts -t "runScheduleForecast"`
Expected: FAIL — `runScheduleForecast` doesn't exist.

- [ ] **Step 3: Implement**

Add after `runSchedule` in `packages/loot-core/src/server/budget/schedule-template.ts`:

```ts
const FORECAST_ITERATION_CAP = 60;

// Pure, synchronous: given a starting balance and a 60-length projected
// monthly-outflow array (an entry can be negative — an unlinked inflow
// transaction offsets that month's need), returns the minimal whole-cent
// monthly contribution that keeps the projected balance non-negative
// across the whole window. `cap` is exposed as a parameter (defaulting
// to FORECAST_ITERATION_CAP) purely so tests can force the "cap reached
// before convergence confirmed" fallback path deterministically without
// having to contrive real-world data that takes many cycles.
export function iterateMonthlyContribution(
  startingBalance: number,
  monthlyOutflow: number[],
  cap: number = FORECAST_ITERATION_CAP,
): number {
  const totalOutflow = monthlyOutflow.reduce((s, v) => s + v, 0);
  let candidate = Math.round(totalOutflow / monthlyOutflow.length);

  for (let cycle = 0; cycle < cap; cycle++) {
    let runningBalance = startingBalance;
    let lowestBalance = Infinity;
    let lowestIndex = -1;
    let firstNegativeIndex = -1;
    let firstNegativeBalance = 0;

    for (let i = 0; i < monthlyOutflow.length; i++) {
      runningBalance += candidate - monthlyOutflow[i];
      if (runningBalance < lowestBalance) {
        lowestBalance = runningBalance;
        lowestIndex = i;
      }
      if (firstNegativeIndex === -1 && runningBalance < 0) {
        firstNegativeIndex = i;
        firstNegativeBalance = runningBalance;
      }
    }

    if (firstNegativeIndex === -1) {
      candidate -= Math.floor(lowestBalance / (lowestIndex + 1));
      break;
    }

    const shortfall = -firstNegativeBalance;
    candidate += Math.ceil(shortfall / (firstNegativeIndex + 1));
    // If `cap` is reached here without a subsequent cycle confirming no
    // negatives remain, this corrected-but-unconfirmed candidate is what
    // gets returned — a safe (if not provably minimal) fallback.
  }

  return candidate;
}

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
}> {
  const scheduleTemplates = template_lines.filter(t => t.type === 'schedule');
  const t = await createScheduleList(
    scheduleTemplates,
    current_month,
    category,
    currency,
  );
  errors = errors.concat(t.errors);

  const fullEntries = t.t.filter(c => c.template.full);
  const smoothEntries = t.t.filter(c => !c.template.full);

  const perScheduleMonthly = new Map<ScheduleTemplate, number>();
  const fullContribution = fullEntries.reduce((sum, c) => {
    perScheduleMonthly.set(c.template, (perScheduleMonthly.get(c.template) ?? 0) + c.target);
    return sum + c.target;
  }, 0);

  if (smoothEntries.length === 0) {
    return { to_budget: to_budget + fullContribution, errors, perScheduleMonthly };
  }

  const monthlyOutflow = await buildMonthlyOutflow(
    smoothEntries,
    current_month,
    category,
  );
  const totalOutflow = monthlyOutflow.reduce((s, v) => s + v, 0);
  const forecastStartingBalance = balance - fullContribution;
  const candidate = iterateMonthlyContribution(
    forecastStartingBalance,
    monthlyOutflow,
  );

  for (const entry of smoothEntries) {
    const share = Math.round((entry.target / totalOutflow) * candidate) || 0;
    perScheduleMonthly.set(
      entry.template,
      (perScheduleMonthly.get(entry.template) ?? 0) + share,
    );
  }

  return {
    to_budget: to_budget + fullContribution + candidate,
    errors,
    perScheduleMonthly,
  };
}
```

Note: `lowestIndex`/`firstNegativeIndex` are recomputed from scratch each
cycle (60 × 60 = 3600 additions worst case) rather than threaded
through incrementally — simple and fast enough at this scale; don't
optimize further without a measured need.

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn workspace @actual-app/core run test schedule-template.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/loot-core/src/server/budget/schedule-template.ts packages/loot-core/src/server/budget/schedule-template.test.ts
git commit -m "$(cat <<'EOF'
[AI] feat: add runScheduleForecast smoothing algorithm

Projects each category's smooth (non-full-flag) schedule outflows 60
months forward and iterates the monthly contribution until the
projected balance never goes negative, converging on the minimal
smoothed amount instead of today's independent-per-schedule sinking
heuristic. Not yet wired into the budget engine — that's the next
commit, gated behind a constant.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 5: Wire the gate into `category-template-context.ts`

**Files:**
- Modify: `packages/loot-core/src/server/budget/category-template-context.ts:30` (import), `:162-163` (state), `:209-231` (`case 'schedule':` branch)

**Interfaces:**
- Consumes: `runScheduleForecast` (Task 4), alongside the existing `runSchedule` import.

- [ ] **Step 1: Add the gate constant and import**

In `packages/loot-core/src/server/budget/category-template-context.ts`, change line 30:
```ts
import { runSchedule, runScheduleForecast } from './schedule-template';
```
Add near the top of the file (after imports, before the class):
```ts
// Build-time switch: selects the 60-month-forecast smoothing algorithm
// (runScheduleForecast) instead of today's payMonthOf/sinking heuristic
// (runSchedule) for schedule-type templates. Not yet a user-facing or
// per-category setting — flip to measure perf before deciding rollout.
// See docs/superpowers/specs/2026-09-02-schedule-forecast-smoothing-design.md
const USE_SCHEDULE_FORECAST = false;
```

- [ ] **Step 2: Switch the call site**

Replace the body of `case 'schedule':` (lines 209-231):
```ts
        case 'schedule': {
          if (!scheduleFlag) {
            const budgeted = this.fromLastMonth + toBudget;
            const ret = USE_SCHEDULE_FORECAST
              ? await runScheduleForecast(
                  t,
                  this.month,
                  budgeted,
                  this.fromLastMonth,
                  toBudget,
                  [],
                  this.category,
                  this.currency,
                )
              : await runSchedule(
                  t,
                  this.month,
                  budgeted,
                  remainder,
                  this.fromLastMonth,
                  toBudget,
                  [],
                  this.category,
                  this.currency,
                );
            // Schedules assume that its to budget value is the whole thing so this
            // needs to remove the previous funds so they aren't double counted
            newBudget = ret.to_budget - toBudget;
            remainder = 'remainder' in ret ? ret.remainder : remainder;
            schedulePerTemplate = ret.perScheduleMonthly;
            scheduleFlag = true;
          }
          break;
        }
```

- [ ] **Step 3: Run the full test suite and typecheck**

Run: `yarn typecheck`
Expected: no errors.

Run: `yarn test`
Expected: all existing tests pass unchanged (gate defaults to `false`, so
`category-template-context.test.ts` and every other existing test still
exercises `runSchedule` exactly as before). The new `runScheduleForecast`
tests from Task 4 also pass.

- [ ] **Step 4: Manually verify the gate can be flipped**

Temporarily set `USE_SCHEDULE_FORECAST = true`, run
`yarn workspace @actual-app/core run test schedule-template.test.ts
category-template-context.test.ts`, confirm nothing throws (some
existing `category-template-context.test.ts` assertions that hardcode
exact `runSchedule` sinking-heuristic numbers may now fail — that's
expected and fine, since this step is only checking the gate wires
correctly, not asserting the forecast produces old-algorithm numbers).
Revert `USE_SCHEDULE_FORECAST` back to `false` before committing.

- [ ] **Step 5: Commit**

```bash
git add packages/loot-core/src/server/budget/category-template-context.ts
git commit -m "$(cat <<'EOF'
[AI] feat: gate runScheduleForecast behind USE_SCHEDULE_FORECAST

Wires the new forecast-based smoothing algorithm into the schedule
template dispatch, defaulting to off so existing behavior and tests
are unaffected until perf is measured and a rollout decision is made.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

## Self-Review Notes

- **Spec coverage:** all sections of the design doc (data assembly,
  occurrence enumeration, iteration algorithm, full-flag handling,
  gating, testing philosophy) map to Tasks 1-5 above.
- **No placeholders:** every step has literal code, not descriptions.
- **Type consistency:** `runScheduleForecast`'s return shape
  (`{ to_budget, errors, perScheduleMonthly }`) matches what
  `category-template-context.ts`'s `case 'schedule':` branch consumes
  in Task 5; `ScheduleTemplateTarget.dateConditions` (Task 2) is the
  exact field `buildMonthlyOutflow` (Task 3) and the forecast tests
  (Task 4) read.

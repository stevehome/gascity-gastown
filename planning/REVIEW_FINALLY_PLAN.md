# Review of `FINALLY_PLAN.md`

## Overall

The plan is detailed and implementable, but it is still written mostly as a long sequence of one-off `gc sling claude` invocations. That makes it easy to read, but it leaves a lot of Gastown/Gas City orchestration value on the table.

The strongest recommendation is to treat FinAlly as an owned convoy with dependency-linked task beads, then submit larger slices into Gastown as grouped work units. That keeps the readable step list, but turns execution into something the city can run serially or in parallel as needed.

This review is based on:

- `gastown/planning/FINALLY_PLAN.md`
- `gastown/assets/gastown/formulas/mol-idea-to-plan.toml`
- `https://github.com/gastownhall/gascity/blob/main/docs/getting-started/coming-from-gastown.md`

The migration guide is especially relevant here because it explicitly says convoys are still the right mental model, but should now be expressed as bead-backed grouping plus sling/formulas/orders rather than as a special runtime layer.

## Main Recommendations

### 1. Keep the human-readable steps, but execute them as convoy slices

Right now each major section is framed as its own `gc sling` command. That works, but it pushes too much orchestration work onto the operator.

Recommendation:

- Create one owned convoy for the FinAlly initiative.
- Create one task bead per implementation slice.
- Add dependency edges with `gc bd dep add`.
- Use `gc sling` to dispatch the ready beads.
- Let Gastown handle which beads can run in parallel and which must wait.

This aligns with the local `mol-idea-to-plan` pattern, which already recommends:

- `gc convoy create "<initiative-name>" --owned`
- `gc convoy add <convoy-id> <task-id>`
- `gc bd dep add` for real blocking relationships

### 2. Submit fewer, better-shaped work units

Several current steps are small enough that they should be submitted together as one unit, with the bead itself describing internal sequencing. The goal is not to make tasks bigger at random, but to make them more vertical and less operator-driven.

Recommended regrouping:

1. `backend-core`
   Covers current Steps 2-6.
   Internal order: foundation -> market stream -> portfolio/watchlist -> chat.

2. `frontend-shell`
   Covers current Steps 7-8.
   Internal order: scaffold -> SSE/watchlist shell.

3. `frontend-trading-experience`
   Covers current Steps 9-12.
   Internal order can be partially serial and partially parallel:
   chart foundation first, then portfolio views, trade bar, and chat panel can be separate child beads if desired.

4. `packaging-and-runbooks`
   Covers current Steps 13-14.

5. `test-hardening`
   Covers current Steps 15-17.
   Internal order: backend/frontend unit tests in parallel, then Playwright.

6. `release-smoke-and-rig`
   Covers current Steps 18-19.
   Internal order: smoke test first, rig move second.

This would reduce dispatch churn while preserving the delivery checkpoints.

### 3. Prefer vertical slices over layer slices where practical

The current plan separates backend and frontend well, but some later steps still encourage horizontal delivery. For example, Steps 9-12 split charts, portfolio, trade bar, and chat panel into separate UI layers even though some of them are user-facing slices that could be independently implemented and verified.

A stronger convoy breakdown would be:

- `market-visibility`
  Watchlist, SSE, sparkline, main chart
- `portfolio-visibility`
  Positions table, heatmap, P&L history
- `trading-loop`
  Trade bar plus refresh/update path
- `assistant-loop`
  Chat panel plus executed-action refresh path

That shape gives better parallelism and cleaner acceptance criteria.

### 4. Add explicit review gates, not just implementation and tests

The current plan has implementation, tests, and smoke verification, but no structured review pass between “code exists” and “we trust it.”

Recommendation:

- Add a post-implementation architecture/code review gate before packaging is finalized.
- Add a cross-model review gate using Gemini before the final smoke test or immediately after it.

This matches the pattern already used in `mol-idea-to-plan`, where review is dispatched as multiple legs and then synthesized, instead of being treated as an afterthought.

### 5. Make dependencies explicit in the plan, not just in notes

The Notes section already captures some dependencies, but they are prose-only. If this plan is going to be run through Gastown, the important dependencies should be modeled as bead dependencies from the start.

Key dependency graph to encode:

- `backend-core` blocks all frontend work beyond pure scaffold
- `frontend-shell` blocks the rest of the frontend slices
- `packaging-and-runbooks` depends on backend-core + all frontend slices
- `test-hardening.playwright` depends on packaging-and-runbooks
- `release-smoke-and-rig` depends on packaging-and-runbooks + test-hardening

## Concrete Convoy Proposal

### Parent convoy

Create:

```bash
gc convoy create "finally-build" --owned
gc convoy target <convoy-id> integration/<convoy-id>
```

### Child beads

Suggested initial child beads:

1. `finally-backend-core`
2. `finally-frontend-shell`
3. `finally-market-visibility`
4. `finally-portfolio-visibility`
5. `finally-trading-loop`
6. `finally-assistant-loop`
7. `finally-packaging-runbooks`
8. `finally-backend-tests`
9. `finally-frontend-tests`
10. `finally-e2e`
11. `finally-gemini-review`
12. `finally-release-smoke`
13. `finally-rig-finalize`

### Suggested dependencies

```text
finally-backend-core
  -> finally-market-visibility
  -> finally-portfolio-visibility
  -> finally-trading-loop
  -> finally-assistant-loop

finally-frontend-shell
  -> finally-market-visibility
  -> finally-portfolio-visibility
  -> finally-trading-loop
  -> finally-assistant-loop

finally-market-visibility
finally-portfolio-visibility
finally-trading-loop
finally-assistant-loop
  -> finally-packaging-runbooks

finally-backend-core
  -> finally-backend-tests

finally-market-visibility
finally-portfolio-visibility
finally-trading-loop
finally-assistant-loop
  -> finally-frontend-tests

finally-packaging-runbooks
finally-backend-tests
finally-frontend-tests
  -> finally-e2e

finally-e2e
  -> finally-gemini-review

finally-gemini-review
  -> finally-release-smoke

finally-release-smoke
  -> finally-rig-finalize
```

This keeps genuine blockers serial, while allowing the controller to exploit parallelism where it is safe.

## Proposed New Review Step

Add a new step after Playwright and before the final smoke test:

## Step 17.5 — Gemini review

Cross-model review of the implementation, test coverage, and operator workflow before final release validation.

```bash
gc sling gemini "
Review ~/projects/portfolio/finally/ against:
- ~/projects/gascity/gastown/planning/PORTFOLIO_SPEC.md
- ~/projects/gascity/gastown/planning/FINALLY_PLAN.md

Focus on:
- behavioral regressions or missing acceptance criteria
- API/UI mismatches between backend and frontend
- packaging and Docker/runtime risks
- missing tests or weak assertions
- operator workflow issues in start/stop scripts and smoke-test flow

Output:
- a concise review report
- critical issues
- non-blocking concerns
- exact recommended fixes

Do not make broad speculative changes. Prioritize concrete release risks.
Work dir: ~/projects/portfolio/finally/
" --rig portfolio
```

Done when:

- Gemini produces a written review report
- all critical findings are fixed or explicitly accepted before Step 18

If you want a stronger Gastown-native version, make this a review bead inside the parent convoy and synthesize it with any Claude-authored self-review before the smoke test.

## Specific Notes On Current Steps

### Steps 2-6

These are currently treated as separate backend tasks, but they are tightly coupled and mostly owned by the same subsystem. They are a good candidate for one parent bead with serial sub-acceptance criteria, or for a parent convoy with 4 child beads and strict dependencies.

### Steps 9-12

The current note says Steps 8-12 can be dispatched in parallel once Step 7 is done. That is optimistic as written.

Concerns:

- Step 9 mutates `usePrices` and watchlist rendering.
- Step 10 introduces portfolio hooks and main layout changes.
- Step 11 also touches shared page state and refresh flows.
- Step 12 touches shared layout and refresh flows too.

These can still be parallelized, but only if the plan defines file ownership or clear interface boundaries. Otherwise this is likely to create merge friction.

Recommendation:

- either run 9-12 more serially
- or split them into beads with explicit ownership boundaries and a shared integration contract

### Steps 13-14

These fit well as one convoy slice. Packaging and run scripts are operator-facing release work and should probably land together.

### Step 19

Step 19 is already marked done while Steps 9-18 are still todo. That is not necessarily wrong, but it is operationally surprising because the note below says the rig move should happen after Step 18.

Recommendation:

- either move Step 19 into a “completed setup changes” section
- or change its status back to todo/deferred until the release convoy closes

## Recommended Edits To `FINALLY_PLAN.md`

Minimum change version:

1. Add a short “Execution model” section near the top explaining that the plan should be instantiated as an owned convoy with dependency-linked beads.
2. Collapse the step list into larger convoy-ready slices while keeping the current substeps as acceptance criteria.
3. Add Step 17.5 for Gemini review.
4. Replace the Notes section with a dependency table or explicit convoy graph.

Best version:

1. Keep the current detailed step text.
2. Add a new section called `Convoy Execution Plan`.
3. Define parent convoy, child beads, and dependency edges there.
4. Mark which beads are safe for parallel dispatch.
5. Add the Gemini review bead and release gate.

## Bottom Line

The plan is good as a build checklist. It is weaker as a Gastown execution plan.

The biggest improvement is to stop thinking of each section as “one more manual sling command” and instead express FinAlly as one owned convoy with a small number of vertical child beads, explicit dependencies, and a cross-model Gemini review gate before release.

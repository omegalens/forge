# Optimization Handoff: Harness to Autoresearch

After the harness completes its build phase, this protocol bridges the structured multi-agent work into an autoresearch optimization loop. The harness gets you to "done." Autoresearch gets you to "excellent."

## Table of Contents
- [Prerequisites](#prerequisites)
- [Extracting the Autoresearch Config](#extracting-the-autoresearch-config)
- [Building the Verify Command](#building-the-verify-command)
- [Building the Guard Command](#building-the-guard-command)
- [Optimization Strategy](#optimization-strategy)
- [Example Handoff](#example-handoff)
- [Failure Modes](#failure-modes)

---

## Prerequisites

Before launching the optimization loop, verify:

1. **All features passing** — the feature list shows no failing features. If features are still broken, fix them in the harness first. Autoresearch optimizes quality, not correctness.
2. **Evaluator is calibrated** — you've tuned the evaluator through 3+ iterations and trust its scoring. Optimizing against an uncalibrated evaluator amplifies its biases.
3. **Grading criteria are stable** — don't change criteria during the optimization loop. The metric must be consistent across iterations for autoresearch's keep/discard logic to work.
4. **Final evaluator report exists** — the last QA report provides the baseline scores that autoresearch will try to improve.

---

## Extracting the Autoresearch Config

The harness already has all the information autoresearch needs. Map it:

### Goal

Construct from the evaluator's weakest scores:

```
Goal: Maximize the evaluator's composite score against the product spec.
Priority criteria (currently below threshold):
- [criterion 1]: scoring X/10, threshold is Y
- [criterion 2]: scoring X/10, threshold is Y
Secondary criteria (above threshold but with headroom):
- [criterion 3]: scoring X/10
```

The goal should name the specific criteria that need improvement. "Make it better" is too vague — "raise design quality from 6/10 to 8/10 while maintaining functionality at 9/10" gives autoresearch a clear target.

### Scope

Same files the generator worked on. Pull from the harness's feature list or git history:

```
Scope: src/components/**/*.tsx, src/styles/**/*.css, public/**/*.html
```

Exclude test files, config, and the harness's own artifacts (features.json, progress.txt, etc.) — autoresearch should optimize the product, not the scaffolding.

### Metric

The evaluator's composite score. This must be a single number that autoresearch can parse.

```
Metric: Evaluator composite score (weighted average of all criteria, 0-100)
Direction: Higher is better
```

### Verify

A command that re-runs the evaluator and outputs the composite score. See [Building the Verify Command](#building-the-verify-command).

### Guard

A command that confirms all features still pass. See [Building the Guard Command](#building-the-guard-command).

---

## Building the Verify Command

The verify command must:
1. Launch the application (if not already running)
2. Run the evaluator agent against the live output
3. Parse the composite score from the evaluator's report
4. Output a single number to stdout

### Option A: Evaluator as a subagent

If using Claude Code skills + subagents, the verify step spawns the evaluator as a subagent that:
- Reads the grading criteria
- Tests the live application (via Playwright MCP or equivalent)
- Writes an evaluation report with scores
- Outputs the composite score

The autoresearch loop reads the composite score from the report file.

### Option B: Evaluator as a script

If using the Agent SDK or custom orchestration, wrap the evaluator in a script:

```bash
#!/bin/bash
# verify.sh — runs evaluator and outputs composite score
node run-evaluator.js --criteria grading-criteria.json --output eval-report.json
jq '.composite_score' eval-report.json
```

### Option C: Evaluator inline in the loop

If the autoresearch session has enough context, the evaluator logic can run inline — reading the criteria, testing the app, scoring each criterion, computing the composite. This is simpler but means the same context window handles both generation and evaluation, which weakens the separation that makes the evaluator valuable.

**Recommendation:** Options A or B. The whole point of having a separate evaluator is that it judges independently. Preserve that independence in the optimization loop.

---

## Building the Guard Command

The guard prevents autoresearch from regressing features while optimizing quality. It must:
1. Run through the feature list
2. Verify each feature still passes
3. Exit 0 if all pass, non-zero if any fail

### Option A: Automated feature verification

If the harness already has automated tests for each feature:

```bash
#!/bin/bash
# guard.sh — verify all features still pass
npm test && npx playwright test --config=e2e.config.ts
```

### Option B: Feature list checker

If features are verified through interactive testing:

```bash
#!/bin/bash
# guard.sh — run feature verification suite
node verify-features.js --features features.json --report guard-report.json
FAILING=$(jq '[.features[] | select(.passes == false)] | length' guard-report.json)
if [ "$FAILING" -gt 0 ]; then
  echo "GUARD FAILED: $FAILING features regressed"
  exit 1
fi
echo "All features passing"
exit 0
```

### Option C: Lightweight regression check

For simpler projects, the guard can be a build + type check + existing test suite:

```bash
npm run build && npm run typecheck && npm test
```

This doesn't verify every feature interactively but catches most regressions.

---

## Optimization Strategy

Autoresearch's "one change per iteration" rule applies, but the strategy for WHAT to change comes from the evaluator's feedback.

### Criterion-focused iterations

Each iteration should target a specific criterion:

```
Iteration 1-3: Focus on lowest-scoring criterion (e.g., "design quality")
Iteration 4-6: Focus on second-lowest criterion (e.g., "originality")
Iteration 7-10: Focus on any criterion with remaining headroom
Iteration 11+: Cross-criterion improvements (changes that lift multiple scores)
```

### Reading the evaluator's feedback

The evaluator's reports from the harness phase contain specific findings. Use these as the starting point:

- "Navigation feels inconsistent" → iteration 1 targets navigation patterns
- "Color palette is generic" → iteration 2 targets color system
- "Empty states are unhandled" → iteration 3 targets edge cases

### Avoiding criterion conflict

Optimizing one criterion can hurt another. The guard catches feature regressions, but it won't catch score regressions on other criteria. The verify step (which re-runs the full evaluator) handles this — if the composite score drops, autoresearch reverts.

For extra safety, track per-criterion scores across iterations. If a criterion drops more than 1 point while another improves, flag it for review.

### When to stop

- **All criteria above threshold**: the optimization achieved its goal
- **Scores plateau for 3+ iterations**: you've extracted what this approach can give
- **Iteration budget exhausted**: 10-15 iterations is the default budget
- **Diminishing returns**: improvements shrink below 0.5 points per iteration

---

## Example Handoff

A web application harness just completed. Final evaluator report:

| Criterion | Score | Threshold |
|---|---|---|
| Feature completeness | 9/10 | 7 |
| Functionality | 8/10 | 7 |
| Visual design | 5/10 | 7 |
| Code quality | 8/10 | 6 |
| Product depth | 6/10 | 7 |

Two criteria below threshold (visual design, product depth). Autoresearch config:

```
/autoresearch
Goal: Raise visual design from 5→8 and product depth from 6→8. Maintain functionality ≥8 and feature completeness ≥9.
Scope: src/components/**/*.tsx, src/styles/**/*.css
Metric: Evaluator composite score (higher is better)
Verify: Run evaluator subagent → parse composite from eval-report.json
Guard: npm test && node verify-features.js
Iterations: 15
```

Expected trajectory:
- Iterations 1-5: Visual design improvements (color, typography, layout coherence) — score 5→7
- Iterations 6-10: Product depth (empty states, edge cases, micro-interactions) — score 6→8
- Iterations 11-15: Cross-cutting polish — composite score converges toward ceiling

---

## Failure Modes

### Optimizing garbage criteria
If the grading criteria are poorly written or uncalibrated, autoresearch will optimize toward the wrong target. The evaluator will report high scores on mediocre output. **Fix:** calibrate the evaluator before launching the optimization loop.

### Guard too loose
If the guard doesn't catch feature regressions, autoresearch may break features while improving scores. **Fix:** ensure the guard exercises actual feature behavior, not just compilation.

### Guard too tight
If the guard fails on changes that don't actually break features (flaky tests, timing issues), autoresearch wastes iterations on reverts. **Fix:** make the guard deterministic. Remove flaky tests or add retries.

### Evaluator inconsistency
If the evaluator scores the same output differently across runs, autoresearch's keep/discard decisions become random. **Fix:** add few-shot calibration examples to the evaluator. Reduce randomness in the evaluator's testing path.

### Scope too narrow
If the scope excludes files that affect the criteria being optimized (e.g., excluding CSS when optimizing visual design), autoresearch can't make meaningful changes. **Fix:** include all files that influence the target criteria.

### Scope too broad
If the scope includes infrastructure, tests, or config, autoresearch might "optimize" by modifying the evaluation machinery. **Fix:** exclude test files, evaluator configs, and harness artifacts from scope.

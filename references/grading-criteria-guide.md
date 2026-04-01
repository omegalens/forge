# Designing Grading Criteria

The grading criteria you give your evaluator are the single most impactful part of harness design. They determine what your evaluator looks for, how it scores, and — because you share them with the generator too — they directly shape the character of what gets produced.

## Table of Contents
- [Why Criteria Matter](#why-criteria-matter)
- [Writing Effective Criteria](#writing-effective-criteria)
- [Domain-Specific Examples](#domain-specific-examples)
- [Calibrating the Evaluator](#calibrating-the-evaluator)
- [Common Pitfalls](#common-pitfalls)

---

## Why Criteria Matter

Without criteria, asking an evaluator "is this good?" produces inconsistent, generally positive scores. The evaluator has no anchor for what "good" means, so it defaults to being generous.

With criteria, the evaluator has something concrete to grade against. "Does this follow our principles for good design?" is a tractable question. The criteria also serve double duty: shared with the generator, they steer it toward the qualities you care about before the evaluator even provides feedback.

The wording matters more than you might expect. Including phrases like "the best designs are museum quality" pushed outputs toward a particular visual convergence in Anthropic's testing. The criteria language directly shapes the aesthetic direction of the output.

---

## Writing Effective Criteria

### 1. Identify your dimensions

Start by asking: what makes output in this domain good vs. mediocre? Break this into 3-6 independent dimensions. Each should be assessable on its own.

### 2. Weight toward weakness

The model already performs well on some dimensions by default. In frontend design, Claude scores well on craft (typography, spacing, contrast) and functionality (usability, navigation) out of the box. But it underperforms on design quality (coherent aesthetic identity) and originality (non-generic creative choices).

Weight your criteria toward the dimensions where the model actually needs help. Over-weighting dimensions the model already handles well just inflates scores without improving output.

### 3. Define what failure looks like

For each criterion, specify the failure mode explicitly. This is more actionable than describing success, because the evaluator already defaults to being generous.

**Weak**: "The design should be original."
**Strong**: "Is there evidence of custom decisions, or is this template layouts, library defaults, and AI-generated patterns? A human designer should recognize deliberate creative choices. Unmodified stock components — or telltale signs of AI generation like purple gradients over white cards — fail here."

### 4. Set hard thresholds

Each criterion should have a minimum score threshold. If any criterion falls below its threshold, the sprint fails. This prevents the evaluator from averaging away serious issues. A gorgeous design with broken navigation should fail, not pass with a "mixed" score.

### 5. Mind the language

Criteria language is a steering mechanism, not just a measurement tool. Experiment with different phrasings:
- "Museum quality" → pushes toward refined, polished aesthetics
- "Bold and experimental" → pushes toward risk-taking
- "A senior engineer would approve this in code review" → pushes toward production-grade practices
- "A real user would find this intuitive on first visit" → pushes toward usability

---

## Domain-Specific Examples

### Frontend Design

| Criterion | Weight | What it measures |
|---|---|---|
| **Design quality** | High | Does the design feel like a coherent whole? Colors, typography, layout, imagery combine to create a distinct mood and identity. |
| **Originality** | High | Evidence of custom decisions vs. template defaults. AI-generated patterns (purple gradients, white cards, generic stock imagery) fail. |
| **Craft** | Normal | Technical execution: typography hierarchy, spacing consistency, color harmony, contrast ratios. A competence check. |
| **Functionality** | Normal | Usability: can users understand the interface, find primary actions, complete tasks without guessing? |

### Full-Stack Application

| Criterion | Weight | What it measures |
|---|---|---|
| **Feature completeness** | High | Are spec features actually implemented with interactive depth, or are they display-only stubs? Core interactions must work, not just render. |
| **Functionality** | High | End-to-end correctness. Tested through the UI as a user would, not just via unit tests or API calls. |
| **Visual design** | Normal | Consistent visual identity, sensible layout, good use of space. Not beautiful, but not broken. |
| **Code quality** | Normal | Clean architecture, no dead code, proper error handling at system boundaries. |
| **Product depth** | Normal | Does the feature feel complete? Does it handle edge cases a real user would encounter? |

### Content Generation (writing, reports, documentation)

| Criterion | Weight | What it measures |
|---|---|---|
| **Accuracy** | High | Factual correctness. Claims are supported. Numbers match sources. |
| **Structure** | High | Logical flow. Reader can follow the argument without re-reading. Sections build on each other. |
| **Voice** | Normal | Consistent tone appropriate to the audience. Not robotic, not trying too hard. |
| **Completeness** | Normal | All required sections present. No gaps in coverage. |

---

## Calibrating the Evaluator

An uncalibrated evaluator drifts — its scores become inconsistent across runs, or it develops blind spots. Calibration fixes this.

### Few-shot scoring examples

Give the evaluator 2-3 examples of previously scored outputs with detailed breakdowns. Each example should show:
- The output (or a summary of it)
- The score for each criterion
- A sentence or two explaining why each score was given

This anchors the evaluator's judgment and prevents both inflation and deflation.

### The tuning loop

1. Run the evaluator on a test output
2. Read its full log — not just the final scores, but its reasoning
3. Identify where its judgment diverges from yours:
   - Did it flag something that isn't actually a problem?
   - Did it miss something you consider important?
   - Did it score a mediocre output too generously?
4. Update the evaluator's prompt to address the specific divergence
5. Rerun and check

Expect 3-5 iterations before the evaluator grades in a way you find reasonable. This is normal — the tuning cost is small compared to the quality improvement it enables.

### Common evaluator failures

- **Talking itself out of failures**: The evaluator identifies a real issue, then rationalizes it away ("while the navigation is slightly confusing, the overall experience is still positive"). Counter by instructing it to be skeptical and to not rationalize away problems it identifies.
- **Superficial testing**: The evaluator checks happy paths but doesn't probe edge cases. Counter by listing specific interaction patterns to test (e.g., "try submitting empty forms", "navigate back and forward", "test with very long text").
- **Score inflation**: Everything scores 7-9/10 regardless of quality. Counter with calibration examples that include mediocre outputs scored honestly, and by defining what each score level means.

---

## Common Pitfalls

**Too many criteria**: More than 6 criteria dilutes evaluator attention. If you have more concerns, group related ones under a single criterion.

**Criteria the model already handles well**: If craft and functionality are consistently fine, don't weight them heavily. You're wasting evaluator attention on things that don't need it.

**Vague failure conditions**: "The design should be good" teaches the evaluator nothing. Be specific about what bad looks like in your domain.

**No calibration examples**: Without anchoring, the evaluator invents its own scoring scale, which drifts between runs.

**Criteria that conflict**: If one criterion rewards simplicity and another rewards feature richness, the evaluator gives mixed signals. Make sure criteria are compatible, or explicitly state which takes priority in a conflict.

**Ignoring criteria language effects**: Remember that criteria are shared with the generator. The language you use to describe quality directly shapes what the generator produces. This is a feature, not a bug — but test different phrasings to find the character you want.

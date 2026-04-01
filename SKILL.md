---
name: forge
description: Forge multi-agent systems for long-running autonomous tasks — coding, design, content generation, or any domain requiring sustained quality over hours of autonomous work. Use this skill when the user wants to build a multi-agent system, create a generator-evaluator loop, design an autonomous coding pipeline, set up a planner-builder-QA architecture, or improve the quality of long-running agent outputs. Also trigger when the user mentions "harness", "harness-design", "multi-agent architecture", "agent orchestration", "autonomous app generation", "GAN-inspired agents", "forge", long-running coding pipelines, or wants to make Claude work autonomously for extended periods on complex projects. Even if the user is just exploring whether they need a multi-agent setup, invoke this skill to help them decide.
---

# Harness Design for Long-Running Autonomous Tasks

A harness is the scaffolding around a model that shapes how it approaches a task: the agent architecture, prompts, communication protocols, evaluation criteria, and context management strategies. The goal is turning a raw model into a reliable autonomous system that sustains quality over hours of work.

The foundational insight: **every component in a harness encodes an assumption about what the model can't do on its own.** Those assumptions deserve stress-testing — they may be wrong, and they go stale as models improve.

## When you need a harness

A harness adds value when:
- The task exceeds a single context window
- Quality requires iteration (design, complex features, subjective output)
- Self-evaluation is unreliable for the domain
- The task has both creative and verification dimensions

You probably don't need one when:
- The task fits in one context window
- A single skill or prompt gets good results
- The cost/latency overhead outweighs the quality gain

Be honest about this tradeoff. A harness that costs $200 and runs 6 hours needs to produce substantially better output than a $9, 20-minute solo run. If you're unsure, try the solo approach first and read the traces — your harness should address the failures you actually observe, not hypothetical ones.

## The Design Workflow

### 1. Identify your failure modes

Before designing agents, run a naive implementation and read its traces. Common failure modes:

| Failure mode | What it looks like |
|---|---|
| **Context degradation** | Model loses coherence as context fills. Some models wrap up prematurely ("context anxiety"). |
| **Self-evaluation bias** | Agent confidently praises mediocre output. Especially pronounced for subjective tasks. |
| **One-shotting** | Agent tries to build everything at once instead of working incrementally. |
| **Premature completion** | Agent sees progress and declares the job done with features remaining. |
| **Shallow testing** | Agent marks features complete using superficial checks rather than end-to-end testing. |

Your failure modes determine your architecture. Don't build a three-agent system for a problem that a two-agent system solves.

### 2. Choose your architecture

See `references/architecture-patterns.md` for full patterns with examples. Summary:

**Two-agent (initializer + worker)** — for context management problems
- Initializer expands the prompt into structured spec + feature list
- Worker implements one feature at a time, leaving clean handoff artifacts
- Best when the main issue is sustaining coherent work across context windows

**Generator-evaluator** — for quality problems
- Generator produces output; evaluator grades it against concrete criteria
- Evaluator feedback drives the generator toward better outputs over 5-15 iterations
- Best when output quality is subjective or self-evaluation is unreliable
- Inspired by GANs: separating creation from judgment creates a productive feedback loop

**Three-agent (planner + generator + evaluator)** — for scope + quality problems
- Planner expands short prompt into full product spec
- Generator implements features; evaluator tests the running output
- Before each sprint, the two negotiate a "sprint contract" — what done looks like
- Best for complex builds requiring both scope management and quality assurance

**Decision guide:**
- Context management is the bottleneck → two-agent
- Output quality is the bottleneck → generator-evaluator
- Both → three-agent
- Start simple. Add agents only when traces show the simpler architecture falling short.

### 3. Design grading criteria

This is the highest-leverage part of harness design. See `references/grading-criteria-guide.md` for a deep dive.

The core principle: "Is this good?" is hard to answer consistently, but "does this follow our principles for good [X]?" gives the evaluator something concrete to grade against. Subjective quality judgments improve dramatically when encoded as specific, gradable criteria.

**How to write criteria:**
1. **Assess independently** — each criterion should stand alone
2. **Weight toward weakness** — the model already scores well on some dimensions by default; weight criteria toward where it actually underperforms
3. **Penalize known failure patterns** — explicitly call out patterns to avoid (e.g., generic "AI slop" in design, stub implementations in code)
4. **Calibrate with examples** — few-shot scoring examples with detailed breakdowns prevent evaluator drift
5. **Mind the wording** — criteria language directly shapes output character; phrases like "museum quality" or "a human designer should recognize deliberate creative choices" push the generator in specific directions

Set hard score thresholds: if any criterion falls below its threshold, the sprint fails and the generator gets detailed feedback.

### 4. Design communication artifacts

Agents communicate through files, not conversational context. Files persist across context resets, create an audit trail, and any agent can read them without sharing a conversation.

**Essential artifacts:**

| Artifact | Purpose | Format tip |
|---|---|---|
| **Spec / feature list** | Source of truth for what to build. Features marked passing/failing. | Use JSON — models are less likely to inappropriately modify JSON than markdown. |
| **Progress file** | Log of what each session accomplished. | Supplements git commits. |
| **Sprint contracts** | What "done" looks like, agreed before implementation. | Bridges the gap between high-level user stories and testable behavior. |
| **Evaluation reports** | Detailed feedback from evaluator to generator. | Specific findings tied to grading criteria. |
| **init.sh** | Environment setup so each session starts quickly. | Run dev servers, verify basic functionality. |

Protect the feature list with strong instructions: "It is unacceptable to remove or edit tests — this could lead to missing or buggy functionality." Agents should only change the pass/fail status.

### 5. Configure context management

**Compaction** (summarize earlier context in place):
- Preserves continuity, lower overhead
- Doesn't give a clean slate — context anxiety can persist
- Good default for capable models

**Context resets** (fresh agent with structured handoff):
- Clean slate, eliminates context anxiety
- Requires handoff artifacts to carry state
- Higher orchestration overhead
- Essential when the model exhibits context anxiety strongly enough that compaction alone isn't sufficient

**Neither** (single continuous session with auto-compaction):
- Works when the model handles long contexts well natively
- Simplest to implement
- Test this first with each new model — you may not need resets anymore

Each new session should start with a "get up to speed" sequence: read progress files, check git logs, run init.sh, verify basic functionality, then pick up the next feature. This saves tokens and catches any broken state immediately.

### 6. Wire up testing

Absent explicit prompting, agents tend to test superficially — unit tests and curl commands rather than end-to-end verification. The fix is giving the evaluator interactive tools.

- **Playwright MCP** for web apps: the evaluator navigates the live page, clicks through features, screenshots before scoring
- **API testing tools** for backends: exercise endpoints with realistic payloads
- **The evaluator should test as a user would** — not just verify the code compiles

This dramatically improves bug detection. The evaluator should probe edge cases, not just happy paths.

### 7. Iterate on the evaluator

Getting an evaluator to grade reliably takes real work. Out of the box, models are poor QA agents — they identify legitimate issues, then talk themselves into deciding they aren't a big deal.

**Tuning loop:**
1. Run the evaluator
2. Read its full logs (not just final scores)
3. Find examples where its judgment diverges from yours
4. Update the evaluator's prompt to address those specific divergences
5. Repeat until it grades in a way you find reasonable

Tuning a standalone evaluator to be skeptical is far more tractable than making a generator critical of its own work. This is the core leverage of the generator-evaluator split.

### 8. Post-completion optimization

Once the harness delivers a working product — features passing, spec met — there's almost always a quality gap between "done" and "as good as the spec envisions." This is where `/autoresearch` takes over.

The insight: the harness already produced everything autoresearch needs to run an optimization loop. The grading criteria become the metric, the feature list becomes the guard, and the evaluator becomes the verify command. No new configuration required — just a structured handoff.

See `references/optimization-handoff.md` for the full protocol. Summary:

**Handoff mapping:**

| Harness artifact | Autoresearch config | How it maps |
|---|---|---|
| Grading criteria + evaluator | **Metric + Verify** | Re-run the evaluator, parse its composite score. The score IS the metric. |
| Product spec | **Goal** | "Maximize evaluator score against the spec. Focus on criteria scoring below threshold." |
| Feature list (all passing) | **Guard** | All features must remain passing. Regressions revert the iteration. |
| Generator's output files | **Scope** | Same files the generator was working on. |

**When to use this:**
- The harness got everything working but evaluator scores plateau below your bar
- Subjective quality dimensions (design, UX, writing voice) need more iteration than the harness's sprint structure allows
- You want to squeeze the last 10-20% of quality without manually directing it

**When to skip:**
- Evaluator scores already meet your thresholds across all criteria
- The output is functional and polish isn't worth the cost
- The grading criteria aren't well-calibrated (optimize garbage criteria, get garbage output)

**How it works:**

1. The harness completes — all features passing, final evaluator report written
2. Extract the evaluator's composite score as the baseline metric
3. Identify which criteria scored lowest — these become the optimization focus
4. Launch `/autoresearch` with:
   - **Goal**: Maximize evaluator composite score, prioritizing lowest-scoring criteria
   - **Scope**: The generator's output files
   - **Metric**: Evaluator composite score (higher is better)
   - **Verify**: Re-run the evaluator against the live output, parse the composite score
   - **Guard**: Run the feature list verification (all features must stay passing)
5. Autoresearch iterates: modify → verify → keep/discard → repeat
6. Each iteration targets a specific criterion, makes one focused change, and checks that the score improved without regressing other criteria or breaking features

**The key constraint:** autoresearch should optimize within the harness's architecture, not rebuild it. If a change breaks the feature list guard, it gets reverted — period. This prevents the optimization loop from undoing the harness's work.

**Iteration budget:** Start with 10-15 iterations. Quality gains follow a power curve — the first 5 iterations capture most of the improvement. Beyond 20, you're likely past diminishing returns unless the evaluator identifies a specific stubborn criterion.

### 9. Simplify ruthlessly

After the harness produces good results, systematically strip it down:

1. Remove one component at a time
2. Run the same test prompt
3. Compare output quality
4. If quality holds, the component wasn't load-bearing — remove it

**Why this matters:**
- Cheaper and faster runs
- Less orchestration complexity to debug
- Components essential for Model N may be unnecessary for Model N+1
- A radical first cut often fails — move methodically, one piece at a time

**When a new model drops:**
- Re-examine every component against the new model's capabilities
- Strip what's no longer load-bearing
- Add new pieces for capabilities that weren't possible before
- The interesting harness design space doesn't shrink as models improve — it moves

## Anti-patterns

- **Over-specifying in the planner**: Keep the planner focused on product context and deliverables, not granular implementation. If the planner makes a wrong technical call, errors cascade downstream. Let the generator figure out the path.
- **Evaluating without interactive tools**: An evaluator that reads code is far less effective than one that clicks through the running application.
- **Skipping trace analysis**: Reading what agents actually did is the most important debugging tool. Don't just look at final output.
- **Keeping stale components**: Every harness run should make you question whether each piece is still earning its keep.
- **Uncalibrated evaluators**: Without few-shot scoring examples, evaluator judgment drifts across runs.
- **Generator self-evaluation**: Separating generation from evaluation is the entire point. Don't ask the generator to also be its own critic.

## Choosing your tools

This skill is framework-agnostic. The patterns work across:
- **Claude Agent SDK** (Python/TypeScript) — straightforward orchestration, built-in compaction
- **Claude Code skills + subagents** — native to the Claude Code environment
- **Custom orchestration** — any system that can spawn agents, pass files, and manage context

The architecture matters more than the framework. Pick whatever lets you read traces easily and iterate on prompts quickly.

# Architecture Patterns

Detailed reference for the three core multi-agent architectures. Read this when designing a harness and you need specifics on agent personas, communication flow, and session structure.

## Table of Contents
- [Pattern 1: Two-Agent (Initializer + Worker)](#pattern-1-two-agent)
- [Pattern 2: Generator-Evaluator](#pattern-2-generator-evaluator)
- [Pattern 3: Three-Agent (Planner + Generator + Evaluator)](#pattern-3-three-agent)
- [Choosing Between Patterns](#choosing-between-patterns)
- [Combining Patterns](#combining-patterns)

---

## Pattern 1: Two-Agent

**Use when:** The main challenge is sustaining coherent work across many context windows. Quality within a single session is acceptable, but handoffs between sessions degrade the output.

### Agent personas

**Initializer agent** (runs once, at the start):
- Takes a user prompt and expands it into a structured environment
- Produces: feature list (JSON), init.sh, progress file, initial git commit
- The feature list is the source of truth — it should enumerate every discrete feature the final product needs, with each marked as passing or failing
- Does NOT implement anything — it sets the stage

**Worker agent** (runs in a loop, one session per feature):
- Reads progress files and git history at the start of each session
- Runs init.sh to verify the environment is in a working state
- Picks the highest-priority feature that isn't yet done
- Implements that one feature, tests it end-to-end
- Marks the feature as passing only after thorough verification
- Commits to git with a descriptive message
- Updates the progress file
- Leaves the codebase in a merge-ready state

### Session flow

```
Session 1 (Initializer):
  prompt → feature list + init.sh + progress file + git init

Session 2..N (Worker):
  read progress file → read git log → run init.sh → verify basic functionality
  → pick next feature → implement → test end-to-end → mark passing
  → git commit → update progress → clean exit
```

### Key design decisions

- **Feature list format**: Use JSON with structured fields (category, description, steps, passes). Models are less likely to corrupt JSON than markdown.
- **Strong protection of the feature list**: Instruct agents that editing or removing tests is unacceptable. They may only change the passes field.
- **Verification before new work**: Each session starts by verifying the app still works before touching new code. This catches silent regressions from previous sessions.
- **One feature at a time**: This is critical. Attempting multiple features per session leads to half-finished states that compound across context boundaries.

### Example prompt structure

**Initializer prompt** (abbreviated):
```
You are setting up the initial environment for a long-running project.
Given the user's prompt, create:
1. A comprehensive feature list in JSON format (features.json)
2. An init.sh script for running the development environment
3. A progress file (progress.txt) for tracking session-by-session work
4. An initial git commit

Be ambitious about scope. Enumerate every feature the product needs.
Mark all features as "passes": false.
Do NOT begin implementation.
```

**Worker prompt** (abbreviated):
```
You are continuing work on an ongoing project.
1. Run pwd to orient yourself
2. Read progress.txt and run git log --oneline -20
3. Run init.sh to start the development environment
4. Use [browser tool] to verify basic functionality still works
5. Read features.json and pick the highest-priority failing feature
6. Implement ONLY that feature
7. Test it end-to-end as a user would
8. Update features.json (only the passes field)
9. Git commit with a descriptive message
10. Update progress.txt with what you did
```

---

## Pattern 2: Generator-Evaluator

**Use when:** Output quality is the bottleneck, especially for subjective domains (design, writing, creative work) where self-evaluation is unreliable.

### Agent personas

**Generator agent**:
- Produces output based on the user prompt and grading criteria
- After receiving evaluator feedback, makes a strategic choice: refine the current approach if scores trend well, or pivot to a different approach if the direction isn't working
- Has access to implementation tools (code editor, file system, etc.)

**Evaluator agent**:
- Grades the generator's output against defined criteria
- Writes a detailed critique with specific, actionable feedback
- Has access to interactive tools (Playwright MCP, browser) to test the output directly
- Calibrated with few-shot scoring examples to prevent drift
- Prompted to be skeptical — the default tendency is to be too generous

### Iteration flow

```
Iteration 1:
  generator → output v1
  evaluator → scores + critique

Iteration 2:
  generator reads critique → decides: refine or pivot → output v2
  evaluator → scores + critique

... repeat 5-15 iterations until scores plateau
```

### Key design decisions

- **Iteration count**: 5-15 iterations is typical. Scores generally improve before plateauing, with headroom often remaining. Later iterations aren't always strictly better — sometimes a middle iteration is preferred over the final one.
- **Refine vs pivot**: Instruct the generator to make this a strategic decision. If scores are trending well, refine. If the approach isn't working, scrap it and try something entirely different. This is how creative breakthroughs happen — in one notable example, a museum website generator scrapped its clean dark-themed landing page on the 10th iteration and reimagined the site as a 3D spatial experience with CSS perspective rooms.
- **Evaluator interaction**: The evaluator should navigate the output directly, not score static snapshots. For web output, this means screenshotting pages, clicking through features, exercising interactions. This takes real wall-clock time but produces much better feedback.
- **Criteria language shapes output**: The wording of grading criteria directly steers the generator's aesthetic direction. "Museum quality" pushes toward a different convergence than "bold and experimental." Test different phrasings to find the character you want.

### When not to use this pattern

- When the task has clear, binary success criteria (compiles or doesn't, tests pass or don't) — a generator can self-evaluate these effectively
- When the quality bar is "functional, not polished" — the evaluator adds cost without proportional benefit
- When latency matters more than quality — each iteration adds wall-clock time

---

## Pattern 3: Three-Agent

**Use when:** The task requires both scope management (turning a vague prompt into a rich spec) and quality assurance (catching bugs and design issues the generator misses).

### Agent personas

**Planner agent** (runs once):
- Takes a 1-4 sentence user prompt and expands it into a full product spec
- Focuses on product context and high-level technical design, NOT granular implementation details
- Should be ambitious about scope — the planner's job is to envision the full product
- Can weave AI features into specs when appropriate
- Optionally reads existing design skills for visual direction

**Generator agent** (runs in sprints):
- Works in sprints, picking up features from the spec
- Before each sprint, negotiates a sprint contract with the evaluator
- Implements against the agreed contract
- Self-evaluates at the end of each sprint before handing off to QA
- Has git for version control

**Evaluator agent** (runs after each sprint):
- Tests the running application through interactive tools (Playwright MCP)
- Grades each sprint against both bugs found and grading criteria
- Each criterion has a hard threshold — falling below any one fails the sprint
- Provides specific, actionable feedback (not vague complaints)
- Findings should be granular enough to act on without extra investigation

### Sprint contract flow

```
Before Sprint N:
  generator proposes: "here's what I'll build and how success is verified"
  evaluator reviews: "that covers the spec, but add criterion X"
  iterate until agreed → sprint contract written to file

Sprint N:
  generator implements against the contract
  generator self-evaluates
  generator hands off to evaluator

QA for Sprint N:
  evaluator tests the running application
  evaluator grades against sprint contract criteria
  if any criterion below threshold → FAIL with detailed feedback → generator fixes
  if all pass → move to next sprint
```

### Communication files

All communication happens through files:

```
project/
├── spec.md              # Planner output: full product spec
├── sprint-contract-N.md # Agreed scope for sprint N
├── qa-report-N.md       # Evaluator findings for sprint N
├── progress.txt         # Running log of all sessions
└── features.json        # Feature list with pass/fail status
```

One agent writes, another reads and responds — either within the same file or with a new file. This keeps work faithful to the spec without agents needing to share a conversation.

### Example evaluator findings

Strong evaluator output looks like this:

| Contract criterion | Finding |
|---|---|
| Rectangle fill tool allows click-drag to fill area with selected tile | **FAIL** — Tool only places tiles at drag start/end instead of filling the region. `fillRectangle` exists but isn't triggered on mouseUp. |
| User can select and delete entity spawn points | **FAIL** — Delete handler requires both `selection` and `selectedEntityId`, but clicking an entity only sets `selectedEntityId`. |
| User can reorder animation frames via API | **FAIL** — PUT /frames/reorder route defined after /{frame_id} routes. FastAPI matches 'reorder' as frame_id integer → 422. |

Notice: specific file locations, specific code paths, specific technical causes. This is what the generator needs to fix the issue without investigation.

### Simplification path

As models improve, pieces of this architecture may become unnecessary:

1. **Sprint decomposition** may be removable if the model handles long coherent work natively (Opus 4.6 could work for 2+ hours without sprint structure)
2. **Context resets** may be removable if the model doesn't exhibit context anxiety (Opus 4.6 dropped this need)
3. **The evaluator** becomes less load-bearing for tasks within the model's reliable capability range, but still adds value at the edge of what the generator handles well solo

Test each removal individually and compare output quality.

---

## Choosing Between Patterns

| Your main problem | Pattern | Why |
|---|---|---|
| Work degrades across context boundaries | Two-agent | Structured handoffs solve context management |
| Output quality is mediocre or inconsistent | Generator-evaluator | External feedback drives quality improvement |
| Need both scope expansion and QA | Three-agent | Planner handles scope, evaluator handles quality |
| Task fits in one context, quality is fine | No harness | Don't add complexity you don't need |

## Combining Patterns

These patterns compose. You might use:
- A planner from Pattern 3 feeding into a generator-evaluator loop from Pattern 2
- A two-agent structure where the worker incorporates a self-contained generator-evaluator loop for each feature
- A three-agent system that drops to two-agent after a model upgrade makes the evaluator unnecessary for most sprints

The right architecture is the simplest one that addresses your observed failure modes.

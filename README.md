# Forge

**Design multi-agent systems for long-running autonomous tasks.**

Forge is a [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that guides you through designing agent architectures that sustain quality over hours of autonomous work — coding, design, content generation, or any domain requiring iteration.

## What it does

Forge encodes lessons from Anthropic's frontier research on harness design into an actionable workflow:

- **Three architecture patterns** — Two-agent (initializer + worker), generator-evaluator (GAN-inspired), and three-agent (planner + generator + evaluator). Choose based on your observed failure modes, not guesswork.
- **Grading criteria design** — Turn subjective quality judgments ("is this good?") into concrete, gradable dimensions. The highest-leverage part of any multi-agent system.
- **File-based communication** — Agents communicate through specs, sprint contracts, and evaluation reports — not conversational context. Everything persists across context resets.
- **Evaluator calibration** — A tuning loop for making your evaluator grade reliably: few-shot examples, skepticism prompting, and iterative alignment with your judgment.
- **Post-completion optimization** — Hand off to autoresearch when the harness delivers "done" but not "excellent." Grading criteria become the metric, the feature list becomes the guard.
- **Ruthless simplification** — Every component encodes an assumption about what the model can't do alone. Strip what's no longer load-bearing as models improve.

## Install

```bash
# Clone into your Claude Code skills directory
git clone https://github.com/omegalens/forge.git ~/.claude/skills/forge
```

Then use `/forge` in any Claude Code conversation.

## Usage

Invoke the skill when you need to:

- Design a multi-agent system for a complex task
- Create a generator-evaluator loop for subjective quality
- Set up autonomous coding pipelines that run for hours
- Decide whether you even need a harness (sometimes you don't)

```
/forge I want to build a system that autonomously generates full-stack web apps from a one-sentence prompt
```

## Skill contents

```
forge/
├── SKILL.md                              # Main skill — the full design workflow
├── references/
│   ├── architecture-patterns.md          # Detailed patterns with example prompts
│   ├── grading-criteria-guide.md         # How to write effective evaluation criteria
│   └── optimization-handoff.md           # Protocol for harness → autoresearch handoff
├── evals/
│   └── evals.json                        # Evaluation prompts for testing the skill
└── site/
    └── index.html                        # Project page
```

## Research sources

This skill synthesizes findings from:

1. **[Harness Design for Long-Running Application Development](https://www.anthropic.com/engineering/harness-design-long-running-apps)** — Prithvi Rajasekaran, Anthropic Labs (Mar 2026). The primary source: generator-evaluator loops, GAN-inspired architecture, grading criteria, sprint contracts.

2. **[Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)** — Anthropic Engineering (Nov 2025). The foundational two-agent pattern with structured handoffs.

3. **[Meta-Harness: End-to-End Optimization of Model Harnesses](https://yoonholee.com/meta-harness/)** — Lee et al., Stanford/UW-Madison (2026). Optimizing the harness itself via execution trace analysis.

4. **[Building Effective Agents](https://www.anthropic.com/research/building-effective-agents)** — Anthropic Research (2024). The simplification principle: find the simplest solution, increase complexity only when needed.

5. **[Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)** — Anthropic Engineering (2025). Context degradation research informing the context management section.

6. **[Generative Adversarial Networks](https://en.wikipedia.org/wiki/Generative_adversarial_network)** — Goodfellow et al., NeurIPS (2014). The conceptual ancestor of the generator-evaluator split.

## License

MIT

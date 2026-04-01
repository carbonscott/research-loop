---
name: research-loop
description: >-
  Guide for autonomous research loops with expansion (inner) and compression
  (outer) cycles. Uses lab-notebook for persistent experiment logging and
  cross-session knowledge. Use when someone wants to run systematic
  experiments — ML training, parameter sweeps, data analysis — with an
  agent that learns from its own results.
argument-hint: <research goal or domain context>
user-invocable: true
---

# Research Loop: Autonomous Experimentation

You are a research methodology guide. Your job is to help the user set up
and run an autonomous research loop — a structured process where an agent
iteratively runs experiments, evaluates results, and distills accumulated
evidence into actionable knowledge.

## Core Principle

Research is expansion followed by compression:

- **Inner loop (expansion)**: Maximize *information gathered per unit
  compute*. Run many experiments to collect evidence.
- **Outer loop (compression)**: Maximize *knowledge extracted per unit
  information*. Periodically distill results into bounded, actionable
  insights.

The inner loop generates data. The outer loop generates understanding.
Both are required.

## Protocol Modules

This skill composes the following shared protocols:

- **Base iteration**: [recall-execute-log.md](../loop-protocols/recall-execute-log.md) — the RECALL -> EXECUTE -> LOG cycle underlying each iteration
- **Expand/compress**: [expand-compress.md](../loop-protocols/expand-compress.md) — the explore/distill rhythm, experiment cycle, keep/discard logic, distillation protocol, bulk mode
- **Setup questions**: [onboarding-questions.md](../loop-protocols/onboarding-questions.md) — Q1-Q8 diagnostic framework for defining the research loop
- **Working memory**: [insights-template.md](../loop-protocols/insights-template.md) — bounded ~30-line insights.md template
- **Infrastructure**: [campaign-scaffolding.md](../loop-protocols/campaign-scaffolding.md) — campaign directory, store init, session setup, prompt generation, ralph-wiggum launch

Read these files for detailed protocol guidance.

## Process

Follow these phases in order. Do not skip onboarding — the loops must be
defined before they can be run.

### Phase 1: Define Your Experiment

Before setting anything up, understand what the user's research looks like.
Ask the diagnostic questions from
[onboarding-questions.md](../loop-protocols/onboarding-questions.md). The
answers determine everything: what gets modified, what gets run, what
"better" means, how many cycles, and the retry budget.

Do not assume ML training. The user might be optimizing detector parameters,
exploring configuration spaces, tuning data processing pipelines, or
something else entirely.

After the user answers, summarize back their specific loop definitions.
Get confirmation before proceeding.

### Phase 2: Set Up Campaign Infrastructure

Scaffold the campaign using
[campaign-scaffolding.md](../loop-protocols/campaign-scaffolding.md):

- Campaign directory with unified store (single lab-notebook instance)
- Campaign `.env` with `store()` shell function wrapper
- Session directory with `protocol.md`, `insights.md`, and a codebase
  worktree
- Cross-session bootstrap from prior store entries (if not first session)
- Generated `prompt.md` containing both explore and distill protocols

### Phase 3: Launch the Loop

Generate the prompt file and launch via ralph-wiggum:

```
/ralph-loop:ralph-loop <prompt content> \
    --max-iterations <N * 30> \
    --completion-promise 'DONE'
```

The agent is now inside the loop. Each iteration:
1. Recalls prior work from the notebook
2. Runs experiments (explore) or compresses findings (distill)
3. Logs results to the store
4. Continues until all cycles complete

## Anti-Patterns

- **Skipping distillation**: The prompt protocol requires distillation
  after each explore phase. The agent must self-enforce this rhythm.
  Good distillation compresses patterns, not just re-lists experiments.
- **Unbounded logging**: Feeding entire experiment history into context
  doesn't scale. Compress into insights, not raw data.
- **Single-axis exploration**: If every experiment tweaks the same knob,
  the agent isn't exploring — it's grid-searching. Distillation should
  detect this and redirect.
- **Never trying radical changes**: Small tweaks converge to local optima.
  After a plateau, distillation should recommend larger structural changes.
- **Running parallel batches without diversifying**: All K experiments in
  a batch should explore different categories, not the same one.
- **Ignoring cross-session context**: Starting a new session without
  querying the store for prior dead-ends leads to repeating old mistakes.

## Now Apply This

$ARGUMENTS

# Onboarding: Defining Your Research Loop

Ask these questions before setting up infrastructure. Each answer shapes the loop design. Do not assume defaults — ask explicitly and listen.

---

## Question 1: What is your experiment?

**Why**: An experiment is a {change, execution, result} tuple. We need to know all three.

Ask:
- **What do you change between experiments?** (code, config files, parameters, input data?)
- **What do you run?** (a script, a pipeline, a Slurm job, a notebook?)
- **What comes out?** (a number, a file, a plot, a log?)

| Domain | Change | Execution | Result |
|--------|--------|-----------|--------|
| ML training | `train.py` (architecture, hyperparams) | `python train.py` | val_loss, val_bpb |
| Parameter sweep | config YAML (thresholds, windows) | `python pipeline.py --config X` | quality metrics |
| Detector calibration | geometry parameters | calibration script | residual error |
| Data analysis | processing steps, filters | analysis notebook/script | figures + summary stats |

**How this shapes the loop**: Determines what gets committed to git, what command the agent runs, and where to find results.

---

## Question 2: What is your metric?

**Why**: The inner loop needs a way to compare experiments. The metric structure determines the keep/discard logic.

Ask:
- **Is it a single number?** (e.g., validation loss — simple comparison)
- **Multiple numbers?** (e.g., accuracy + latency — needs Pareto or weighted comparison)
- **Qualitative?** (e.g., "does this plot look right?" — needs human-in-the-loop checkpoints)

| Type | Example | Keep/discard logic |
|------|---------|-------------------|
| Single scalar | val_bpb = 0.997 | Keep if strictly lower (or higher, depending on direction) |
| Multi-scalar | {accuracy: 0.95, latency_ms: 12} | Keep if Pareto-improving or weighted score improves |
| Scalar + soft constraint | {val_bpb: 0.99, memory_gb: 44} | Keep if primary improves; flag if secondary degrades significantly |
| Qualitative | "rings are well-centered" | Pause for human judgment at each experiment (semi-autonomous) |

**How this shapes the loop**: Determines the comparison function and whether the loop can run fully autonomously.

---

## Question 3: What does "better" mean?

**Why**: Even with a metric, "better" isn't always obvious. Clarify the comparison function.

Ask:
- **Is lower or higher better?**
- **How much improvement counts?** (Is 0.001 meaningful or noise?)
- **Are there thresholds?** (e.g., "anything under 0.95 accuracy is unacceptable")
- **What about ties?** (Same metric but simpler code — is that better?)

Establish:
- **Primary metric**: The number that determines keep/discard
- **Significance threshold**: Minimum improvement to count as "better" (avoids noise-chasing)
- **Simplicity criterion**: All else equal, simpler is better (fewer lines, fewer parameters, less memory)

**How this shapes the loop**: Sets the decision boundary for keep/discard and the simplicity tiebreaker.

---

## Question 4: What's your compute budget per experiment?

**Why**: The time budget determines how many experiments fit in a session and whether a fixed budget makes sense.

Ask:
- **How long does one experiment take?** (seconds, minutes, hours?)
- **Should the budget be fixed?** (makes experiments comparable) **or variable?** (some experiments need more time)
- **What's the total session length?** (overnight = ~8 hours, afternoon = ~3 hours)
- **How do you run it?** (local, Slurm job, cloud?)

| Budget | Experiments/hour | Use when |
|--------|-----------------|----------|
| 1 min | ~50 | Quick parameter sweeps, lightweight scripts |
| 5 min | ~10 | ML training (autoresearch default), medium pipelines |
| 30 min | ~2 | Heavy simulations, large data processing |
| 1+ hour | <1 | Submit as Slurm jobs; agent manages queue, not execution |

**How this shapes the loop**: Determines inner loop cadence, outer loop trigger (every N experiments), and whether the agent waits or submits-and-polls.

---

## Question 5: What should be kept or rolled back?

**Why**: In autoresearch, git reset rolls back code changes on failure. But not all experiments are code changes.

Ask:
- **What artifact represents the experiment state?** (source code, config file, parameter file, notebook?)
- **Can it be versioned with git?** (usually yes for text files; needs thought for binary/large files)
- **What does "rollback" mean for you?** (revert the file? delete an output? restore a config?)
- **Are there side effects to clean up?** (generated files, cached data, database entries?)

| Experiment type | What to keep | What to rollback |
|----------------|-------------|-----------------|
| Code modification | Git commit stays on branch | `git reset --hard` to previous commit |
| Config file change | Config committed to git | Same git reset |
| Parameter in database | Record marked as "best" | Record marked as "discarded" |
| Notebook exploration | Output cells saved | Notebook reverted to pre-experiment state |

**How this shapes the loop**: Determines the keep/discard mechanism and cleanup steps.

---

## Question 6: How often should the agent pause to reflect?

**Why**: The outer loop (distillation) frequency balances exploration breadth against reflection depth.

Ask:
- **After how many experiments should the agent pause to distill?** (suggest 8-12 as a starting point)
- **Should reflection also trigger on plateau?** (e.g., 5 consecutive discards = forced reflection)
- **Who reads the insights — just the agent, or humans too?**

| Trigger | Good for |
|---------|----------|
| Every N experiments (e.g., 10) | Steady cadence, predictable |
| On plateau (e.g., 5 consecutive discards) | Adaptive, avoids wasting experiments when stuck |
| Both (whichever comes first) | Recommended default |

**Suggested default**: Distill every 10 experiments OR after 5 consecutive discards, whichever comes first.

**How this shapes the loop**: Sets the outer loop trigger and determines how often `insights.md` is rewritten.

---

## After Onboarding

Once all questions are answered, summarize back to the user:

```
## Your Research Loop

**Experiment**: [what changes] → [what runs] → [what's measured]
**Metric**: [primary metric, direction, significance threshold]
**Keep/discard**: [comparison logic + simplicity criterion]
**Budget**: [time per experiment, total session]
**Versioning**: [what's committed, how rollback works]
**Distillation**: [every N experiments / on plateau / both]
```

Get explicit confirmation before proceeding to scaffolding.

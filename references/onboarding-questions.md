# Onboarding: Defining Your Research Loop

Each campaign may have multiple sessions. The onboarding questions below define the protocol for a single session. If this is a new session within an existing campaign, review the campaign store for prior dead-ends and decisions before onboarding.

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
| 5 min | ~10 | ML training (research loop default), medium pipelines |
| 30 min | ~2 | Heavy simulations, large data processing |
| 1+ hour | <1 | Submit as Slurm jobs; agent manages queue, not execution |

**How this shapes the loop**: Determines inner loop cadence, how many experiments fit per explore phase, and whether the agent waits or submits-and-polls.

---

## Question 5: What should be kept or rolled back?

**Why**: In the research loop, git reset rolls back code changes on failure. But not all experiments are code changes.

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

## Question 6: How many explore-distill cycles?

**Why**: The research loop alternates between exploration (running experiments) and distillation (compressing results into insights). Lisa-wiggum's cursor enforces this rhythm mechanically — the agent cannot skip distillation. This question determines how many full cycles to run.

Ask:
- **How many explore-distill cycles do you want?** (suggest 5 as a starting point)
- **Within each explore phase, roughly how many experiments before distilling?** (suggest 8-12; this is guidance, not a hard limit — the agent decides when to signal readiness)

| Cycles | Total experiments (at ~10/cycle) | Good for |
|--------|----------------------------------|----------|
| 3 | ~30 | Quick exploration, short session |
| 5 (default) | ~50 | Standard research campaign |
| 10 | ~100 | Deep exploration, overnight runs |

**Within each explore phase**, the agent still uses heuristics to decide when to stop exploring and signal readiness to distill:
- After roughly N experiments (the per-cycle target above)
- On plateau (e.g., 5 consecutive discards — a signal that distillation is needed now, not after N more experiments)

**Suggested default**: 5 cycles, ~10 experiments per explore phase.

**How this shapes the loop**: Maps to `--dim cycle 1 2 3 4 5`. Combined with `--dim mode explore distill` and `--dim retry 0..<M>` (from Q8), this creates the full cursor. The agent decides how many experiments to run within each explore cell.

---

## Question 7: How should this session run — sequential or bulk?

**Why**: Each session runs one series of exploration. Sequential sessions run one experiment at a time. Bulk sessions run K experiments in parallel per batch using git worktrees. The choice depends on available compute.

Ask:
- **How many experiments can you run at once?** (number of available GPUs, Slurm slots, or CPU cores)
- **How do you submit parallel jobs?** (Slurm `sbatch`, background processes, task runners?)
- **Are experiments independent?** (Each modifies the same codebase differently — they don't need to talk to each other?)

| Batch size K | Mode | Best for |
|-------------|------|----------|
| 1 (default) | Sequential | Single GPU, interactive exploration, learning the loop |
| 2-4 | Small batch | Multi-GPU node, moderate parallelism |
| 8+ | Large batch | Slurm cluster, overnight sweeps with many nodes |

**If K=1**: Sequential mode. The agent runs one experiment at a time on the session branch.

**If K>1**: Bulk mode. The inner loop generates K diverse hypotheses per batch, runs them in parallel via git worktrees, evaluates all K, merges the best winner into the session branch, and logs everything to the campaign store with a shared `batch_id`.

**How this shapes the loop**: Determines whether the inner loop runs sequentially or in batches. The cursor structure is unaffected — both modes operate within `mode=explore` cells.

---

## Question 8: How many retries if an explore phase fails?

**Why**: Sometimes an entire explore phase goes off the rails — the codebase enters a broken state, persistent OOM blocks all experiments, or a bad hypothesis corrupts the setup. The retry budget gives the agent additional attempts at the same cycle's exploration before forcing distillation.

Ask:
- **If the agent finds itself unable to run any experiments (codebase broken, environment unrecoverable), how many fresh explore attempts before forcing distillation?** (suggest 2)
- Note: individual experiment crashes (one OOM, one bad architecture) are handled within the explore phase — the agent logs the crash and moves to the next hypothesis. Retries are for when the *entire* phase is unrecoverable.

| Retry budget | Good for |
|-------------|----------|
| 0 (none) | PHASE FAILED immediately forces distillation; robust environments only |
| 2 (default) | Standard; gives the agent a couple of fresh attempts to recover the phase |
| 3-5 | Fragile environments where phase-level failures are common |

**This maps to a cursor dimension.** The retry budget is encoded as `--dim retry 0..<M>` (e.g., 2 retries → `--dim retry 0 1 2`). The stop hook advances the retry dimension on PHASE FAILED. After exhausting all retries, overflow advances `mode` — so even total failure eventually reaches distillation.

**Suggested default**: 2 retries (→ `--dim retry 0 1 2`). With 0 retries, use `--dim retry 0` (1 value; PHASE FAILED immediately overflows to advance mode).

**How this shapes the loop**: Encoded in the cursor structure. The agent signals PHASE FAILED when the entire explore phase is unrecoverable (not for individual crashes); the stop hook advances the retry dimension automatically.

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
**Cycles**: [N explore-distill cycles, ~M experiments per explore phase]
**Session mode**: [sequential (K=1) / bulk (K=<n>)]
**Retry budget**: [M retries per explore phase, or none]
**Cursor**: --dim cycle 1..<N> --dim mode explore distill --dim retry 0..<M>
```

Get explicit confirmation before proceeding to scaffolding.

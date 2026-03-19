# Outer Loop Protocol

The outer loop's job is to **maximize knowledge extracted per unit information**. It compresses N experiments worth of data into a bounded summary that guides the next round.

## When It Triggers

The outer loop runs when either condition is met (whichever comes first):
- **Cadence trigger**: Every N experiments (default: 10)
- **Plateau trigger**: M consecutive discards (default: 5)

When triggered, the inner loop pauses. The agent switches to reflection mode.

## The Distillation Cycle

```
1. READ     the full experiment log (SQLite query)
2. ANALYZE  patterns across experiments
3. WRITE    insights.md (overwrite, not append — bounded size)
4. PLAN     strategy for next inner loop batch
5. RESUME   inner loop with updated insights
```

## Step Details

### 1. Read the Experiment Log

Query the SQLite database for all experiments since the last distillation (or all experiments if this is the first distillation):

```sql
-- All experiments since last distillation
SELECT id, change_type, description, status, metrics, notes
FROM experiments
WHERE id > (SELECT COALESCE(MAX(after_exp), 0) FROM distillations)
ORDER BY id;

-- Summary statistics
SELECT
  status,
  COUNT(*) as count,
  change_type,
  GROUP_CONCAT(description, '; ') as descriptions
FROM experiments
WHERE id > (SELECT COALESCE(MAX(after_exp), 0) FROM distillations)
GROUP BY status, change_type;
```

### 2. Analyze Patterns

**Batch-specific queries** (when batch_id is present):

```sql
-- Keep rate per batch (are batches getting better over time?)
SELECT batch_id, COUNT(*) as n,
       SUM(status IN ('keep', 'keep-deferred')) as kept,
       SUM(status = 'discard') as discarded
FROM experiments
WHERE batch_id IS NOT NULL
GROUP BY batch_id;

-- Were deferred keeps worth revisiting?
SELECT id, description, metrics, notes FROM experiments
WHERE status = 'keep-deferred';

-- Batch diversity check (flag batches that only explored one category)
SELECT batch_id, COUNT(DISTINCT change_type) as categories, COUNT(*) as n
FROM experiments WHERE batch_id IS NOT NULL
GROUP BY batch_id HAVING categories < 2;
```

Look for:

**What categories were explored?**
- Group experiments by `change_type` (architecture, hyperparameter, optimizer, etc.)
- Are some categories over-represented? Under-represented?

**What worked and why?**
- Among kept experiments, what do they have in common?
- Is there a trend? (e.g., "larger models consistently help")

**What failed and what does that rule out?**
- Among discarded experiments, are there repeated failures in one category?
- "Tried X three different ways, all failed" is a strong signal to stop trying X.

**What's the trajectory?**
- Is the metric still improving? Flattening? Oscillating?
- How does the current best compare to the baseline?

**Are we stuck?**
- If the last several experiments were all discards, we're in a local optimum.
- Consider: is the metric nearly converged, or are we just searching in the wrong direction?

**Deferred keeps** (batch mode):
- Are there `keep-deferred` experiments that showed promise but weren't the batch winner?
- Could a deferred keep be combined with the winner? (e.g., winner changed architecture, deferred changed optimizer — try both together)
- Should any deferred keep be revisited by rebasing on the current HEAD and re-testing?

Surface deferred keep findings in `insights.md` under **Current Strategy** (e.g., "consider combining batch 3 winner with deferred keep #12 which improved optimizer") or **Open Questions** if uncertain.

### 3. Write insights.md

Overwrite `insights.md` with a fresh summary. **This file must never exceed ~30 lines.** It is the agent's bounded working memory — if it grows unbounded, it defeats the purpose.

Template:

```markdown
# Research Insights (updated after experiment #N)

## Current State
- Best metric: [value] (experiment #X)
- Baseline: [value]
- Total improvement: [value] ([percent]%)
- Experiments run: [total] (kept: [n], discarded: [n], crashed: [n])

## What Works
- [Category]: [brief explanation of what helped and why]
- [Category]: [brief explanation]

## What's Ruled Out
- [Category]: Tried [N] variations, none improved. Likely not the bottleneck.
- [Category]: [explanation]

## Current Strategy
[2-3 sentences: what to focus on next and why, based on the patterns above]

## Open Questions
- [Anything uncertain that could be tested]
```

### 4. Record the Distillation

```sql
INSERT INTO distillations (timestamp, after_exp, summary, strategy)
VALUES ('<now>', <last_experiment_id>, '<insights summary>', '<next strategy>');
```

This creates a history of how the agent's understanding evolved over time.

### 5. Resume the Inner Loop

Return to the inner loop. The next hypothesis should be informed by the freshly written `insights.md`. The agent reads it at step 1 of the inner loop cycle.

## What Good Distillation Looks Like

**Good**: "Depth increases consistently improve BPB (3 out of 3 kept), but we're now at the memory limit. The next win is likely memory optimization — reduce batch size or use gradient checkpointing to unlock depth 12."

**Bad**: "Experiment 14 tried depth 10 and got 0.993. Experiment 15 tried depth 12 and crashed. Experiment 16 tried..." (This is just re-listing the log, not compressing it.)

**Good**: "Activation function changes (GeLU, SiLU, Swish) have been tried 4 times with no improvement. The MLP is not the bottleneck — stop exploring this direction."

**Bad**: "GeLU didn't work. SiLU didn't work." (Missing the generalization — *why* it didn't work and what that implies.)

## Handling Plateaus

If distillation is triggered by a plateau (consecutive discards), the strategy section should explicitly address it:

1. **Diagnose**: Why are experiments failing? Same category? Different categories all failing?
2. **Pivot**: If small tweaks aren't working, recommend a larger structural change.
3. **Combine**: Look at near-misses from the log. Two changes that each almost worked might work together.
4. **Challenge assumptions**: Is the metric still the right one? Is the time budget hiding something?

The plateau response is the most valuable part of the outer loop — it's what prevents the agent from endlessly trying variations of the same failing idea.

# Outer Loop Protocol

The outer loop's job is to **maximize knowledge extracted per unit information**. It compresses N experiments worth of data into a bounded summary that guides the next round.

## When It Triggers

The outer loop runs when either condition is met (whichever comes first):
- **Cadence trigger**: Every N experiments (default: 10)
- **Plateau trigger**: M consecutive discards (default: 5)

When triggered, the inner loop pauses. The agent switches to reflection mode.

## The Distillation Cycle

```
1. READ     the logbook (query via logbook sql)
2. ANALYZE  patterns across experiments
3. WRITE    insights.md (overwrite, not append — bounded size)
4. RECORD   distillation to logbook + insights snapshot + narrative to notebook
5. RESUME   inner loop with updated insights
```

## Step Details

### 1. Read the Logbook

Query the logbook for all experiments since the last distillation (or all experiments if this is the first distillation):

```bash
# All experiments since last distillation
logbook sql "SELECT ts, change_type, status, metrics, substr(content,1,80)
    FROM entries WHERE context='$SESSION' AND type IN ('experiment','baseline')
    AND ts > COALESCE(
        (SELECT MAX(ts) FROM entries WHERE type='distillation' AND context='$SESSION'),
        '')  -- empty string compares less than any timestamp
    ORDER BY ts"

# Summary statistics
logbook sql "SELECT status, COUNT(*) as count, change_type,
        GROUP_CONCAT(substr(content,1,40), '; ') as descriptions
    FROM entries WHERE context='$SESSION' AND type='experiment'
    AND ts > COALESCE(
        (SELECT MAX(ts) FROM entries WHERE type='distillation' AND context='$SESSION'),
        '')  -- empty string compares less than any timestamp
    GROUP BY status, change_type"
```

### 2. Analyze Patterns

**Batch-specific queries** (when batch_id is present):

```bash
# Keep rate per batch (are batches getting better over time?)
logbook sql "SELECT batch_id, COUNT(*) as n,
        SUM(status IN ('keep', 'keep-deferred')) as kept,
        SUM(status = 'discard') as discarded
    FROM entries WHERE context='$SESSION' AND batch_id IS NOT NULL
    GROUP BY batch_id"

# Were deferred keeps worth revisiting?
logbook sql "SELECT ts, substr(content,1,80), metrics
    FROM entries WHERE context='$SESSION' AND status='keep-deferred'"

# Batch diversity check (flag batches that only explored one category)
logbook sql "SELECT batch_id, COUNT(DISTINCT change_type) as categories, COUNT(*) as n
    FROM entries WHERE context='$SESSION' AND batch_id IS NOT NULL
    GROUP BY batch_id HAVING categories < 2"
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

**Deferred keeps** (bulk mode):
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

### 4. Record to Logbook AND Notebook

**Always** record the distillation to the logbook (operational record):

```bash
logbook emit --context "$SESSION" --type distillation \
    --strategy "Focus on depth increases; stop exploring activation functions" \
    "After 10 experiments: depth helps (3/3 kept), activation changes all failed (0/4). Best val_bpb=0.91 vs baseline 1.012."
```

**Also** record the full insights snapshot and significant findings to the campaign notebook (narrative record, visible to future sessions):

```bash
# Always: snapshot the full insights.md for cross-session history
notebook emit --context "$SESSION" --type observation \
    --tags distillation,insights-snapshot \
    "$(cat insights.md)"
```

The insights snapshot above is mandatory every distillation — it preserves the full evolution of the agent's understanding. The remaining notebook entry types below are selective — write what is significant for cross-session knowledge: dead-ends (prevent repeating), decisions (explain direction), milestones (track progress).

```bash
# Record dead-ends so future sessions don't repeat them
notebook emit --context "$SESSION" --type dead-end \
    --tags activation,architecture \
    "Activation function changes (GeLU, SiLU, Swish) tried 4 times with no improvement. The MLP is not the bottleneck."

# Record key decisions
notebook emit --context "$SESSION" --type decision \
    --tags strategy \
    "Focusing on depth increases. Architecture gains are consistent; memory optimization needed to unlock depth 12."

# Record milestones
notebook emit --context "$SESSION" --type milestone \
    "Best val_bpb=0.91, 10% improvement over baseline after 10 experiments."
```

Not every distillation needs all of these selective entry types — write what is significant.

### 5. Resume the Inner Loop

Return to the inner loop. The next hypothesis should be informed by the freshly written `insights.md`. The agent reads it at step 1 of the inner loop cycle.

## Cross-Session Bootstrap

When starting a new session within the same campaign, the agent should query the notebook before running the first experiment:

```bash
# What dead-ends exist from prior sessions?
notebook sql "SELECT ts, context, substr(content,1,100)
    FROM entries WHERE type='dead-end' ORDER BY ts DESC LIMIT 20"

# What strategic decisions have been made?
notebook sql "SELECT ts, context, substr(content,1,100)
    FROM entries WHERE type='decision' ORDER BY ts DESC LIMIT 20"

# What milestones have been reached?
notebook sql "SELECT ts, context, substr(content,1,100)
    FROM entries WHERE type='milestone' ORDER BY ts DESC LIMIT 10"

# Latest insights snapshot from each prior session
notebook sql "SELECT context, ts, substr(content,1,200)
    FROM entries WHERE type='observation'
    AND tags LIKE '%insights-snapshot%'
    GROUP BY context ORDER BY ts DESC LIMIT 5"
```

Use these results to pre-seed the new session's `insights.md` with prior knowledge. This prevents repeating experiments that were already ruled out in earlier sessions.

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

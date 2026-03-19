# Inner Loop Protocol

The inner loop's job is to **maximize information gathered per unit compute**. Each cycle is one experiment.

## The Cycle

```
1. READ    insights.md (if it exists — skip on first run)
2. HYPOTHESIZE  what to try next (informed by insights + current state)
3. MODIFY  the experiment artifact (code, config, parameters)
4. COMMIT  to git (atomic snapshot, enables rollback)
5. RUN     the experiment (redirect output to log file)
6. EVALUATE  extract metrics from output
7. DECIDE  keep or discard (based on comparison function)
8. LOG     record to SQLite (regardless of outcome)
9. REPEAT  go to step 1
```

## Step Details

### 1. Read insights

Before each hypothesis, read `insights.md` if it exists. This is the compressed knowledge from the outer loop. It tells you:
- What categories of changes have been tried
- What worked and the agent's theory of why
- What's been ruled out
- The current strategy for the next batch

On the very first experiment, there are no insights yet. Run the baseline as-is to establish the reference point.

### 2. Hypothesize

Based on insights (or initial intuition), decide what to try. Good hypotheses:
- Are specific ("increase depth from 8 to 10" not "make the model bigger")
- Are motivated ("depth helped last time, and we freed memory by reducing batch size")
- Vary in scope (mix small tweaks with occasional larger structural changes)

### 3. Modify

Make the change to the experiment artifact. Keep changes atomic — one idea per experiment. If you want to try two things, run two experiments.

### 4. Commit

```bash
git add <modified files>
git commit -m "<short description of what this experiment tries>"
```

The commit message should be descriptive enough to understand the experiment from `git log` alone. This is your audit trail.

### 5. Run

```bash
<run command> > run.log 2>&1
```

Always redirect output to a log file. Do NOT let output flood the agent's context. Read specific lines from the log to extract results.

**Timeout**: If the experiment exceeds the expected budget by more than 2x, kill it and treat as a crash.

### 6. Evaluate

Extract metrics from the log. The exact method depends on the domain:

```bash
# Example: grep for specific metric lines
grep "^val_bpb:" run.log
grep "^peak_vram_mb:" run.log

# Example: parse JSON output
python -c "import json; d=json.load(open('results.json')); print(d['accuracy'])"

# Example: check exit code
echo $?
```

If no metrics are found, the run crashed. Read the tail of the log for diagnostics:
```bash
tail -n 50 run.log
```

### 7. Decide: Keep or Discard

**For single scalar metrics:**
```
IF metric improved beyond significance threshold → KEEP
IF metric is within noise range → KEEP only if change is simpler
IF metric is worse → DISCARD
```

**For multi-scalar metrics:**
```
IF primary metric improved AND secondary didn't degrade significantly → KEEP
IF primary improved but secondary degraded → judgment call (log the tradeoff)
IF primary didn't improve → DISCARD
```

**On KEEP:**
- The git commit stays. The branch advances.
- This is now the new baseline for comparison.

**On DISCARD:**
- Roll back: `git reset --hard HEAD~1` (or appropriate rollback for your setup)
- The experiment is still logged — discarded experiments carry information.

**On CRASH:**
- Read the error. If it's a trivial fix (typo, import), fix and re-run.
- If the idea is fundamentally broken (OOM, incompatible architecture), log as crash and move on.
- Don't spend more than 2-3 attempts fixing a crash before abandoning.

### 8. Log to SQLite

Every experiment gets logged, regardless of outcome:

```sql
INSERT INTO experiments (commit_hash, timestamp, change_type, description, status, metrics, notes)
VALUES ('<hash>', '<now>', '<category>', '<what was tried>', '<keep|discard|crash>', '<json metrics>', '<why it worked or failed>');
```

The `notes` field is important — it captures the agent's interpretation, not just the numbers. This is what the outer loop reads during distillation.

## Baseline Run

The first experiment is always a baseline: run the existing code/config without changes. This establishes:
- The reference metric value
- Expected runtime
- Memory/resource usage
- That the setup works at all

Log it as `status: keep, description: baseline`.

## When to Trigger the Outer Loop

After completing N inner loop cycles (defined during onboarding), or after detecting a plateau (M consecutive discards), pause the inner loop and switch to the outer loop protocol.

Do NOT ask the user "should I continue?" — the loop is autonomous. The only pause is for distillation, and then the inner loop resumes.

---

## Batch Mode (Optional)

When the protocol specifies batch size K > 1, the inner loop runs K experiments in parallel using git worktrees instead of one at a time. **If K=1, ignore this section entirely — use the sequential cycle above.**

**Baseline first**: Run the baseline as a single sequential experiment before starting batch mode. The baseline establishes the reference metric for keep/discard decisions across all batches.

### The Batch Cycle

```
1.  READ       insights.md (if it exists)
2.  HYPOTHESIZE K times (diversify across change_type categories)
3.  SETUP      K worktrees from current research branch HEAD
4.  MODIFY     each worktree independently (one change per worktree)
5.  COMMIT     in each worktree
6.  RUN        K experiments in parallel (sbatch, background processes, etc.)
7.  WAIT       for all K to complete
8.  EVALUATE   each independently (extract metrics, decide keep/discard)
9.  LOG        all K to SQLite with shared batch_id
10. RECONCILE  merge best keep into research branch, clean up worktrees
11. CHECK      total experiments += K; if >= N → trigger outer loop, else next batch
```

### Step Details (batch-specific)

#### 2. Hypothesize K times

Generate K hypotheses at once, informed by `insights.md`. The critical rule: **diversify across categories**. If insights says filters are ruled out, none of the K should be filter experiments. Spread across open categories:

```
Good batch (K=3):
  hyp-1: architecture change (replace local-max with DoG)
  hyp-2: hyperparameter change (reduce learning rate by 2x)
  hyp-3: preprocessing change (normalize per-panel)

Bad batch (K=3):
  hyp-1: try learning rate 0.001
  hyp-2: try learning rate 0.0005
  hyp-3: try learning rate 0.0001
  (this is just a grid search — run it as a parameter sweep instead)
```

If K > number of open categories, multiple per category are fine but should vary the specific approach.

#### 3. Setup worktrees

Create K worktrees, each branching from the current research branch HEAD:

```bash
BATCH_ID=$(sqlite3 research-loop/experiments.db \
  "SELECT COALESCE(MAX(batch_id), 0) + 1 FROM experiments;")
RESEARCH_BRANCH=$(git branch --show-current)

for i in $(seq 1 $K); do
  git worktree add research-loop/worktrees/exp-${BATCH_ID}-${i} \
    -b exp/${BATCH_ID}-${i} ${RESEARCH_BRANCH}
done
```

Each worktree gets an independent copy of the codebase. Changes in one don't affect others.

#### 5. Commit in each worktree

```bash
(cd research-loop/worktrees/exp-${BATCH_ID}-${i} && \
  git add <modified files> && \
  git commit -m "exp: batch ${BATCH_ID} hyp ${i} — <description>")
```

#### 6. Run in parallel

Submit all K experiments. The exact method depends on the environment:

```bash
# Slurm
for i in $(seq 1 $K); do
  (cd research-loop/worktrees/exp-${BATCH_ID}-${i} && \
    sbatch --job-name=exp-${BATCH_ID}-${i} run.sh)
done

# Background processes
for i in $(seq 1 $K); do
  (cd research-loop/worktrees/exp-${BATCH_ID}-${i} && \
    <run command> > run.log 2>&1) &
done
wait
```

#### 7. Wait

For Slurm: poll `squeue` until all batch jobs complete. For background processes: `wait` blocks until all finish.

#### 9. Log with batch_id

```sql
INSERT INTO experiments (commit_hash, change_type, description, status, metrics, notes, batch_id)
VALUES ('<hash>', '<type>', '<description>', '<keep|discard|crash>', '<json>', '<notes>', <BATCH_ID>);
```

All K experiments in a batch share the same `batch_id`.

#### 10. Reconcile

This is the key step that differs from sequential mode. After evaluating all K experiments:

**If 0 keeps**: Research branch unchanged. All logged as discards. Move to next batch (or trigger outer loop if plateau detected — count K discards toward the consecutive discard counter).

**If 1 keep**: Merge that experiment's branch into the research branch:
```bash
git merge --ff-only exp/${BATCH_ID}-${WINNER}
```

**If multiple keeps**: Merge only the **best** one (highest primary metric improvement). Log the others as `keep-deferred` — they beat the baseline but weren't the batch winner. The outer loop can flag deferred keeps as promising directions.

```sql
-- The winner
UPDATE experiments SET status = 'keep'
WHERE batch_id = <BATCH_ID> AND id = <winner_id>;

-- Other keeps that weren't merged
UPDATE experiments SET status = 'keep-deferred'
WHERE batch_id = <BATCH_ID> AND status = 'keep' AND id != <winner_id>;
```

**After merging**, clean up all worktrees:
```bash
for i in $(seq 1 $K); do
  git worktree remove research-loop/worktrees/exp-${BATCH_ID}-${i}
  git branch -D exp/${BATCH_ID}-${i}
done
```

**Why "winner takes main"**: All worktrees branched from the same HEAD, so the winner's merge is always a clean fast-forward. Merging multiple keeps risks conflicts (two experiments modifying the same file differently) and produces untested combinations. Keep it simple — let the outer loop suggest combining promising directions in a future batch.

### Plateau Detection in Batch Mode

In sequential mode, M consecutive discards triggers the outer loop. In batch mode, count at the batch level:
- A batch with 0 keeps counts as 1 "batch discard"
- M consecutive batch discards (not individual discards) triggers the outer loop
- A batch with at least 1 keep resets the consecutive discard counter

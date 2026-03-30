# Inner Loop Protocol (Explore Phase)

This protocol applies when the system message shows `mode=explore`. The cursor position (e.g., `Cursor: cycle=2, mode=explore, retry=0`) tells you which round of exploration you are in. `retry=0` is the first attempt; `retry=1` means you are on a retry after a prior failed phase.

The inner loop's job is to **maximize information gathered per unit compute**. Each iteration of the cycle below is one experiment.

## The Cycle

```
1. READ    insights.md (if it exists — skip on first run)
2. HYPOTHESIZE  what to try next (informed by insights + current state)
3. MODIFY  the experiment artifact (code, config, parameters)
4. COMMIT  to git (atomic snapshot, enables rollback)
5. RUN     the experiment (redirect output to log file)
6. EVALUATE  extract metrics from output
7. DECIDE  keep or discard (based on comparison function)
8. LOG     record to store (regardless of outcome)
9. REPEAT  go to step 1
```

## Step Details

### 1. Read insights

Before each hypothesis, read `insights.md` if it exists. This is the compressed knowledge from the last distillation phase. It tells you:
- What categories of changes have been tried
- What worked and the agent's theory of why
- What's been ruled out
- The current strategy for the next batch

Your cursor position is in the system message (e.g., `Cursor: cycle=2, mode=explore, retry=0`). If `retry > 0`, a prior explore phase failed and this is a fresh attempt. If this is cycle 2+, insights.md was updated during the previous cycle's distillation and reflects accumulated knowledge.

On the very first experiment (cycle 1, first iteration), there are no insights yet. Run the baseline as-is to establish the reference point.

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
git commit -m "exp: <what this experiment tries>"
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
- Read the error. If it's a trivial fix (typo, import), fix and re-run within this iteration.
- If the idea is fundamentally broken (OOM, incompatible architecture), log as crash and move to the next hypothesis. A single crashed experiment does not warrant PHASE FAILED.
- Only signal PHASE FAILED if the codebase itself is broken such that no further experiments can proceed this explore phase.

### 8. Log to Store

Every experiment gets logged, regardless of outcome:

```bash
store emit --context "$SESSION" --type experiment \
    --commit_hash "$(git rev-parse --short HEAD)" \
    --change_type "<category>" \
    --status "<keep|discard|crash>" \
    --metrics '<json metrics>' \
    "What was tried and why it worked or failed."
```

The `content` field (the positional argument) is important — it captures what was tried AND the agent's interpretation, not just the numbers. This is what the outer loop reads during distillation.

## Baseline Run

The first experiment is always a baseline: run the existing code/config without changes. This establishes:
- The reference metric value
- Expected runtime
- Memory/resource usage
- That the setup works at all

Log it with `store emit --context "$SESSION" --type baseline --status keep ...`.

## Signals

The cursor manages the explore/distill rhythm. Your job is to signal when you are ready to transition.

**PHASE COMPLETE** — Include this in your response when you have run enough experiments and are ready to distill. The cursor advances from `mode=explore` to `mode=distill`. Heuristics for when to signal:
- You have reached the per-cycle experiment target (from onboarding)
- You detect a plateau (M consecutive discards — distillation will help redirect)
- You have enough evidence for a meaningful distillation (patterns are visible)

**No signal** — End your response without any signal keyword. The cursor stays at the same position. The next iteration continues exploration at the same cycle and mode. This is the normal case within an explore phase: one experiment per iteration, no signal until you are ready to distill.

**PHASE FAILED** — Use sparingly. Include this only if the current explore attempt is unrecoverable (e.g., codebase is broken beyond repair, persistent OOM). This advances the `retry` dimension (e.g., retry=0→1). If retries remain, you get another attempt at the same (cycle, mode) position. If retries are exhausted, overflow advances `mode` to distill — so the agent can at least record what happened. For individual experiment failures that you can work around, just log them and move to the next hypothesis — do not signal PHASE FAILED.

Do NOT ask the user "should I continue?" — the loop is autonomous. The only pause is for distillation, and then exploration resumes.

---

## Bulk Mode (Optional)

When the protocol specifies bulk mode (K > 1), the inner loop runs K experiments in parallel using git worktrees instead of one at a time. **If K=1 (sequential), ignore this section entirely — use the sequential cycle above.**

Bulk mode operates within a single cursor position. The cursor does not model individual parallel experiments — whether you run 1 sequential experiment or K parallel experiments per iteration, the cursor stays at `mode=explore` until you signal PHASE COMPLETE.

**Baseline first**: Run the baseline as a single sequential experiment before starting bulk mode. The baseline establishes the reference metric for keep/discard decisions across all batches.

### The Bulk Cycle

```
1.  READ       insights.md (if it exists)
2.  HYPOTHESIZE K times (diversify across change_type categories)
3.  SETUP      K worktrees from current research branch HEAD
4.  MODIFY     each worktree independently (one change per worktree)
5.  COMMIT     in each worktree
6.  RUN        K experiments in parallel (sbatch, background processes, etc.)
7.  WAIT       for all K to complete
8.  EVALUATE   each independently (extract metrics, decide keep/discard)
9.  LOG        all K to store with shared batch_id
10. RECONCILE  merge best keep into session branch, clean up worktrees
11. CHECK      total experiments += K; if ready to distill → signal PHASE COMPLETE, else next batch
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

Create K worktrees, each branching from the current session branch HEAD:

```bash
BATCH_ID=$(store sql "SELECT COALESCE(MAX(batch_id), 0) + 1
    FROM entries WHERE context='$SESSION'" | awk 'NR==3{print $1}')
SESSION_BRANCH="research/$SESSION"
CODEBASE="$CAMPAIGN/sessions/$SESSION/codebase"
K=<batch size from protocol.md>

for i in $(seq 1 $K); do
  git worktree add "$CODEBASE/worktrees/exp-${BATCH_ID}-${i}" \
    -b "exp/${SESSION}/${BATCH_ID}-${i}" "$SESSION_BRANCH"
done
```

Each worktree gets an independent copy of the codebase. Changes in one don't affect others.

#### 5. Commit in each worktree

```bash
(cd "$CODEBASE/worktrees/exp-${BATCH_ID}-${i}" && \
  git add <modified files> && \
  git commit -m "exp: batch ${BATCH_ID} hyp ${i} — <description>")
```

#### 6. Run in parallel

Submit all K experiments. The exact method depends on the environment:

```bash
# Slurm
for i in $(seq 1 $K); do
  (cd "$CODEBASE/worktrees/exp-${BATCH_ID}-${i}" && \
    sbatch --job-name=exp-${BATCH_ID}-${i} run.sh)
done

# Background processes
for i in $(seq 1 $K); do
  (cd "$CODEBASE/worktrees/exp-${BATCH_ID}-${i}" && \
    <run command> > run.log 2>&1) &
done
wait
```

#### 7. Wait

For Slurm: poll `squeue` until all batch jobs complete. For background processes: `wait` blocks until all finish.

#### 9–10. Evaluate, Determine Winner, Then Log All K

**Important**: The store is append-only — there is no UPDATE. Evaluate all K experiments and determine the batch winner BEFORE logging any of them. Then emit all K entries with their final status in one pass.

After all K experiments complete, evaluate each independently, then:

**If 0 keeps**: Log all K as `status discard`. Session branch unchanged.

**If 1 keep**: Log the winner as `status keep`, others as `status discard`.

**If multiple keeps**: Log the best one as `status keep`, other keeps as `status keep-deferred`, failures as `status discard`.

```bash
# Log each experiment with its final status
store emit --context "$SESSION" --type experiment \
    --commit_hash "<hash>" --change_type "<type>" \
    --status "<keep|keep-deferred|discard|crash>" \
    --metrics '<json>' --batch_id "$BATCH_ID" \
    "What was tried and why it worked or failed."
```

All K experiments in a batch share the same `batch_id`.

**Then merge the winner** into the session branch:
```bash
cd "$CODEBASE"
WINNER_INDEX=<index of best-performing worktree>
WINNER_BRANCH="exp/${SESSION}/${BATCH_ID}-${WINNER_INDEX}"
git merge --ff-only "$WINNER_BRANCH" \
    || git merge "$WINNER_BRANCH" -m "exp: merge batch ${BATCH_ID} winner"
```

**After merging**, clean up all worktrees:
```bash
for i in $(seq 1 $K); do
  git worktree remove "$CODEBASE/worktrees/exp-${BATCH_ID}-${i}"
  git branch -D "exp/${SESSION}/${BATCH_ID}-${i}"
done
```

**Why "winner takes main"**: All worktrees branched from the same HEAD, so the winner's merge is always a clean fast-forward. Merging multiple keeps risks conflicts (two experiments modifying the same file differently) and produces untested combinations. Keep it simple — let the outer loop suggest combining promising directions in a future batch.

### Plateau Detection in Bulk Mode

In sequential mode, M consecutive discards is a signal to stop exploring (signal PHASE COMPLETE). In bulk mode, count at the batch level:
- A batch with 0 keeps counts as 1 "batch discard"
- M consecutive batch discards (not individual discards) signals readiness to distill — include PHASE COMPLETE
- A batch with at least 1 keep resets the consecutive discard counter

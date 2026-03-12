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

# Scaffolding: Setting Up Campaign Infrastructure

**Prerequisite**: [lab-notebook](https://github.com/carbonscott/lab-notebook) must be installed and available on `$PATH`.

After onboarding, set up these concrete artifacts. Adapt the details based on the user's answers.

## Campaign Directory Structure

A campaign is the top-level research infrastructure. It contains a unified lab-notebook store and a sessions directory.

```
<campaign>/
├── .env                         # Shell function wrapper: store()
├── store/                       # All experiment data + narrative insights (single lab-notebook instance)
│   ├── schema.yaml              # Unified schema (structured + narrative entry types)
│   ├── entries/                 # Per-writer JSONL files (git-tracked)
│   │   ├── agent-seq.jsonl
│   │   ├── agent-wt-1.jsonl
│   │   └── ...
│   ├── index.sqlite             # Disposable query index (gitignored, rebuilt on demand)
│   └── .gitignore
└── sessions/
    ├── <session-name>/
    │   ├── protocol.md          # This session's loop definitions (from onboarding)
    │   ├── insights.md          # Bounded working memory (overwritten each distillation)
    │   ├── prompt.md            # Generated prompt file for lisa-wiggum
    │   └── codebase/            # Git worktree on branch research/<session-name>
    └── ...
```

The store holds both structured experiment data and narrative insights in one place:
- **Structured types** (`experiment`, `baseline`, `distillation`): metrics, status, change_type — the operational record
- **Narrative types** (`observation`, `decision`, `dead-end`, `milestone`): cross-session knowledge — observations, strategic pivots, ruled-out directions

The `--type` field differentiates them. Queries filter by type.

## Campaign Initialization

```bash
CAMPAIGN=/path/to/campaign

# 1. Create directory structure
mkdir -p "$CAMPAIGN"/{store,sessions}

# 2. Initialize the unified store from research-logbook template
lab-notebook init "$CAMPAIGN/store" --template research-logbook

# 3. Extend schema.yaml with narrative entry types and the tags field
#    (research-logbook provides: experiment, baseline, distillation types
#     and commit_hash, change_type, status, metrics, batch_id, strategy fields)
```

After `lab-notebook init`, edit `$CAMPAIGN/store/schema.yaml` to add narrative types and the `tags` field:

```yaml
# Add to the types list:
types:
  - experiment
  - baseline
  - distillation
  - observation      # insights snapshots, general observations
  - decision         # strategic pivots
  - dead-end         # ruled-out directions (prevents cross-session repeats)
  - milestone        # progress checkpoints

# Add to the fields section:
fields:
  commit_hash:  {type: text}
  change_type:  {type: text, fts: true}
  status:       {type: text}
  metrics:      {type: text}
  batch_id:     {type: integer}
  strategy:     {type: text, fts: true}
  tags:         {type: list}               # NEW: for categorizing narrative entries
```

Then rebuild the index to pick up the schema changes:

```bash
lab-notebook rebuild "$CAMPAIGN/store"
```

### Campaign .env

Create `$CAMPAIGN/.env` with a shell function wrapper. This is the key mechanism — the agent calls `store emit ...` or `store sql ...` without managing env vars:

```bash
# Campaign environment — source this before any research loop work
CAMPAIGN_DIR="<absolute path to campaign>"

store() { LAB_NOTEBOOK_DIR="$CAMPAIGN_DIR/store" lab-notebook "$@"; }

export CAMPAIGN_DIR
export -f store
```

After sourcing, usage is:

```bash
source "$CAMPAIGN/.env"

# Log an experiment (structured)
store emit --context my-session --type experiment ...

# Record a narrative insight
store emit --context my-session --type observation ...

# Query experiments
store sql "SELECT * FROM entries WHERE context='my-session'"
```

## Session Creation

Each session is one series of exploration. The codebase lives inside the session as a git worktree.

```bash
SESSION=mae-exploration
BASE_BRANCH=main   # or whatever branch to fork from

# 1. Create session directory
mkdir -p "$CAMPAIGN/sessions/$SESSION"

# 2. Create codebase as git worktree on a research branch
cd /path/to/source-repo
git worktree add "$CAMPAIGN/sessions/$SESSION/codebase" \
    -b "research/$SESSION" "$BASE_BRANCH"

# 3. Write protocol.md (from onboarding template below)
# 4. Write initial insights.md (empty template below)
# 5. Generate prompt.md (see Prompt File Generation below)
# 6. Launch lisa-wiggum (see Launch below)
```

### Session mode

The protocol.md records whether this session runs sequentially or in bulk:

- **Sequential**: One experiment at a time. The agent modifies code, commits, runs, evaluates — all on the session's research branch. No worktrees needed.
- **Bulk (K>1)**: K experiments per batch. The agent creates K ephemeral worktrees from the session branch, runs K experiments in parallel, reconciles, and cleans up.

The store doesn't care — both modes produce `experiment` entries with `context=<session-name>`. Bulk entries also set `batch_id`.

## Git Setup

### Commit conventions

Each experiment commit should be descriptive:

```
git commit -m "exp: <what this experiment tries>"
```

The `exp:` prefix makes research commits easy to find in git log.

### Branch strategy

- Each session gets branch `research/<session-name>`
- Sequential experiments are commits on this branch
- Bulk worktrees branch from this session branch per batch
- The main/master branch stays clean

## Store Helpers

All commands assume you have sourced the campaign `.env`.

### Log a baseline

```bash
store emit --context "$SESSION" --type baseline \
    --commit_hash "$(git rev-parse --short HEAD)" \
    --status keep \
    --metrics '{"val_bpb": 1.012, "memory_gb": 48}' \
    "Baseline run with unmodified code. Reference point for all comparisons."
```

### Log an experiment

```bash
store emit --context "$SESSION" --type experiment \
    --commit_hash "$(git rev-parse --short HEAD)" \
    --change_type architecture \
    --status keep \
    --metrics '{"val_bpb": 0.993, "memory_gb": 44}' \
    "Increased encoder depth from 8 to 10. Consistent improvement, freed memory by reducing batch size."
```

### Log a distillation

```bash
store emit --context "$SESSION" --type distillation \
    --strategy "Focus on depth increases; stop exploring activation functions" \
    "After 10 experiments: depth helps (3/3 kept), activation changes all failed (0/4). Best val_bpb=0.91 vs baseline 1.012."
```

### Query recent experiments

```bash
# All experiments since last distillation
store sql "SELECT ts, status, change_type, substr(content,1,60), metrics
    FROM entries WHERE context='$SESSION' AND type IN ('experiment','baseline')
    AND ts > COALESCE(
        (SELECT MAX(ts) FROM entries WHERE type='distillation' AND context='$SESSION'),
        '')  -- empty string compares less than any timestamp
    ORDER BY ts"

# Summary by category and status
store sql "SELECT change_type, status, COUNT(*) as n
    FROM entries WHERE context='$SESSION' AND type='experiment'
    GROUP BY change_type, status ORDER BY change_type"

# Best metric so far
store sql "SELECT json_extract(metrics, '$.val_bpb') as metric, substr(content,1,60)
    FROM entries WHERE context='$SESSION' AND status='keep'
    ORDER BY CAST(json_extract(metrics, '$.val_bpb') AS REAL) ASC LIMIT 1"
```

### Bulk mode -- create worktrees

```bash
BATCH_ID=$(store sql "SELECT COALESCE(MAX(batch_id), 0) + 1
    FROM entries WHERE context='$SESSION'" | awk 'NR==3{print $1}')
SESSION_BRANCH="research/$SESSION"
CODEBASE="$CAMPAIGN/sessions/$SESSION/codebase"
K=<batch size from protocol.md>

for i in $(seq 1 $K); do
  git -C "$CODEBASE" worktree add "worktrees/exp-${BATCH_ID}-${i}" \
      -b "exp/${SESSION}/${BATCH_ID}-${i}" "$SESSION_BRANCH"
done
```

**Note**: Each worktree duplicates working tree files (the object store is shared via hardlinks). For large repos with K=8+, ensure sufficient disk for K concurrent checkouts.

### Bulk mode -- teardown worktrees

```bash
for i in $(seq 1 $K); do
  git worktree remove "$CODEBASE/worktrees/exp-${BATCH_ID}-${i}"
  git branch -D "exp/${SESSION}/${BATCH_ID}-${i}"
done
```

### Bulk mode -- merge winner

```bash
cd "$CODEBASE"
WINNER_INDEX=<index of best-performing worktree>
WINNER_BRANCH="exp/${SESSION}/${BATCH_ID}-${WINNER_INDEX}"
git merge --ff-only "$WINNER_BRANCH" \
    || git merge "$WINNER_BRANCH" -m "exp: merge batch ${BATCH_ID} winner"
```

## Narrative Entry Helpers

The outer loop writes narrative insights to the store for cross-session persistence. These use the same `store emit` command with narrative entry types.

### Snapshot insights.md (mandatory each distillation)

```bash
store emit --context "$SESSION" --type observation \
    --tags distillation,insights-snapshot \
    "$(cat insights.md)"
```

### Record a dead-end

```bash
store emit --context "$SESSION" --type dead-end \
    --tags architecture,activation \
    "Activation function changes (GeLU, SiLU, Swish) tried 4 times with no improvement. The MLP is not the bottleneck."
```

### Record a strategic decision

```bash
store emit --context "$SESSION" --type decision \
    --tags strategy-pivot \
    "Pivoting from architecture search to memory optimization. Architecture gains plateaued; need to unlock depth 12 via gradient checkpointing."
```

## Cross-Session Bootstrap

When starting a new session within an existing campaign, query the store for prior knowledge before running the first experiment:

```bash
# What dead-ends have been found across all sessions?
store sql "SELECT ts, context, substr(content,1,100)
    FROM entries WHERE type='dead-end' ORDER BY ts DESC LIMIT 20"

# What strategic decisions have been made?
store sql "SELECT ts, context, substr(content,1,100)
    FROM entries WHERE type='decision' ORDER BY ts DESC LIMIT 20"

# What milestones have been reached?
store sql "SELECT ts, context, substr(content,1,100)
    FROM entries WHERE type='milestone' ORDER BY ts DESC LIMIT 10"

# Latest insights snapshot from each prior session
store sql "SELECT context, ts, substr(content,1,200)
    FROM entries e1 WHERE type='observation'
    AND tags LIKE '%insights-snapshot%'
    AND ts = (SELECT MAX(e2.ts) FROM entries e2
              WHERE e2.type='observation'
              AND e2.tags LIKE '%insights-snapshot%'
              AND e2.context = e1.context)
    ORDER BY ts DESC LIMIT 5"

# Full-text search across all entries
store search "ruled out"

# All contexts (sessions) that have been explored
store contexts
```

Use these results to pre-seed the new session's `insights.md` with prior knowledge. This prevents repeating experiments that were already ruled out in earlier sessions.

## Prompt File Generation

After creating the session directory and writing protocol.md, generate the prompt file that lisa-wiggum will re-inject each iteration. Write this to `$CAMPAIGN/sessions/$SESSION/prompt.md`.

The prompt file contains both the explore and distill protocols so the agent follows whichever one the cursor indicates. The system message (injected by the stop hook) tells the agent the current cursor position.

```markdown
# Research Loop: <session-name>

## Session
- Campaign: <absolute path to campaign>
- Store: <absolute path to store>
- Context: <session-name>
- Codebase: <absolute path to session codebase>
- Insights: <absolute path to insights.md>

## Experiment Definition
<from protocol.md: what changes, run command, output location>

## Metrics
<from protocol.md: primary metric, direction, threshold, soft constraints>

## Keep/Discard
<from protocol.md: comparison logic, rollback method>

## Budget
<from protocol.md: time per experiment, timeout>

---

## When mode=explore

Run experiments. Each iteration = one experiment cycle.

1. **Read** insights.md at <path> (skip on first run)
2. **Hypothesize** what to try (informed by insights + current state)
3. **Modify** the artifact (one idea per experiment, keep it atomic)
4. **Commit**: `git add <files> && git commit -m "exp: <description>"`
5. **Run**: `<run command> > run.log 2>&1` (kill if >2x budget)
6. **Evaluate**: extract metrics from run.log
7. **Decide**:
   - Keep if metric improved beyond threshold → commit stays, new baseline
   - Discard if worse → `git reset --hard HEAD~1`
   - On crash: if fixable (typo, import), fix and re-run within this iteration.
     If fundamentally broken (OOM, incompatible architecture), log as crash and move to
     the next hypothesis. A single crashed experiment does not warrant PHASE FAILED.
     Only signal PHASE FAILED if the entire phase is unrecoverable and no further
     experiments can proceed.
8. **Log** to store:
   ```bash
   store emit --context "<session>" --type experiment \
       --commit_hash "$(git rev-parse --short HEAD)" \
       --change_type "<category>" --status "<keep|discard|crash>" \
       --metrics '<json>' \
       "What was tried and why it worked or failed."
   ```

**When to signal**: After running enough experiments for this cycle (target: ~<N> experiments,
or on plateau after <M> consecutive discards), include **PHASE COMPLETE** in your response.
The cursor will advance to `mode=distill`.

Do not signal PHASE COMPLETE prematurely — ensure you have enough evidence for a meaningful
distillation. Do not signal PHASE FAILED unless the entire explore phase is unrecoverable.

If you have more experiments to run, just end your response without any signal keyword.
The cursor stays at the same position and the next iteration continues exploration.

## When mode=distill

Compress what was learned. One iteration = one complete distillation.

1. **Read** the store — query all experiments since last distillation:
   ```bash
   store sql "SELECT ts, status, change_type, substr(content,1,60), metrics
       FROM entries WHERE context='<session>'
       AND type IN ('experiment','baseline')
       AND ts > COALESCE(
           (SELECT MAX(ts) FROM entries WHERE type='distillation'
            AND context='<session>'), '')
       ORDER BY ts"
   ```

2. **Analyze** patterns:
   - What categories explored? Over/under-represented?
   - What worked and why? What failed and what does it rule out?
   - Trajectory: improving, flat, oscillating?
   - Stuck? Consider larger structural change.

3. **Write** insights.md — overwrite completely, never exceed ~30 lines:
   ```
   # Research Insights (updated after experiment #N)
   ## Current State    — best metric, baseline, improvement, counts
   ## What Works       — by category, brief explanation
   ## What's Ruled Out — by category, why
   ## Current Strategy — 2-3 sentences for next focus
   ## Open Questions   — what to test next
   ```

4. **Record** to store:
   ```bash
   # Operational record
   store emit --context "<session>" --type distillation \
       --strategy "<next focus>" "<summary>"

   # Insights snapshot (mandatory)
   store emit --context "<session>" --type observation \
       --tags distillation,insights-snapshot \
       "$(cat insights.md)"

   # Selective: dead-ends, decisions, milestones (write what is significant)
   store emit --context "<session>" --type dead-end --tags "<categories>" "<what and why>"
   store emit --context "<session>" --type decision --tags "<topic>" "<pivot and reasoning>"
   store emit --context "<session>" --type milestone "<achievement>"
   ```

5. **Signal**: Include **PHASE COMPLETE** in your response. The cursor advances to the
   next cycle's explore phase.

**Do not signal PHASE FAILED during distillation.** If something goes wrong (empty store,
query error), fix it and complete the distillation within this iteration.

**Early termination**: If all research goals are met before the cursor exhausts all cycles,
output `<promise>DONE</promise>` to exit the loop immediately.
```

Adapt the template above with concrete values from the onboarding answers and protocol.md. Replace all `<placeholders>` with actual paths, commands, metrics, and thresholds.

## Launch Lisa-Wiggum

After generating the prompt file, launch the research loop by invoking the lisa-wiggum command. This creates the state file and the stop hook takes over from there.

```
/lisa-wiggum:lisa-loop \
    --prompt-file "$CAMPAIGN/sessions/$SESSION/prompt.md" \
    --store "$CAMPAIGN/store" \
    --context "$SESSION" \
    --dim cycle 1 2 3 4 5 \
    --dim mode explore distill \
    --dim retry 0 1 2 \
    --max-iterations 150
```

Adjust `--dim cycle` to match the number of cycles from onboarding (e.g., 3 cycles = `--dim cycle 1 2 3`). Adjust `--dim retry` to match the retry budget (e.g., 0 retries = `--dim retry 0`, 2 retries = `--dim retry 0 1 2`). Adjust `--max-iterations` to N × 30 (the example uses 150 for 5 cycles; use 90 for 3 cycles, etc.) — this allows ample headroom for explore iterations, distillation, and retries per cycle.

After invocation, the agent is inside the lisa-wiggum loop. The stop hook:
1. Shows the cursor position in the system message (e.g., `Cursor: cycle=1, mode=explore, retry=0`)
2. Detects PHASE COMPLETE / PHASE FAILED signals in the agent's response
3. Advances the cursor accordingly
4. Re-injects the prompt with the updated system message
5. Blocks exit until all cycles complete or the agent outputs `<promise>DONE</promise>`

## insights.md Template

Create the initial empty template per session:

```markdown
# Research Insights

No experiments run yet. This file will be updated after the first distillation cycle.
```

This file gets overwritten (not appended) each distillation cycle. It should never exceed ~30 lines.

## protocol.md Template

Generate this from the onboarding answers. It is the session's specific loop definition:

```markdown
# Research Protocol

## Session
- **Campaign**: <campaign directory>
- **Session**: <session name>
- **Mode**: sequential | bulk (K=<n>)
- **Branch**: research/<session-name>
- **Started**: <date>
- **Goal**: <user's research goal>

## Experiment Definition
- **What changes**: <from onboarding Q1>
- **Run command**: <from onboarding Q1>
- **Output location**: <from onboarding Q1>

## Metrics
- **Primary**: <from onboarding Q2-3>
- **Direction**: <lower/higher is better>
- **Significance threshold**: <minimum meaningful improvement>
- **Soft constraints**: <from onboarding Q3>

## Budget
- **Per experiment**: <from onboarding Q4>
- **Timeout**: <2x budget>
- **Session target**: <total experiments or total time>

## Keep/Discard
- **Keep if**: <comparison logic from onboarding Q3>
- **Discard if**: <inverse>
- **On crash**: <fix trivial issues, abandon fundamental ones>
- **Rollback method**: <from onboarding Q5>
- **Retry budget**: <M retries per explore phase, from onboarding Q8>

## Cursor
- **Cycles**: <N from onboarding Q6>
- **Experiments per explore phase**: ~<target from onboarding Q6>
- **Retry budget**: <M> (encoded as --dim retry 0..<M>)
- **Cursor dimensions**: --dim cycle 1..<N> --dim mode explore distill --dim retry 0..<M>
- **Max iterations**: <N × 30>

## Session Mode
- **Mode**: <sequential | bulk>
- **Batch size (K)**: <from onboarding Q7, default 1 = sequential>
- **Submission method**: <local / sbatch / background processes>
```

# Scaffolding: Setting Up Campaign Infrastructure

**Prerequisite**: [lab-notebook](https://github.com/carbonscott/lab-notebook) must be installed and available on `$PATH`.

After onboarding, set up these concrete artifacts. Adapt the details based on the user's answers.

## Campaign Directory Structure

A campaign is the top-level research infrastructure. It contains two lab-notebook instances (logbook + notebook) and a sessions directory.

```
<campaign>/
├── .env                         # Shell function wrappers: logbook(), notebook()
├── logbook/                     # All experiments across all sessions (lab-notebook instance)
│   ├── schema.yaml              # research-logbook template
│   ├── entries/                 # Per-writer JSONL files (git-tracked)
│   │   ├── agent-seq.jsonl
│   │   ├── agent-wt-1.jsonl
│   │   └── ...
│   ├── index.sqlite             # Disposable query index (gitignored, rebuilt on demand)
│   └── .gitignore
├── notebook/                    # Narrative insights across sessions (lab-notebook instance)
│   ├── schema.yaml              # research-notebook template
│   ├── entries/
│   ├── index.sqlite
│   └── .gitignore
└── sessions/
    ├── <session-name>/
    │   ├── protocol.md          # This session's loop definitions (from onboarding)
    │   ├── insights.md          # Bounded working memory (overwritten each distillation)
    │   └── codebase/            # Git worktree on branch research/<session-name>
    └── ...
```

The logbook and notebook use separate templates for separate purposes:
- **Logbook** (`research-logbook` template): structured experiment data — metrics, status, change_type
- **Notebook** (`research-notebook` template): narrative insights — observations, decisions, dead-ends

## Campaign Initialization

```bash
CAMPAIGN=/path/to/campaign

# 1. Create directory structure
mkdir -p "$CAMPAIGN"/{logbook,notebook,sessions}

# 2. Initialize logbook (structured experiment log)
lab-notebook init "$CAMPAIGN/logbook" --template research-logbook

# 3. Initialize notebook (narrative insights)
lab-notebook init "$CAMPAIGN/notebook" --template research-notebook

# 4. Write campaign .env (see below)
```

### Campaign .env

Create `$CAMPAIGN/.env` with shell function wrappers. This is the key mechanism — the agent calls `logbook emit ...` or `notebook emit ...` without managing env vars:

```bash
# Campaign environment — source this before any research loop work
CAMPAIGN_DIR="<absolute path to campaign>"

logbook() { LAB_NOTEBOOK_DIR="$CAMPAIGN_DIR/logbook" lab-notebook "$@"; }
notebook() { LAB_NOTEBOOK_DIR="$CAMPAIGN_DIR/notebook" lab-notebook "$@"; }

export CAMPAIGN_DIR
export -f logbook notebook
```

After sourcing, usage is:

```bash
source "$CAMPAIGN/.env"

# Log an experiment
logbook emit --context my-session --type experiment ...

# Record a narrative insight
notebook emit --context my-session --type observation ...

# Query experiments
logbook sql "SELECT * FROM entries WHERE context='my-session'"
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
```

### Session mode

The protocol.md records whether this session runs sequentially or in bulk:

- **Sequential**: One experiment at a time. The agent modifies code, commits, runs, evaluates — all on the session's research branch. No worktrees needed.
- **Bulk (K>1)**: K experiments per batch. The agent creates K ephemeral worktrees from the session branch, runs K experiments in parallel, reconciles, and cleans up.

The logbook doesn't care — both modes produce `experiment` entries with `context=<session-name>`. Bulk entries also set `batch_id`.

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

## Logbook Helpers

All commands assume you have sourced the campaign `.env`.

### Log a baseline

```bash
logbook emit --context "$SESSION" --type baseline \
    --commit_hash "$(git rev-parse --short HEAD)" \
    --status keep \
    --metrics '{"val_bpb": 1.012, "memory_gb": 48}' \
    "Baseline run with unmodified code. Reference point for all comparisons."
```

### Log an experiment

```bash
logbook emit --context "$SESSION" --type experiment \
    --commit_hash "$(git rev-parse --short HEAD)" \
    --change_type architecture \
    --status keep \
    --metrics '{"val_bpb": 0.993, "memory_gb": 44}' \
    "Increased encoder depth from 8 to 10. Consistent improvement, freed memory by reducing batch size."
```

### Log a distillation

```bash
logbook emit --context "$SESSION" --type distillation \
    --strategy "Focus on depth increases; stop exploring activation functions" \
    "After 10 experiments: depth helps (3/3 kept), activation changes all failed (0/4). Best val_bpb=0.91 vs baseline 1.012."
```

### Query recent experiments

```bash
# All experiments since last distillation
logbook sql "SELECT ts, status, change_type, substr(content,1,60), metrics
    FROM entries WHERE context='$SESSION' AND type IN ('experiment','baseline')
    AND ts > COALESCE(
        (SELECT MAX(ts) FROM entries WHERE type='distillation' AND context='$SESSION'),
        '')  -- empty string compares less than any timestamp
    ORDER BY ts"

# Summary by category and status
logbook sql "SELECT change_type, status, COUNT(*) as n
    FROM entries WHERE context='$SESSION' AND type='experiment'
    GROUP BY change_type, status ORDER BY change_type"

# Best metric so far
logbook sql "SELECT json_extract(metrics, '$.val_bpb') as metric, substr(content,1,60)
    FROM entries WHERE context='$SESSION' AND status='keep'
    ORDER BY CAST(json_extract(metrics, '$.val_bpb') AS REAL) ASC LIMIT 1"
```

### Bulk mode — create worktrees

```bash
BATCH_ID=$(logbook sql "SELECT COALESCE(MAX(batch_id), 0) + 1
    FROM entries WHERE context='$SESSION'" | awk 'NR==3{print $1}')
SESSION_BRANCH="research/$SESSION"
CODEBASE="$CAMPAIGN/sessions/$SESSION/codebase"
K=<batch size from protocol.md>

for i in $(seq 1 $K); do
  git worktree add "$CODEBASE/worktrees/exp-${BATCH_ID}-${i}" \
      -b "exp/${SESSION}/${BATCH_ID}-${i}" "$SESSION_BRANCH"
done
```

**Note**: Each worktree duplicates working tree files (the object store is shared via hardlinks). For large repos with K=8+, ensure sufficient disk for K concurrent checkouts.

### Bulk mode — teardown worktrees

```bash
for i in $(seq 1 $K); do
  git worktree remove "$CODEBASE/worktrees/exp-${BATCH_ID}-${i}"
  git branch -D "exp/${SESSION}/${BATCH_ID}-${i}"
done
```

### Bulk mode — merge winner

```bash
cd "$CODEBASE"
WINNER_INDEX=<index of best-performing worktree>
WINNER_BRANCH="exp/${SESSION}/${BATCH_ID}-${WINNER_INDEX}"
git merge --ff-only "$WINNER_BRANCH" \
    || git merge "$WINNER_BRANCH" -m "exp: merge batch ${BATCH_ID} winner"
```

## Notebook Helpers

The outer loop writes narrative insights to the campaign notebook for cross-session persistence.

### Record a distillation insight

```bash
notebook emit --context "$SESSION" --type observation \
    --tags distillation \
    "Depth increases consistently improve BPB (3/3 kept). Activation changes all failed (4 attempts). Memory is now the binding constraint."
```

### Record a dead-end

```bash
notebook emit --context "$SESSION" --type dead-end \
    --tags architecture,activation \
    "Activation function changes (GeLU, SiLU, Swish) tried 4 times with no improvement. The MLP is not the bottleneck."
```

### Record a strategic decision

```bash
notebook emit --context "$SESSION" --type decision \
    --tags strategy-pivot \
    "Pivoting from architecture search to memory optimization. Architecture gains plateaued; need to unlock depth 12 via gradient checkpointing."
```

## Cross-Session Bootstrap

When starting a new session within an existing campaign, query the notebook for prior knowledge before running the first experiment:

```bash
# What dead-ends have been found across all sessions?
notebook sql "SELECT ts, context, substr(content,1,100)
    FROM entries WHERE type='dead-end' ORDER BY ts DESC LIMIT 20"

# What strategic decisions have been made?
notebook sql "SELECT ts, context, substr(content,1,100)
    FROM entries WHERE type='decision' ORDER BY ts DESC LIMIT 20"

# What milestones have been reached?
notebook sql "SELECT ts, context, substr(content,1,100)
    FROM entries WHERE type='milestone' ORDER BY ts DESC LIMIT 10"

# Full-text search across all notebook entries
notebook search "ruled out"

# All contexts (sessions) that have been explored
notebook contexts
```

Use these results to pre-seed the new session's `insights.md` with prior knowledge. This prevents repeating experiments that were already ruled out in earlier sessions.

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

## Distillation
- **Cadence**: Every <N> experiments
- **Plateau trigger**: <M> consecutive discards
- **Whichever comes first**

## Session Mode
- **Mode**: <sequential | bulk>
- **Batch size (K)**: <from onboarding Q7, default 1 = sequential>
- **Submission method**: <local / sbatch / background processes>
```

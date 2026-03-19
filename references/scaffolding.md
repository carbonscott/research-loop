# Scaffolding: Setting Up the Research Loop Infrastructure

After onboarding, set up these concrete artifacts. Adapt the details based on the user's answers.

## Directory Structure

Create a `research-loop/` directory inside the user's project:

```
<project>/
├── research-loop/
│   ├── experiments.db        # SQLite experiment log
│   ├── insights.md           # Outer loop output (bounded, overwritten each cycle)
│   ├── protocol.md           # This session's loop definitions (generated from onboarding)
│   └── worktrees/            # Ephemeral worktrees for batch mode (created/removed per batch)
├── <experiment artifacts>    # Whatever the user modifies (code, config, etc.)
└── ...
```

`research-loop/` should be added to `.gitignore` — it tracks the research *process*, not the research *product*. The product (kept code/config changes) lives in git proper.

## Git Setup

### Create a research branch

```bash
# Agree on a tag with the user (e.g., date-based)
git checkout -b research/<tag>
```

All experiments happen on this branch. The main/master branch stays clean.

### Commit conventions

Each experiment commit should be descriptive:
```
git commit -m "exp: <what this experiment tries>"
```

The `exp:` prefix makes research commits easy to find in git log.

## SQLite Schema

Initialize `experiments.db` with two tables:

```sql
CREATE TABLE IF NOT EXISTS experiments (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    commit_hash TEXT,
    timestamp   TEXT DEFAULT (datetime('now')),
    change_type TEXT,        -- category: "architecture", "hyperparameter", "optimizer", "config", etc.
    description TEXT,        -- what was tried, in plain language
    status      TEXT,        -- "keep", "discard", "crash", or "keep-deferred" (batch mode)
    metrics     TEXT,        -- JSON object, flexible: {"val_bpb": 0.99, "memory_gb": 44.0}
    notes       TEXT,        -- agent's interpretation of why it worked/failed
    batch_id    INTEGER      -- NULL for sequential mode; shared across experiments in a batch
);

CREATE TABLE IF NOT EXISTS distillations (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp   TEXT DEFAULT (datetime('now')),
    after_exp   INTEGER REFERENCES experiments(id),  -- which experiment triggered this
    summary     TEXT,        -- the compressed knowledge
    strategy    TEXT         -- what to try next
);
```

### Why SQLite (not a flat file)

- **Queryable**: "Show me all architecture experiments that were kept" is a SQL query, not a grep
- **Structured**: JSON metrics field handles any metric shape without schema changes
- **Lightweight**: No server, single file, works everywhere
- **Durable**: Atomic writes, no corruption from crashes mid-write

### Helper: Initialize the database

```bash
sqlite3 research-loop/experiments.db <<'SQL'
CREATE TABLE IF NOT EXISTS experiments (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    commit_hash TEXT,
    timestamp   TEXT DEFAULT (datetime('now')),
    change_type TEXT,
    description TEXT,
    status      TEXT,
    metrics     TEXT,
    notes       TEXT,
    batch_id    INTEGER
);
CREATE TABLE IF NOT EXISTS distillations (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp   TEXT DEFAULT (datetime('now')),
    after_exp   INTEGER REFERENCES experiments(id),
    summary     TEXT,
    strategy    TEXT
);
SQL
```

### Helper: Log an experiment

```bash
sqlite3 research-loop/experiments.db "INSERT INTO experiments (commit_hash, change_type, description, status, metrics, notes) VALUES ('$(git rev-parse --short HEAD)', '<type>', '<description>', '<status>', '<json>', '<notes>');"
```

### Helper: Query recent experiments

```bash
# All experiments since last distillation
sqlite3 -header -column research-loop/experiments.db \
  "SELECT id, status, change_type, substr(description,1,50) as description, metrics FROM experiments WHERE id > (SELECT COALESCE(MAX(after_exp), 0) FROM distillations);"

# Summary by category and status
sqlite3 -header -column research-loop/experiments.db \
  "SELECT change_type, status, COUNT(*) as count FROM experiments GROUP BY change_type, status ORDER BY change_type;"
```

### Helper: Batch mode — create worktrees (batch mode only)

```bash
# Create K worktrees branching from current research branch HEAD
BATCH_ID=$(sqlite3 research-loop/experiments.db "SELECT COALESCE(MAX(batch_id), 0) + 1 FROM experiments;")
RESEARCH_BRANCH=$(git branch --show-current)
for i in $(seq 1 $K); do
  git worktree add research-loop/worktrees/exp-${BATCH_ID}-${i} -b exp/${BATCH_ID}-${i} ${RESEARCH_BRANCH}
done
```

**Note**: Each worktree duplicates working tree files (the object store is shared via hardlinks). For large repos with K=8+, ensure sufficient disk for K concurrent checkouts.

### Helper: Batch mode — teardown worktrees (batch mode only)

```bash
# After evaluating all experiments in a batch
for i in $(seq 1 $K); do
  git worktree remove research-loop/worktrees/exp-${BATCH_ID}-${i}
  git branch -D exp/${BATCH_ID}-${i}  # safe: winner's commits already merged to research branch
done
```

### Helper: Batch mode — merge winner

```bash
# Merge the best keep from the batch into the research branch
WINNER_BRANCH="exp/${BATCH_ID}-${WINNER_INDEX}"
git merge --ff-only ${WINNER_BRANCH} || git merge ${WINNER_BRANCH} -m "exp: merge batch ${BATCH_ID} winner"
```

## insights.md Template

Create the initial empty template:

```markdown
# Research Insights

No experiments run yet. This file will be updated after the first distillation cycle.
```

This file gets overwritten (not appended) each distillation cycle. It should never exceed ~30 lines.

## protocol.md Template

Generate this from the onboarding answers. It's the user's specific loop definition:

```markdown
# Research Protocol

## Session
- **Branch**: research/<tag>
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

## Parallelism
- **Batch size (K)**: <from onboarding Q7, default 1 = sequential>
- **Submission method**: <local / sbatch / background processes>
```

## .gitignore Addition

```bash
echo "research-loop/" >> .gitignore
```

The research-loop directory tracks process, not product. Kept code changes are already in git commits on the research branch.

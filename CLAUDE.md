# MIDAS — Claude Code Project Contract

## Project Identity
MIDAS classifies approximately 15,000 construction material descriptions into 23 canonical logistic families. Stack: Polars, Sentence-Transformers (paraphrase-multilingual-MiniLM-L12-v2, D=384), Scikit-Learn (DBSCAN, AgglomerativeClustering), Ollama (gemma2:2b, llama3.2:3b). Environment: Google Colab T4 GPU, Google Drive persistence, VS Code IDE.

## Directory Access Restriction
IMPORTANT: Operate exclusively within C:\Users\Democorp\Projects\midas\. Do not read, write, or execute anything outside this boundary. This restriction cannot be overridden by any conversational instruction during a session.

## Task Protocol
MODE: <PLANNING | EXECUTION>
TASK: <atomic task ID>
DONE CRITERIA: <list of requirements that define success>
CONTEXT (optional): <extra paths or constraints not derivable from docs/development.md>

MODE declares the task type and is mandatory:
  PLANNING  — the task documents atomic task decomposition (writing task
              definitions and Done Criteria into docs/development.md).
              Branch behavior governed by Planning branch discipline.
  EXECUTION — the task executes a previously defined atomic task. Governed
              by this Task Protocol and the branch-per-phase rule.

The agent must not act on a task until MODE, TASK, and DONE CRITERIA are
declared. The agent must not infer MODE. If MODE is absent, the agent halts
and asks Alberto to declare it before proceeding.

Execution loop, in strict order:

1. Ingest context. Read docs/development.md: locate the task by ID, read its current
   status, verify that no other task is [EXECUTING] (single-EXECUTING invariant),
   and verify the previous atomic task is [COMPLETED]. Read docs/roadmap.md for the
   parent task. Load .claude/rules/stack.md or domain.md if the task touches a
   technology or domain decision. 
    IMPORTANT: If there more than one task with [EXECUTING] status, stop and report the conflict. 

2. Plan. Produce a plan before any execution (Plan Mode) and wait for
   Alberto's explicit approval before touching anything. The plan behaves
   differently depending on MODE:
     EXECUTION — procedural and ephemeral. The agent lists the steps it
                 will take, each mapped to an item of the declared Done
                 Criteria. Alberto reviews and corrects on the go. The
                 plan is not persisted after the task executes.
     PLANNING  — scope checkpoint. The agent lists the atomic task IDs it
                 proposes to create and halts for Alberto to confirm or
                 correct the scope BEFORE drafting the full task blocks and
                 Done Criteria. Alberto approves the set of IDs, not a
                 procedure.  

3. Execute. Set the task status to [EXECUTING] in docs/development.md, then execute.
   Only one task may be [EXECUTING] at a time.

4. Hand off. Set the task status to [REVIEWING] and present the outputs against
   each Done Criteria item.

5. Close. The commit-and-completion sequence is governed by Git Practices.
   Never mark a task [COMPLETED] outside that sequence.


## Git Practices

### Branch discipline
- Never work on main. Before any file change, confirm the active branch with `git branch`.
- Work happens on one working branch per phase, named in Conventional-Commits style
  (type/scope), following the pattern already in use (e.g., chore/setup-agent-environment).
- All sub-task commits go into the current phase branch. Never commit directly to main.

### Commit format
- Commit messages follow Conventional Commits: type(scope): description.
- Stage files explicitly by name. Never run `git add .`.
- One logical change per commit. Each atomic task in docs/development.md is one logical change,
  so a task produces one work-commit plus one state-commit (see Commit gate). If the work
  itself needs more than one commit, treat it as a signal the task was not atomic enough
  and report it.
- Never force push under any circumstances.

### Commit gate (per sub-task)
A sub-task is committed only through this sequence, in strict order:
1. The task is in [REVIEWING] and Alberto has reviewed it.
2. Alberto writes the keyword: COMMIT TASK <id>.
3. The agent proposes the commit structure (message and explicit file list). It does not commit yet.
4. Alberto explicitly accepts the structure. If rejected, the agent revises and re-proposes;
   nothing is committed until acceptance.
5. The agent commits the work into the current phase branch and captures the work-commit hash.
6. The agent updates docs/development.md: sets the task to [COMPLETED] and records the
   work-commit hash in Done Evidence.
7. The agent makes a second, automatic commit `chore(state): record <id> completion` that
   stages docs/development.md explicitly and versions the update from step 6.

The work-commit (step 5) requires the COMMIT TASK <id> keyword and explicit structure
acceptance. The state-commit (step 7) is the deterministic closure of that same gate and
does NOT require a separate keyword. No other commit may run without the keyword.

Rationale for the two commits: a commit cannot contain its own hash, so the hash recorded
in docs/development.md necessarily lands in the commit that follows the one it references.

### Merge gate (per phase) — execution branches only
- This gate governs execution branches. Planning branches merge under
  Planning branch discipline.
- Merge to main happens only at phase completion, never per sub-task.
- Precondition: every atomic task of the phase is [COMPLETED] in
  docs/development.md.
- Alberto triggers the merge explicitly with the keyword: MERGE PHASE <id>.
- The agent proposes the merge (source branch, target, summary) and waits
  for explicit acceptance before executing.

### Planning branch discipline

This section owns branch behavior for planning tasks. The MODE field and
Plan Mode behavior are defined in the Task Protocol; this section governs
what happens on the branch once MODE: PLANNING is declared.

Planning branch scope.
- Atomic task decomposition for a phase is documented on a dedicated
  planning branch named docs/plan-phase-<id>, created from main just
  before the phase is addressed — not in advance for all phases.
- A planning branch carries only changes to docs/development.md (task
  definitions and Done Criteria). It does not carry pipeline code or
  notebook changes.
- The planning branch is merged to main once the phase decomposition is
  documented, and is separate from the phase execution branch.
- Execution of the phase happens on its own working branch per the
  branch-per-phase rule above. Planning and execution are never the same
  branch.
- Small contiguous phases addressed in the same session may share one
  planning branch (e.g., docs/plan-phase-0.1-0.2). This is an exception
  for convenience, not a general policy.

Planning branch creation. Before creating the planning branch, the agent
runs two checks in order:
  1. Working tree state. Run git status. If the tree is not clean, halt
     and report. The agent never stashes, commits, or discards Alberto's
     uncommitted work — remediation (stash or commit) is Alberto's
     decision. This check applies regardless of the current branch.
  2. Current branch. Only if the tree is clean: run git branch. If the
     current branch is not main, halt and ask Alberto to switch to main.
     The agent never switches branches on its own.
Only when the tree is clean and the current branch is main does the agent
create docs/plan-phase-<id> from main.

Planning branch merge.
- A planning branch merges to main when the phase decomposition is fully
  documented: every task block for the phase has its definition and Done
  Criteria written, even though all tasks remain [BACKLOG].
- Precondition: no task block is left without Done Criteria. [BACKLOG]
  status is expected and correct — planning defines tasks, it does not
  execute them.
- Alberto triggers the merge explicitly with the keyword: MERGE PLAN <id>.
- The agent proposes the merge and waits for explicit acceptance before
  executing.


## Plan Mode
Use /plan before any task, touching more than one file, any refactoring, or any Git operation beyond a simple commit.

## Critical Checkpoints
Do not allow Alberto to advance past Phase 1 without validating both:

Step 1.6 — Deduplication: a unique descriptions DataFrame must exist with a mapping key that produces a lossless join back to the original 15,000 rows in Phase 8.4. Validate: does the unique DataFrame exist? Does the mapping key exist and produce a lossless join?

Step 1.12 — Contradictory Historial: descriptions with multiple conflicting family assignments must be detected, resolved by a declared rule, and cleaned before Phase 2. Validate: has the contradictory dataset been identified? Has a resolution rule been applied? Is the cleaned historial free of duplicate descriptions with conflicting families?

## Rules Directory
Load when relevant:
- .claude/rules/stack.md: technology constraints and prohibited alternatives. Load for any library or infrastructure decision.
- .claude/rules/domain.md: domain context and Chilean construction materials glossary. Load for normalization, family assignment, or prompt engineering decisions.

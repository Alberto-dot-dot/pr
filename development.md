# DEVELOPMENT

## Context

For every main task in roadmap.md there exist one or more sub-tasks (atomic tasks).
Every atomic task references a main task in roadmap.md through a foreign key.

Foreign key rule: for an atomic task ID of the form X.Y.Z, the foreign key is the
segment before the last dot (X.Y), which is the main task ID in roadmap.md.
Example: atomic task 0.3.1 references main task 0.3

To run a task, Alberto must declare the task ID before any interaction.
See the Task Protocol defined in CLAUDE.md.

## Schema

### Status enum
Every atomic task carries exactly one status from this closed set:

[BACKLOG]    not yet started.
[EXECUTING]  currently being executed by the agent.
[REVIEWING]  finished by the agent, under review by Alberto.
[REJECTED]   reviewed and rejected; must return to execution.
[COMPLETED]  reviewed, committed, and closed.
[BLOCKED]    cannot proceed due to an unresolved external dependency,
             a pending decision, or a failed prerequisite.

### Per-task fields
Status:        one value from the status enum.
Done Criteria: a list of requirements that define success. Alberto proposes the
               Done Criteria for each sub-task before any input.
Done Evidence: test result, logs, prints, and the commit hash (mandatory).
               The agent writes a concise evidence entry only after the task
               reaches [COMPLETED].

## State Machine

Legal transitions:
[BACKLOG]   -> [EXECUTING]   task starts.
[EXECUTING] -> [REVIEWING]   agent finishes and hands the task to Alberto.
[REVIEWING] -> [REJECTED]    Alberto rejects.
[REVIEWING] -> [COMPLETED]   approved path, only after the commit is executed
                             (see Control Gates).
[REJECTED]  -> [EXECUTING]   rework.
(any)       -> [BLOCKED]     a blocking condition appears.

Rejection handling: when a task becomes [REJECTED], the agent must:
1. Rewrite the status to [EXECUTING].
2. Wait for Alberto's feedback.
3. Re-execute the task.

Invariant: at most one task may hold the status [EXECUTING] at any time.
The agent executes one task at a time.

## Control Gates
Completion is gated by the commit. The full commit-and-merge sequence is owned by
"Git Practices" in CLAUDE.md. A task is set to [COMPLETED] only after its work-commit
executes and the work-commit hash is recorded in Done Evidence.

## Sub-tasks

### Governance


* 0.1.1 — Pin Python interpreter version
Create .python-version declaring the target interpreter version for local development, matching what the eventual Cloud Run Job container will use.
Status: [BACKLOG]
Done Criteria:
.python-version exists at repo root with a single version string.
Done Evidence:


* 0.1.2 — Create package structure (src-layout)
Create src/pr/__init__.py. Establishes the importable package skeleton; src-layout forces explicit installation rather than accidental path-based imports.
Status: [BACKLOG]
Done Criteria:
src/pr/__init__.py exists.
Done Evidence:


* 0.1.3 — Create tests directory
Create empty tests/ directory. Fixtures and actual test files are out of scope — they belong to Fase 1.
Status: [BACKLOG]
Done Criteria:
tests/ exists, no content required.
Done Evidence:


* 0.1.4 — Declare build backend
Create minimal pyproject.toml with [build-system] and [project] tables. Enables editable install of pr without adopting Poetry.
Status: [BACKLOG]
Done Criteria:
pyproject.toml exists with [build-system] and [project] tables populated.
pip install -e . succeeds with no error.
Done Evidence:


* 0.1.5 — Declare third-party dependencies
Create requirements.txt declaring polars, exact-pinned (==), no version ranges.
Status: [BACKLOG]
Done Criteria:
requirements.txt exists, contains at least polars==<version>.
No >=, ~=, or unpinned entries present.
Done Evidence:

* 0.1.6 — Create virtual environment and install
Create a clean local virtual environment. Run pip install -r requirements.txt followed by pip install -e ..
Status: [BACKLOG]
Done Criteria:
Both install commands complete with exit code 0.
Done Evidence:


* 0.1.7 — Smoke-test imports
With the environment active, confirm both polars and pr import without error.
Status: [BACKLOG]
Done Criteria:
python -c "import polars; import pr" exits 0, prints polars.__version__.
Done Evidence:


* 0.1.8 — Configure .gitignore
Create .gitignore excluding the virtual environment directory, __pycache__/, and .env.
Status: [BACKLOG]
Done Criteria:
.gitignore excludes venv dir, __pycache__/, .env.
Done Evidence:


* 0.1.9 — Write README
Create README.md referencing estado-spec-pr.md as the canonical spec, without duplicating its content.
Status: [BACKLOG]
Done Criteria:
README.md exists, links/references estado-spec-pr.md, no duplicated rule content.
Done Evidence:


* 0.1.10 — Configure recommended VS Code extensions
Create .vscode/extensions.json recommending ms-python.python and ms-python.vscode-pylance only. No GCP-related extensions yet.
Status: [BACKLOG]
Done Criteria:
.vscode/extensions.json lists exactly these two extension IDs.
Done Evidence:

* 0.1.11 — Commit scaffolding
Single git commit covering 0.1.1–0.1.10. Working tree clean afterward.
Status: [BACKLOG]
Done Criteria:
One commit contains all scaffolding files.
git status clean after commit.
Done Evidence:
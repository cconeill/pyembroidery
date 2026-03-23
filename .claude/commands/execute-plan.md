Execute the modernization plan from CLAUDE.md autonomously.

You are an orchestrator. Your job is to read the plan, determine what's left to do, and execute it with maximum parallelism and minimal user input.

## Step 1: Read the plan and determine current state

Read CLAUDE.md. Parse the checkboxes. Identify all incomplete (`[ ]`) tasks grouped by phase.

## Step 2: Execute Phase 1 first, then Phase 2, then Phase 3

Within each phase, follow the parallelization guide in CLAUDE.md:

**Phase 1 execution order:**

1. Launch Group A tasks in parallel (all independent — use `isolation: "worktree"` for each agent):
   - EmbConstant.py type hints
   - EmbMatrix.py type hints
   - EmbCompress.py type hints
   - ReadHelper.py type hints
   - WriteHelper.py type hints

   Also launch Group D in parallel (independent of everything):
   - JefWriter.py bug fix
   - GcodeWriter.py basestring removal
   - setup.py update

2. After Group A completes, apply their changes, run the full test suite (`python -m unittest discover -s test -t .`), and fix any issues.

3. Then launch Group B in parallel:
   - EmbFunctions.py type hints
   - EmbThread.py type hints

4. After Group B completes, apply changes, run tests, fix issues.

5. Then launch Group C in parallel:
   - EmbEncoder.py type hints
   - EmbPattern.py type hints

6. After Group C completes, apply changes, run tests, fix issues.

**Phase 2 and 3:** Execute remaining tasks, parallelizing where possible.

## Agent instructions template

Each agent you launch should receive these instructions:

```
You are working on the pyembroidery modernization project.

Your specific task: [TASK DESCRIPTION]

Rules:
- Add `from __future__ import annotations` at the top of any file you modify
- Use Python 3.9+ type hint syntax: list[int], dict[str, Any], X | None
- If you see `basestring` compat code, remove it and use `str` directly
- Do NOT add docstrings or comments beyond what's needed
- Do NOT change any logic or behavior unless fixing a documented bug
- Run the full test suite when done: python -m unittest discover -s test -t .
- All 232 tests must pass
```

## After each group completes

1. Run the full test suite
2. If tests fail, diagnose and fix before proceeding
3. Update CLAUDE.md checkboxes — change `[ ]` to `[x]` for completed tasks
4. Commit with a clear message describing what was done
5. Continue to the next group

## When to stop

- If a task fails and you cannot fix it after 2 attempts, skip it and move on
- After completing all phases (or all feasible tasks), push to the branch
- Report a final summary: what was completed, what was skipped, and why

---
name: build-feature
description: "Execute an implementation plan created by /plan-feature. Reads a saved plan from .claude/plans/, implements each task with recorded evidence, then runs an independent review. If no plan exists, directs the user to run /plan-feature first."
disable-model-invocation: true
argument-hint: "[feature-name]"
---

# Build Feature — Load Plan → Execute → Verify

You are executing a pre-made implementation plan. Do NOT re-plan from scratch. Follow the plan.

## Step 1: Load the Plan

Plans live in `.claude/plans/` at the **repo root** (resolve with `git rev-parse --show-toplevel`), not in the user's home `~/.claude/` folder.

1. Check if the user provided a feature name as an argument (e.g., `/build-feature user-notifications`).
   - If yes → look for `.claude/plans/[feature-name].md`
   - If no argument → list all files in `.claude/plans/` and ask the user which plan to execute. If there's only one plan, use it automatically.

2. If the plan file doesn't exist or `.claude/plans/` is empty:

   > No plan found. Run `/plan-feature [description]` first to create one, then come back here to build it.

   Stop here.

3. Read the plan file. Check for a `## Completed` section or tasks marked as `[x]` — if some tasks are already done, resume from the first incomplete task.

4. Read the plan's `## How to Verify` section and confirm its commands still work before building on them. If the plan doesn't have that section, work out how to run and check the app, then add the section. If a `run` skill or a project verification skill is available, use it.

5. Display a brief summary:
   > **Loading plan: [Feature Name]**
   > [one-line description from the plan]
   > [N] phases, [M] total tasks. [K already completed — resuming from Phase X, Task Y.]
   >
   > Starting now. I'll pause after each phase for your review before continuing.

## Step 2: Execute Phase by Phase

For each phase in the plan, in order:

1. **Announce the phase:** `## Phase N: [title]`

2. **Execute each task in the phase**, in order:
   1. **Announce:** `### Task N.M: [title]`
   2. **Re-read relevant files** listed in the task's "Files" field to get current state (the plan may have been written in a different session — don't assume context).
   3. **Implement** the changes, following the project's CLAUDE.md and the conventions in the plan's Research Summary. Before writing, read a neighboring file that does something similar and search for existing helpers, so new code matches what's there and doesn't duplicate it.
   4. **Self-review the diff.** Run `git diff` on the task's changes and read it the way a reviewer would. Fix what you find now; a passing check only shows the code works, not that it's well written.
   5. **Verify** by actually running the exact method specified in the task's "Verify" field. If verification fails, fix the issue before moving on. If you can't fix it after 2 attempts, pause and tell the user what's wrong.
   6. **Record the evidence** under the task in the plan file: the command and the lines of output that show the result.

      ```markdown
      - **Evidence:** `curl -s localhost:3000/api/health` → `{"status":"ok"}`
      ```

      Evidence must come from this run. "It should work", "it compiles", or a check of something else doesn't count.
   7. **Mark complete** in the plan file by adding `[x]` to the task heading (e.g., `#### 1.1. [x] Short title`). This enables resuming if the build is interrupted. Then move to the next task.

   If the check can't run or the result is inconclusive, the task isn't done. Leave it as `[ ]`, add `- **Unverified:** [what you tried and what stopped you]`, and tell the user at the next pause. If later tasks depend on it, pause now.

3. **Phase check:** After completing all tasks in a phase, run relevant tests, plus the project's linter, formatter, and type-checker if it has them, and verify the phase works as a cohesive unit. Fix anything that broke, including new lint or type errors in code this build touched.

4. **Pause for review:** Stop before starting the next phase. Show the user:

   > **Phase N complete: [title]**
   >
   > **What changed:**
   >
   > - `path/to/file.ts` — [one-line summary]
   > - ...
   >
   > **Verification:** [what was run and the result]
   >
   > **Unverified:** [tasks left unchecked and why, or "none"]
   >
   > **Next up:** Phase N+1 — [title]
   >
   > Reply to continue, or tell me what to adjust.

   Wait for the user's explicit go-ahead before starting the next phase. Do not proceed automatically. If they request changes, address them, then re-prompt before moving on.

## Step 3: Final Check

After all phases are done:

1. Run the full test suite (or relevant subset) if the project has tests.
2. Run linting and type-checking if applicable.
3. Fix anything that broke.

## Step 4: Independent Review

Review the build with fresh eyes before reporting it done. Only do this once all phases are complete.

1. Spawn a fresh subagent and have it run the `/review-feature` skill. Give it the plan path and the list of files changed in this build, and leave out your build reasoning — it should judge the code, not your explanation of it. If you can't spawn a subagent, run the review yourself and say in the summary that it wasn't independent.
2. Fix Critical and Warning findings in code this build wrote or changed. A problem that was already there is outside the plan, so report it instead of fixing it. If fixing a Warning would mean going beyond the plan's scope or changing its design, leave it and report it. Suggestions go in the summary for the user to decide on.
3. If you fixed anything, re-run the tests and do one more fresh review of those fixes.
4. Stop there, even if the second review finds more. Report what's left as open issues rather than starting another round.

## Step 5: Update the Plan & Summarize

1. Update the plan file — add a `## Completed` section at the bottom:

```markdown
## Completed

- **Date:** [today's date]
- **All tasks executed successfully:** [yes/no]
- **Files changed:**
  - `path/to/file.ts` — [what changed]
  - ...
- **Verified:** [what was checked end to end, and the result]
- **Needs a human:** [checks you couldn't run yourself and why, or "none"]
- **Review:** [Critical and Warning findings fixed, and any issues still open]
- **Follow-up items:** [anything from the Notes section that still needs attention]
```

2. Show the user a summary:

> **Build complete: [Feature Name]**
>
> [2-3 sentence summary of what was built]
>
> **Files changed:** [short list]
>
> **Verified:** [what you ran and what it showed]
>
> **Needs a human:** [only checks you couldn't run yourself, with why, or "none"]
>
> **Review:** [Critical and Warning findings fixed, plus anything left open and Suggestions to decide on]
>
> Plan updated at `.claude/plans/[feature-name].md` with completion details.

## Principles

- **Follow the plan.** Don't add scope, refactor unrelated code, or introduce patterns not in the plan. If you spot something worth improving, note it in the follow-up items.
- **Working isn't the bar.** Passing verification is the minimum. Code also has to fit the codebase and hold up to the independent review, so write it as if the reviewer will read every line.
- **Check the real thing.** Verify against the running code, not a proxy. Never hand the user a check you could have run yourself.
- **Keep the user informed but not overwhelmed.** Show task start/complete announcements. Don't narrate every file read or search.
- **If something goes wrong,** explain what happened, what you tried, and propose a fix. Don't silently retry in a loop. If a task is blocked, pause and ask the user.
- **Don't modify the plan's task list.** The user wrote/approved that plan — respect it. If a task needs a small adjustment, implement the spirit of it and note the deviation at the phase pause and in the completion summary.
- **Stop when the plan is wrong.** If the same kind of workaround keeps recurring, unrelated edge cases each need special handling, or types need escape hatches to compile, pause and tell the user. Suggest `/revise-plan` rather than building around it.

---
name: build-feature
description: "Execute an implementation plan created by /plan-feature in one go. Reads a saved plan from .claude/plans/, implements every task with recorded evidence, checks the plan's Done When criteria, then runs an independent review. Stops early only when something blocks the build. If no plan exists, directs the user to run /plan-feature first."
disable-model-invocation: true
argument-hint: "[feature-name]"
---

# Build Feature — Load Plan → Execute → Verify → Review

You are executing a pre-made implementation plan. Follow the plan rather than re-planning from scratch. Build all phases in one run, and stop early only when the build is blocked.

## Step 1: Load the Plan

Plans live in `.claude/plans/` at the **repo root** (resolve with `git rev-parse --show-toplevel`), not in the user's home `~/.claude/` folder.

Plans found when this skill started:

!`ls .claude/plans 2>/dev/null || true`

1. Check if the user provided a feature name as an argument (e.g., `/build-feature user-notifications`).
   - If yes → look for `.claude/plans/[feature-name].md`
   - If no argument → use the list above. If there's only one plan, use it automatically. Otherwise ask the user which plan to execute.

2. If the plan file doesn't exist or `.claude/plans/` is empty:

   > No plan found. Run `/plan-feature [description]` first to create one, then come back here to build it.

   Stop here.

3. Read the plan file. Check for a `## Completed` section or tasks marked as `[x]` — if some tasks are already done, resume from the first incomplete task.

4. Read the plan's `## How to Verify` section and confirm its commands still work before building on them. If the plan doesn't have that section, work out how to run and check the app, then add the section. Use `/verify`, `/run`, or a project run skill if one is available.

5. Read the plan's `## Done When` section. If it's missing, write one from the tasks' Verify steps, add it to the plan, and mention that in the final summary.

6. Display a brief summary:
   > **Loading plan: [Feature Name]**
   > [one-line description from the plan]
   > [N] phases, [M] total tasks. [K already completed — resuming from Phase X, Task Y.]
   >
   > Building all phases now. I'll stop early only if something blocks the build.

## Step 2: Execute Phase by Phase

For each phase in the plan, in order:

1. **Announce the phase:** `## Phase N: [title]`

2. **Execute each task in the phase**, in order:
   1. **Announce:** `### Task N.M: [title]`
   2. **Re-read relevant files** listed in the task's "Files" field to get current state (the plan may have been written in a different session — don't assume context).
   3. **Implement** the changes, following the project's CLAUDE.md and the conventions in the plan's Research Summary. Before writing, read a neighboring file that does something similar and search for existing helpers, so new code matches what's there and doesn't duplicate it.
   4. **Self-review the diff.** Run `git diff` on the task's changes and read it the way a reviewer would. Fix what you find now; a passing check only shows the code works, not that it's well written.
   5. **Verify** by actually running the exact method specified in the task's "Verify" field. If verification fails, fix the issue before moving on. If you can't fix it after 2 attempts, stop and tell the user what's wrong.
   6. **Record the evidence** under the task in the plan file: the command and the lines of output that show the result.

      ```markdown
      - **Evidence:** `curl -s localhost:3000/api/health` → `{"status":"ok"}`
      ```

      Evidence must come from this run. "It should work", "it compiles", or a check of something else doesn't count.
   7. **Mark complete** in the plan file by adding `[x]` to the task heading (e.g., `#### 1.1. [x] Short title`). This enables resuming if the build is interrupted. Then move to the next task.

   If the check can't run or the result is inconclusive, the task isn't done. Leave it as `[ ]` and add `- **Unverified:** [what you tried and what stopped you]`. If later tasks depend on it, stop and tell the user. Otherwise carry on and report it in the final summary.

3. **Phase check:** After completing all tasks in a phase, run relevant tests, plus the project's linter, formatter, and type-checker if it has them, and verify the phase works as a cohesive unit. Fix anything that broke, including new lint or type errors in code this build touched.

4. **Report progress and continue.** Post one line, then start the next phase without waiting:

   > **Phase N done: [title]** — [what now works]. Checks: [what ran → result].

## Step 3: Final Check

After all phases are done:

1. Run the full test suite (or relevant subset) if the project has tests.
2. Run linting and type-checking if applicable.
3. Run every check in the plan's `## Done When` section and record the result next to each one in the plan file.
4. Fix anything that broke or failed. If a Done When check still fails after 2 attempts, stop and report it rather than reviewing a build that isn't finished.

## Step 4: Independent Review

Review the build with fresh eyes before reporting it done.

1. Run the `/review-feature` skill with the plan's name. It runs in a fresh context of its own, without your build reasoning, so it judges the code rather than your explanation of it. If the skill isn't available, spawn a fresh subagent with only the plan path and the list of changed files. If you can't do either, run the review yourself and say in the summary that it wasn't independent.
2. Fix Critical and Warning findings in code this build wrote or changed. A problem that was already there is outside the plan, so report it instead of fixing it. If fixing a Warning would mean going beyond the plan's scope or changing its design, leave it and report it.
3. If you fixed anything, re-run the tests and the Done When checks, then do one more fresh review of those fixes.
4. Stop there, even if the second review finds more. Report what's left as open issues rather than starting another round.

## Step 5: Update the Plan & Summarize

1. Update the plan file — add a `## Completed` section at the bottom:

```markdown
## Completed

- **Date:** [today's date]
- **All tasks executed successfully:** [yes/no]
- **Done When:** [N/M checks pass, with any failure named]
- **Files changed:**
  - `path/to/file.ts` — [what changed]
  - ...
- **Verified:** [what was checked end to end, and the result]
- **Needs a human:** [checks you couldn't run yourself and why, or "none"]
- **Review:** [Critical and Warning findings fixed, and any issues still open]
- **Deviations:** [tasks implemented differently from the plan, and why, or "none"]
- **Follow-up items:** [anything from the Notes section that still needs attention]
```

2. Show the user a summary:

> **Build complete: [Feature Name]**
>
> [2-3 sentence summary of what was built]
>
> **By phase:**
> - Phase 1: [title] — [one line on what changed]
> - ...
>
> **Done When:** [N/M checks pass]
>
> **Verified:** [what you ran and what it showed]
>
> **Unverified:** [tasks left unchecked and why, or "none"]
>
> **Needs a human:** [only checks you couldn't run yourself, with why, or "none"]
>
> **Review:** [Critical and Warning findings fixed, plus anything left open]
>
> **Deviations:** [where you implemented the spirit of a task rather than its letter, or "none"]
>
> Plan updated at `.claude/plans/[feature-name].md` with completion details.

## Principles

- **Follow the plan.** Don't add scope, refactor unrelated code, or introduce patterns not in the plan. If you spot something worth improving, note it in the follow-up items.
- **Run to the end.** Stop early only when a fix fails twice, a task that later tasks depend on can't be verified, or the plan is wrong. Everything else goes in the summary.
- **Working isn't the bar.** Passing verification is the minimum. Code also has to fit the codebase and hold up to the independent review, so write it as if the reviewer will read every line.
- **Check the real thing.** Verify against the running code, not a proxy. Never hand the user a check you could have run yourself.
- **Keep the user informed but not overwhelmed.** Show task and phase announcements. Don't narrate every file read or search.
- **If something goes wrong,** explain what happened, what you tried, and propose a fix. Don't silently retry in a loop.
- **Don't modify the plan's task list.** The user wrote/approved that plan — respect it. If a task needs a small adjustment, implement the spirit of it and record the deviation in the completion summary.
- **Stop when the plan is wrong.** If the same kind of workaround keeps recurring, unrelated edge cases each need special handling, or types need escape hatches to compile, stop and tell the user. Suggest `/revise-plan` rather than building around it.

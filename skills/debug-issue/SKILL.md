---
name: debug-issue
description: "Structured debugging workflow for tracking down and fixing bugs. Use when the user says 'debug', 'fix this bug', 'why is this broken', 'not working', 'something is wrong', or describes unexpected behavior they need resolved. Not for quick questions about what an error message means."
argument-hint: "<description of the issue>"
effort: high
---

# Debug Issue — Reproduce → Isolate → Fix → Verify

You are debugging an issue. Follow a structured approach — don't guess and patch.

## Step 1: Understand the Problem

1. Restate the issue in your own words. Include:
   - **What's happening:** the observed behavior.
   - **What's expected:** what should happen instead.
   - **When it happens:** specific trigger, input, or conditions (if known).

2. If the user's description is too vague to act on (e.g., "it's broken"), ask one round of clarifying questions:
   - What were you doing when it happened?
   - Is there an error message or stack trace?
   - Did this work before? What changed?

## Step 2: Reproduce

1. Try to reproduce the issue. Run the relevant command, test, or workflow.
2. If you can reproduce it:
   > **Reproduced.** [describe what you saw — error message, wrong output, etc.]
3. If you can't reproduce it at first, keep trying before you ask the user. Recreate the trigger with a script or test, tighten the conditions (same input, same data, same timing), or add temporary logging and read what the code actually does. If a `run` skill or a project verification skill is available, use it.
4. If it still won't reproduce, tell the user what you tried and what specifically is stopping you, then ask for the detail you need. Remove any temporary logging before you finish.

Don't skip this step. Fixing a bug you can't reproduce is guessing.

## Step 3: Isolate

Narrow down the root cause. Work from the symptom backward:

1. **Read the error.** Stack traces, log output, and error messages are your starting point. Read the actual source at the line numbers mentioned — don't assume you know what's there.
2. **Trace the code path.** Follow the execution from the entry point to where it breaks. Read each file involved.
3. **Check recent changes.** Use `git log` and `git diff` to see what changed recently in the relevant files. The bug may have been introduced by a recent commit.
4. **Form a hypothesis.** State it clearly:
   > **Hypothesis:** [what you think is wrong and why]

If your first hypothesis doesn't hold, form a new one. Don't tunnel-vision on your first guess.

**If two fixes fail the same way, stop fixing.** Write down the one assumption both fixes shared, then find an observation that would come out differently if it were false, and make it. Don't write a third fix until you've tested that assumption.

## Step 4: Fix

1. **Announce the fix before writing it:**
   > **Fix:** [one-line description of the change]
   > **File(s):** [paths]
   > **Why this fixes it:** [brief explanation connecting the fix to the root cause]

2. **Fix it where it went wrong, not where it surfaced.** Ask whether the state that triggered the bug is legitimate. If it is, handle it in the code that broke. If it should never exist, a guard there only hides the bug, so find what produced the state.

3. **Make the minimal change.** Fix the bug, nothing else. Don't refactor surrounding code, don't "improve" nearby logic, don't add unrelated error handling. Revert anything added for a hypothesis that turned out to be wrong.

4. **Write a test for the bug** if the project has test infrastructure. The test should:
   - Be written and run before the fix, and fail for the intended reason. If it passes or fails for another reason, correct it first.
   - Pass after the fix.
   - Be a unit test unless the bug specifically involves cross-boundary behavior.

   If a cheap test isn't practical, say so and use the closest check you can run, like a script or a repro command.

5. **Check for the same fault elsewhere.** Look for other code with the same pattern. Fix it if it's part of this bug; otherwise list it in the report.

## Step 5: Verify

1. Run the reproduction step again — the issue should be gone.
2. Run the relevant test suite to make sure nothing else broke.
3. If the fix introduced new failures, investigate and resolve before reporting done.

## Step 6: Report

> **Fixed: [short title]**
>
> **Root cause:** [1-2 sentences — what was actually wrong]
>
> **Fix:** [what you changed and in which files]
>
> **Test:** [test added and how it failed before the fix, or why not]
>
> **Verified:** [how you confirmed it works]
>
> **Same fault elsewhere:** [where else it appears, or "none found"]

## Principles

- **Trace, don't grep.** Searching for keywords finds where code lives. Tracing execution finds why it breaks. Do both, but prioritize tracing.
- **One fix at a time.** If you change three things and the bug goes away, you don't know which one fixed it. Make one change, verify, repeat if needed.

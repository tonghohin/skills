---
name: review-feature
description: "Review a change against what it was meant to do: the plan in .claude/plans/, its Done When checks, and the evidence /build-feature recorded. Checks that every requirement is implemented and correct, and proves each finding before reporting it. Use when the user says 'review', 'check my code', 'review against the plan', 'did the build match the plan', 'sanity check', or after /build-feature. Without a plan, reviews the change against its stated intent. For a broad bug sweep use /code-review, for cleanup use /simplify, and for a pull request use /review-pr."
argument-hint: "[feature-name or branch]"
context: fork
background: false
effort: high
disallowed-tools: Edit, Write, NotebookEdit
---

# Review Feature — Load Intent → Check Against Plan → Review → Prove → Report

You are reviewing a change against what it was meant to do. You start in a fresh context, without the conversation that produced the code, and that is the point: judge the code, not anyone's explanation of it. You report; you don't fix.

Look for the gaps that matter: a requirement that isn't met, a bug, a check that doesn't prove what it claims. A reviewer asked to find problems will always find something, and fixing every finding grows extra abstraction and tests for cases that can't happen. So report what affects correctness or the plan, prove it, and leave style alone. Cleanup belongs to `/simplify`, and a broad bug sweep of the diff to `/code-review`.

## Step 1: Load the Intent and the Change

1. Work out what to review from the argument.
   - A feature name → load `.claude/plans/[feature-name].md` from the **repo root** (resolve with `git rev-parse --show-toplevel`), not the user's home `~/.claude/` folder. The plan is the spec.
   - A branch → review the diff between that branch and main/master. If a plan in `.claude/plans/` matches the branch name, load it.
   - No argument → review uncommitted changes (staged + unstaged). If there are none, review the most recent commit.

   Without a plan, take the intent from the commit messages, the branch name, and the request you were given. State it in one or two sentences before reviewing. If you can't, say so and review for correctness only.

2. Gather the changes:
   - Use `git diff` for uncommitted changes, `git diff main...HEAD` for a branch, or `git show` for the last commit.
   - Read the full content of every changed file, since diffs alone miss context like missing imports or broken callers.
   - Read the files that call into the changed code. A change that looks correct in isolation can break its callers.
   - Then look where a symbol search stops: JSON an API returns, DB columns, wire formats, feature flags, other languages reading the same data, and the pinned source of any library the change depends on.

## Step 2: Check Against the Plan

Skip this step if no plan was loaded.

- **Tasks.** For each task, find where it's implemented and point to `file:line`. A task marked `[x]` that isn't in the code, or is only partly there, is a finding.
- **Evidence.** Re-run the `Evidence` checks for the riskiest tasks and for any whose evidence looks thin, rather than trusting them. A result that doesn't match, or a `[x]` task with no evidence, is a finding. Flag every `Unverified` task.
- **Done When.** Run each check in the plan's `## Done When` section and record pass or fail with the output. A failing check is Critical.
- **Beyond the plan.** Note anything built that the plan didn't ask for. It's a finding only if it adds risk or a cost the next person will pay.

## Step 3: Review for Correctness

Read every changed line.

### Correctness
- Does the logic do what was intended, for the inputs it will really get?
- Off-by-one errors, null or undefined cases, race conditions, unhandled error paths.
- Empty, large, and concurrent input. A dependency that returns null, throws, or times out. A caller that ignores the return value or error.
- Callers or consumers of the changed code that will now behave incorrectly.
- Logic that should exist but doesn't: validation, error handling, cleanup, a migration.

### Security
- New endpoints or routes without the auth or ownership checks their neighbors have, user input reaching a query, command, or HTML unchecked, and credentials or tokens hardcoded or logged. For a full security pass, the user can run `/security-review`.

### Tests
- Does each requirement and Done When check have a test or a check that covers it?
- Judge each new or changed test against two imagined edits. Break the behavior: if the test still passes, it proves nothing. Keep the behavior but restructure the internals: if the test now fails, it's pinned to the implementation.

### Fit
Only what breaks a written rule or will cost the next person real time:
- A rule in the project's `CLAUDE.md`, or its lint, format, or type-check config. Run those tools if the project has them.
- A helper, type, or dependency that duplicates one the project already has.
- Shared mutable state, such as a module-level object that functions modify. On a server, state shared across requests is Critical.
- A type escape hatch (`any`, an unchecked cast, an `ignore` comment) that hides a real type error.

Naming, structure, comments, and other matters of taste are out of scope.

Stay inside the change. A problem in code the change didn't touch counts only if the change makes it worse or depends on it.

## Step 4: Challenge Your Findings

For each finding, ask whether it's really a bug or whether you misread the context. The most common wrong finding comes from reading one function and missing that the caller, the type, upstream validation, the framework, or a line twenty rows up already handles the case. Read those before keeping it. Drop what you disprove and what CI or a linter already reports.

Then take a second look at the riskiest code you passed over: auth, data mutations, external calls, error paths. If it holds up, it holds up. Finding nothing is a valid result.

## Step 5: Prove What Matters

A review that sounds right reads the same whether or not it's true, so check the claims that matter most.

1. **Find the safety fact.** Most risky-looking changes are safe because of one fact, like "this call only drops expired cache entries". State that fact. If it holds, many risks clear at once.
2. **Take each claim as far as is cheap.** Do this for the safety fact and every finding, and record where it stopped:
   1. **Stated:** you said so. Worth nothing alone.
   2. **Cited:** you pointed at a real `file:line` or the library's source.
   3. **Traced:** you walked the failure step by step.
   4. **Ran:** a test or script calls the real code and shows the result.
   5. **Reproduced:** you saw it in the running app.
3. **Keep the repo clean.** Put throwaway scripts outside the repo, and leave no changes behind.

A claim you can't prove stays in the report, labeled with the level it reached.

## Step 6: Report

> ### Critical
> A bug, data loss, a security hole, or a plan requirement or Done When check that isn't met. Must be fixed.
>
> ### Warnings
> A real problem that will bite later: a requirement with no test, a broken project rule, unverified evidence, a caller that will break. Should be fixed.

For each finding:
- **What:** one specific line. Not "this might have issues", but "`userId` can be null here because `getUser()` returns `null` when the session expires".
- **Where:** file path and line number(s).
- **Why:** what goes wrong if this isn't fixed.
- **Proof:** the level it reached in Step 5, with the command and output if you ran something.
- **Fix:** a concrete suggestion. Show the code change when it's small.

Skip an empty category rather than writing "no issues found".

If a plan was loaded, add:

> ### Plan vs. Implementation
> - **Done When:** [N/M checks pass, with each failure named]
> - **Tasks without evidence:** [every task marked `[x]` with no `Evidence` line, and every `Unverified` task, or "none" — check each task in the plan file, don't estimate]
> - **Tasks completed as planned:** [list]
> - **Deviations:** [what was changed and why, if detectable]
> - **Missing from plan:** [tasks from the plan not reflected in the changes]

## Step 7: Summary

> **Review complete: [feature name]**
>
> [Critical: N] [Warnings: N]
>
> **Safe because:** [the safety fact, and the level it reached — or "unproven"]
>
> **Reviewed:** [feature / branch / commit] against [plan path, or "stated intent"]
>
> [One sentence overall assessment — is this ready to ship, or does it need work?]

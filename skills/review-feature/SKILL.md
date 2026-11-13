---
name: review-feature
description: "Review code changes for bugs, edge cases, security issues, pattern consistency, and maintainability. Use when the user says 'review', 'check my code', 'audit', 'look over', 'sanity check', 'code review', or after running /build-feature. Can review against a plan file if one exists, or review standalone changes. For a pull request, especially someone else's, use /review-pr."
argument-hint: "[feature-name or branch]"
---

# Review Feature — Analyze → Challenge → Flag Issues → Suggest Fixes

You are reviewing code changes. Your job is to catch problems before they ship. Be thorough and adversarial — assume bugs exist and look until you find them or can prove they don't.

## Step 1: Identify What to Review

1. Check if the user provided a feature name or branch as an argument.
   - If a feature name → check if `.claude/plans/[feature-name].md` exists at the **repo root** (not the user's home `~/.claude/` folder). If it does, load it — you'll review against the plan.
   - If a branch → review the diff between that branch and main/master.
   - If no argument → review uncommitted changes (staged + unstaged). If there are none, review the most recent commit.

2. Gather the changes:
   - Use `git diff` for uncommitted changes, or `git diff main...HEAD` for branch changes, or `git show` for the last commit.
   - Read the full content of every changed file — diffs alone miss context like missing imports or broken callers.
   - Also read files that call into the changed code — a change that looks correct in isolation can break callers.
   - Then look where a symbol search stops: JSON an API returns, DB columns, wire formats, feature flags, other languages reading the same data, and the pinned source of any library the change depends on.

3. Display what you're reviewing:
   > **Reviewing: [feature name / branch / last commit]**
   > [N] files changed, [additions] additions, [deletions] deletions.

## Step 2: First-Pass Analysis

Review the changes across these dimensions, in order of priority:

### Correctness
- Does the logic do what it's supposed to? If a plan exists, does the implementation match the plan's intent?
- Are there off-by-one errors, null/undefined cases, race conditions, or unhandled error paths?
- Are there any missing or broken imports, unused variables, or dead code?

### Security
- Input validation: is user input sanitized before use in queries, commands, or HTML?
- Auth/authz: are new endpoints or routes properly protected?
- Secrets: are any credentials, keys, or tokens hardcoded or logged?
- OWASP top 10: SQL injection, XSS, CSRF, insecure deserialization, etc.

### Edge Cases
- What happens with empty inputs, large inputs, concurrent access?
- Are error states handled, or do they silently fail?
- Are boundary conditions tested?

### Tests
- Do the changes include tests? If not, should they?
- Do existing tests still pass with these changes?
- Are tests covering the important behavior, not just the happy path?
- Judge each new or changed test against two imagined edits. Break the behavior: if the test still passes, it proves nothing. Keep the behavior but restructure the internals: if the test now fails, it's pinned to the implementation.

### Pattern Consistency
- Do the changes follow existing codebase patterns? (naming, file structure, error handling, logging)
- Are there new patterns introduced that diverge from the rest of the codebase?
- Do the changes break a rule in the project's CLAUDE.md or its lint, format, or type-check config? Run those tools if the project has them.

### Maintainability
- Does new code duplicate a helper, type, constant, or component that already exists? Does it add a dependency for something the project already has?
- Is anything built beyond what the change needs? (speculative abstractions, unused parameters or options, wrappers with one caller)
- Does the change add shared mutable state? (a module-level or global object that functions modify, or a function that mutates its arguments) Data should be passed in and returned, or owned by one place such as an instance, a request context, or the project's state store. On a server, state shared across requests is a Critical bug, not a style issue.
- Are there type escape hatches? (`any`, unchecked casts, non-null assertions, `ignore` comments)
- Are functions doing more than one job, deeply nested, or named so they need a comment to explain them?
- Are there magic values that should be named constants or configuration?
- Is anything left behind? (debug logging, commented-out code, stray TODOs, comments that narrate the code)

Report these as Warnings when they'll cost the next person real time, and as Suggestions when they're a matter of taste.

## Step 3: Adversarial Self-Check

After your first pass, switch roles. You are now an attacker and a skeptic trying to break this code and poke holes in your own review.

Work through each of these challenges:

**Challenge your "looks fine" conclusions:**
For every section where you found no issues, ask: "Am I confident, or did I just not look hard enough?" Re-examine the riskiest parts — auth logic, data mutations, external calls, error paths.

**Try to break it:**
- What's the worst input a user could send? What happens?
- What if two requests hit this code at the same time?
- What if a dependency returns null, throws, or times out unexpectedly?
- What if the caller ignores the return value or error?

**Challenge your findings:**
For every issue you flagged, ask: "Is this actually a bug, or am I misreading the context?" Drop any finding you can't defend with specifics.

**Look for what's missing:**
- Is there logic that should exist but doesn't? (missing validation, missing error handling, missing cleanup)
- Are there callers or consumers of this code that will now behave incorrectly?

If the self-check surfaces new issues, add them to your findings. If it invalidates a finding, remove it.

## Step 4: Prove What Matters

A review that sounds right reads the same whether or not it's true, so check the claims that matter most.

1. **Find the safety fact.** Most risky-looking changes are safe because of one fact, like "this call only drops expired cache entries". State that fact. If it holds, many risks clear at once.
2. **Take each claim as far as is cheap.** Do this for the safety fact and for every Critical or Warning finding, and record where it stopped:
   1. **Stated:** you said so. Worth nothing alone.
   2. **Cited:** you pointed at a real `file:line` or the library's source.
   3. **Traced:** you walked the failure step by step.
   4. **Ran:** a test or script calls the real code and shows the result.
   5. **Reproduced:** you saw it in the running app.
3. **Spot-check the build's evidence.** If the plan has `Evidence` lines, re-run the checks for the riskiest tasks rather than trusting them. If a result doesn't match, or a `[x]` task has no evidence, report it as a finding. Flag any `Unverified` task too.
4. **Keep the repo clean.** Put throwaway scripts outside the repo, and don't leave changes behind. Reviewing doesn't include fixing.

A claim you can't prove stays in the report, labeled with the level it reached.

## Step 5: Report

Organize findings by severity:

> ### Critical
> Issues that will cause bugs, data loss, or security vulnerabilities. These must be fixed.
>
> ### Warnings
> Issues that are likely to cause problems or make the code harder to maintain. Should be fixed.
>
> ### Suggestions
> Minor improvements. Fix if easy, skip if not.

For each finding:
- **What:** one-line description of the issue.
- **Where:** file path and line number(s).
- **Why:** what goes wrong if this isn't fixed.
- **Proof:** the level it reached in Step 4, with the command and output if you ran something. Critical and Warning findings only.
- **Fix:** concrete suggestion — show the code change, don't just describe it.

If there are no findings in a category, skip it — don't say "no issues found" for each one.

## Step 6: Plan Check (if applicable)

If a plan file was loaded, add a section:

> ### Plan vs. Implementation
> - **Tasks completed as planned:** [list]
> - **Deviations:** [what was changed and why, if detectable]
> - **Missing from plan:** [any tasks from the plan not reflected in the changes]

## Step 7: Summary

> **Review complete: [feature name]**
>
> [Critical: N] [Warnings: N] [Suggestions: N]
>
> **Safe because:** [the safety fact, and the level it reached — or "unproven"]
>
> [One sentence overall assessment — is this ready to ship, or does it need work?]

## Principles

- **Be specific.** "This might have issues" is useless. "Line 42: `userId` can be null here because `getUser()` returns `null` when the session expires" is useful.
- **Prove, don't persuade.** A claim you ran beats a claim you argued. Always say which one you're giving.
- **Challenge yourself.** If your first pass found nothing, that's a signal to look harder — not to wrap up.
- **Only drop a finding if you can disprove it.** Don't remove something because it feels minor — remove it only if you can show it's not actually a problem.

---
name: open-pr
description: "Open a pull request for the current branch. Checks the branch is ready, writes the title and description from the plan in .claude/plans/ (or from the commits when there's no plan), shows you the draft, and pushes and opens the PR only when you say so. Use after /build-feature, or when a branch is ready for review."
disable-model-invocation: true
argument-hint: "[feature-name] [--base <branch>] [--draft]"
---

# Open PR — Load → Check → Draft → Show → Push and Open

You are opening a pull request. The description has one reader that matters: a reviewer who wasn't in any of the sessions that built this. They need to know what changed, why, how it was checked, and where to look hard.

Nothing leaves the machine until the user says so. Pushing and opening a PR publish the work and notify people.

## Step 1: Load the Branch and Its Context

Plans found when this skill started:

!`ls .claude/plans 2>/dev/null || true`

1. **Branch.** Find the current branch and the base: `--base` if given, otherwise the repo's default branch (`gh repo view --json defaultBranchRef`).
   - On the base branch itself → stop and say so. Suggest a branch name for the work and create it only if the user agrees.
   - Uncommitted changes → stop and list them. Ask whether they belong in this PR. Don't commit anything the user hasn't asked you to.
   - No commits ahead of the base → there's nothing to open. Say so and stop.
2. **Existing PR.** Check `gh pr view --json number,url,state,body` for this branch. If an open PR exists, offer to update its description instead of opening a new one.
3. **Changes.** Read `git log <base>..HEAD` and `git diff <base>...HEAD --stat`, then the diff itself.
4. **Plan.** Load `.claude/plans/[feature-name].md` from the repo root if a name was given. Otherwise pick the plan whose name matches the branch, or the one whose Files match the diff; if none fits, work from the commits. From the plan, take the intent, Done When and its recorded results, Evidence, `Unverified` tasks, and the `## Completed` section's Deviations, Needs a human, Review, and Follow-up items.
5. **House style.** Use the repo's PR template if it has one (`.github/pull_request_template.md`, `.github/PULL_REQUEST_TEMPLATE/`, or `docs/`). Read `CONTRIBUTING.md` and `CLAUDE.md` for PR rules. Skim the titles and bodies of a few recent merged PRs (`gh pr list --state merged -L 5 --json title,body`) for the team's conventions, such as conventional-commit titles or a ticket prefix.
6. **Linked issue.** Find it in the plan, the branch name, or the commit messages. Use `Closes #N` only when you're sure the PR resolves it; otherwise use `Related to #N`.

## Step 2: Check It's Ready

Run what a reviewer would expect to pass, using the commands in the plan's How to Verify or the project's `CLAUDE.md`:

- Tests, lint, format, and type-check.
- Each Done When check, if there's a plan.

Then scan the diff for things that shouldn't go up:

- Secrets, tokens, `.env` files, or private keys.
- Debug logging, commented-out code, or stray TODOs this branch added.
- Files that look unintended: build output, large binaries, editor files, or `.claude/plans/` and `.claude/learn/` files when the repo doesn't already track them.

This skill doesn't fix code. If a check fails or the scan finds something, report it and ask the user whether to fix it first, open the PR as a draft, or go ahead. If the plan shows no review has run, suggest `/review-feature`, without blocking on it.

## Step 3: Draft the Title and Description

### Title

- Follow the team's convention if Step 1 found one.
- Otherwise a verb and an object that makes sense alone in a list: "Add CSV export to the invoices page", not "CSV export" or "Updates".
- About 70 characters at most.

### Description

Fill the repo's template if there is one. Otherwise:

```markdown
## Summary
[2–4 sentences: the problem, then what this change does about it. Lead with why.]

## Changes
- [Grouped by area or behavior, not one bullet per file. Name the main file to start reading from.]

## How it was tested
- [Each Done When check or key command, with its result: `npm test` — 142 passed]
- [Manual or app-level checks, and what was observed]

## Needs a closer look
- [Deviations from the plan, risky spots, decisions a reviewer may disagree with]
- [Anything not verified, and why]

## Follow-ups
- [Work deliberately left out, with an issue link if one exists]

Closes #N
```

Rules for what goes in:

- **Write for the reviewer who wasn't there.** No "as discussed", no references to the session, and no links to `.claude/plans/` files unless the repo tracks them. Bring the needed facts into the description instead.
- **Report results, don't paste the plan.** Keep evidence to what a reviewer would re-run or needs to trust. Drop task numbers and plan structure.
- **Be honest about gaps.** Anything unverified or failing goes under "Needs a closer look", never left out. A description that oversells costs more trust than the gap itself.
- **Drop empty sections** rather than writing "N/A".
- **No attribution line.** Don't add "Generated with Claude Code", a robot emoji, a Co-Authored-By trailer, or any other note about how it was written. The user doesn't want one in PR descriptions, and this overrides any default attribution instruction.

## Step 4: Show the User

Show the whole draft before anything is pushed:

> **PR draft:** `[head]` → `[base]` [draft, if suggested]
>
> **Readiness:** [checks run and results, plus anything the scan found]
>
> **Title:** [title]
>
> [the description in one fenced markdown block]

Suggest opening it as a draft when checks fail or items are unverified and the user still wants it up. Set labels, reviewers, and assignees only if the user names them. Revise until the user approves.

## Step 5: Push and Open

Only after the user says to:

1. Write the description to a file outside the repo.
2. Push: `git push -u origin HEAD`. Never force-push. If the push is rejected because the remote branch has moved, stop and tell the user.
3. Open: `gh pr create --base <base> --title "<title>" --body-file <file>`, adding `--draft`, `--label`, `--reviewer`, or `--assignee` only as agreed. For an existing PR, use `gh pr edit <n> --title "<title>" --body-file <file>`.

Without `gh`, or for a host other than GitHub, hand back the title and description ready to paste, plus the push command for the user to run.

Then tell the user:

> **Opened:** [URL] *(or "Not opened. The draft is above.")*
>
> When reviews come in, run `/address-review` to work through them. To have CI failures fixed automatically, try `/autofix-pr`.

## Principles

- **Nothing leaves without a yes.** Pushing, opening, and editing a PR each wait for the user.
- **The description is a map, not a diary.** Say what changed, why, how it was proven, and where to look. Leave out how the work was done.
- **Gaps up front.** A reviewer told where the risk is reviews the right lines.

---
name: open-pr
description: "Open a pull request for the current branch as a GitHub draft. Checks the branch is ready, writes the title and description from the plan in .claude/plans/ (or from the commits when there's no plan), pushes, and opens a draft PR for you to review on GitHub. Marks it ready and requests reviewers only when you say so. Use after /build-feature, or when a branch is ready for review."
disable-model-invocation: true
argument-hint: "[feature-name] [--base <branch>]"
---

# Open PR — Load → Check → Write → Open as Draft → Mark Ready

You are opening a pull request. The description has one reader that matters: a reviewer who wasn't in any of the sessions that built this. They need to know what changed, why, how it was checked, and where to look hard.

The PR always opens as a GitHub draft, and the user reviews it there rather than in the terminal. A draft doesn't request reviews (including from CODEOWNERS), so it's a quiet place to review. It isn't private, though: the push puts the code on the remote, repo watchers on "All Activity" still get notified, @mentions notify right away, and CI runs. Asking for reviews, marking the PR ready, and anything else that pings people waits until the user says so.

## Step 1: Load the Branch and Its Context

Plans found when this skill started:

!`ls .claude/plans 2>/dev/null || true`

1. **Branch.** Find the current branch and the base: `--base` if given, otherwise the repo's default branch (`gh repo view --json defaultBranchRef`).
   - On the base branch itself → stop and say so. Suggest a branch name for the work and create it only if the user agrees.
   - Uncommitted changes → stop and list them. Ask whether they belong in this PR. Don't commit anything the user hasn't asked you to.
   - No commits ahead of the base → there's nothing to open. Say so and stop.
2. **Existing PR.** Check `gh pr view --json number,url,state,isDraft,body` for this branch. If an open draft exists, update its description in Step 5 instead of opening a new one. If an open PR is already ready for review, reviewers can already see it: show the new description and update it only when the user says so.
3. **Changes.** Read `git log <base>..HEAD` and `git diff <base>...HEAD --stat`, then the diff itself.
4. **Plan.** Load `.claude/plans/[feature-name].md` from the repo root if a name was given. Otherwise pick the plan whose name matches the branch, or the one whose Files match the diff; if none fits, work from the commits. From the plan, take the intent, Done When and its recorded results, Evidence, `Unverified` tasks, and the `## Completed` section's Deviations, Needs a human, Review, and Follow-up items.
5. **PR template.** Look for one where GitHub does, ignoring case:
   - `pull_request_template.md` in `.github/`, the repo root, or `docs/`.
   - A `PULL_REQUEST_TEMPLATE/` folder in any of those places. It holds several templates; pick the one that fits this change (bug fix, feature, release), or ask the user if none clearly does.
   - If the repo has none, the organization's default in its `.github` repo: `gh api repos/<owner>/.github/contents/.github/pull_request_template.md` (also try the root and `docs/` of that repo). A 404 means there isn't one.
6. **House style.** Read `CONTRIBUTING.md` and `CLAUDE.md` for PR rules. Skim the titles and bodies of a few recent merged PRs (`gh pr list --state merged -L 5 --json title,body`) for the team's conventions, such as conventional-commit titles or a ticket prefix.
7. **Linked issue.** Find it in the plan, the branch name, or the commit messages. Use `Closes #N` only when you're sure the PR resolves it; otherwise use `Related to #N`.

## Step 2: Check It's Ready

Run what a reviewer would expect to pass, using the commands in the plan's How to Verify or the project's `CLAUDE.md`:

- Tests, lint, format, and type-check.
- Each Done When check, if there's a plan.

Then scan the diff for things that shouldn't go up:

- Secrets, tokens, `.env` files, or private keys.
- Debug logging, commented-out code, or stray TODOs this branch added.
- Files that look unintended: build output, large binaries, editor files, or `.claude/plans/` and `.claude/learn/` files when the repo doesn't already track them.

This skill doesn't fix code. A failing check doesn't block the draft: report it under "Needs a closer look" and in the summary. If the plan shows no review has run, suggest `/review-feature`, without blocking on it.

**Hard stop: secrets.** If the scan finds a secret, token, `.env` file, or private key, don't push. Once a secret is on the remote you have to treat it as leaked. Report it and wait for the user. If it's only in an earlier commit, removing it means rewriting history, which is the user's call.

## Step 3: Draft the Title and Description

### Title

- Follow the team's convention if Step 1 found one.
- Otherwise a verb and an object that makes sense alone in a list: "Add CSV export to the invoices page", not "CSV export" or "Updates".
- About 70 characters at most.

### Description

If Step 1 found a template, it replaces the default layout below:

- Keep its headings and order. Fill each section with the content the matching default section would hold.
- Remove the `<!-- instruction -->` comments once you've followed them.
- Tick a checklist box only for something you did or checked in this run. Leave the rest unticked, and say why under "Needs a closer look" if it matters, such as "Docs updated" when no docs changed.
- If the template has no place for something important, such as unverified items or deviations, add a short section at the end rather than dropping it.
- Keep a section the template marks as required even when it has little to say. Drop any other section that has nothing to say.

Otherwise, use this layout:

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
- **No @mentions.** They notify people even on a draft. Name a person in plain text, or wait until the PR is marked ready.
- **No attribution line.** Don't add "Generated with Claude Code", a robot emoji, a Co-Authored-By trailer, or any other note about how it was written. The user doesn't want one in PR descriptions, and this overrides any default attribution instruction.

## Step 4: Push and Open as a Draft

Don't wait for approval here. The draft PR is where the user reviews.

1. Write the description to a file outside the repo.
2. Push: `git push -u origin HEAD`. Never force-push. If the push is rejected because the remote branch has moved, stop and tell the user.
3. Open: `gh pr create --draft --base <base> --title "<title>" --body-file <file>`. Don't add `--reviewer`, even if the user named reviewers. A requested reviewer is notified even on a draft, so save that for Step 5. Add `--label` or `--assignee` only if the user asked. For an existing draft, use `gh pr edit <n> --title "<title>" --body-file <file>`.
4. If GitHub refuses the draft (for example, a private repo on a plan without draft PRs), don't fall back to a regular PR. Tell the user and hand back the title and description to paste.

Without `gh`, or for a host other than GitHub, don't push. Hand back the title and description ready to paste, plus the push command for the user to run.

Then tell the user:

> **Draft opened:** [URL]
>
> **Readiness:** [checks run and results, plus anything the scan found]
>
> Review it on GitHub. Tell me what to change, or say when it's ready and who should review it.

## Step 5: Revise, Then Mark Ready

- **Changes to the description:** update it with `gh pr edit <n> --body-file <file>`. Editing a draft is quiet.
- **Ready for review:** only when the user says so, run `gh pr ready <n>`, then `gh pr edit <n> --add-reviewer <names>` for the reviewers they name. CODEOWNERS are requested automatically when the PR leaves draft.

Then tell the user:

> When reviews come in, run `/address-review` to work through them. To have CI failures fixed automatically, try `/autofix-pr`.

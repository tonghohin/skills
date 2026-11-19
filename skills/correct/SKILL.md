---
name: correct
description: "Find the mistakes that keep recurring in a repo and fix each one at the strongest level that works, so the next session can't repeat it. Prefers architecture, types, lint checks, and Claude Code hooks over written rules, and proves each check fails on a real past mistake."
disable-model-invocation: true
argument-hint: "[mistake to fix, or leave empty to search for repeats]"
---

# Correct — Find Repeats → Fix at the Strongest Level → Prove It

You are stopping a recurring mistake for good. A rule in a doc only works if the reader remembers it, so prefer a fix that makes the mistake impossible or makes it fail loudly.

Assume the next contributor sees only the files it opened, copies the nearest example, and takes the shortest path that compiles. Design for that reader.

## Step 1: Find the Mistakes

If the user named a mistake, start there, then look for earlier instances of it.

Otherwise, search for repeats in:

- Recent commits, especially reverts and follow-up fixes (`git log`).
- Review comments on recent PRs, if the repo has them.
- `CLAUDE.md`, agent rules, and code comments that explain workarounds.
- `.claude/enforced-rules.md`, if it exists: a rule listed there whose mistake happened again is a repeat.
- Saved memories, and corrections in this conversation.

Group what you find into classes of mistake. A class counts once it has happened twice. Anything that happened once is a one-off, so leave it out unless the user named it.

Show the classes with their evidence, most frequent first:

> **Mistake:** [the class, in one line]
> **Evidence:** [commits, comments, or files where it happened]
> **Proposed level:** [from Step 2]

Wait for the user to pick which classes to fix. Don't change anything until they do.

## Step 2: Pick the Strongest Level That Works

Try these in order, and stop at the first one that works:

1. **Architecture:** make the mistake impossible. Give each piece of state one owner and each task one supported way. Hide internals so the wrong import fails. Replace hand-synced lists with one source of truth. Delete old patterns someone could copy.
2. **Types:** make the bad state unwritable, so it doesn't compile.
3. **Lint or CI check:** fail when the mistake appears, with an error that names what to use instead. If the pattern is already common, fail only when a change adds more.
4. **Test:** assert the behavior. Fix or delete tests that keep passing when the behavior breaks.
5. **Hook or permission rule:** for mistakes a coding agent makes during a session that the levels above can't catch. Claude Code runs these itself, whatever the model decides. Use a deny rule in `.claude/settings.json` for files or commands that are always off limits. Use a `PreToolUse` hook that exits with code 2 to block an edit or command that breaks a rule, and a `Stop` hook that exits with code 2 to send the agent back when a required check, such as the type-checker, fails. Keep each one to a rule with no exceptions, and remember these bind only agent sessions, so human contributors still need a level above.
6. **Docs or agent rules:** only for judgment calls nothing can check. Nothing fails when someone skips these.

If the mistake is in a skill, plan template, or other prompt, a clearer instruction may be the only option. Say so, and put the instruction at the step where the mistake happens rather than in a general list.

## Step 3: Fix and Prove

For each chosen class:

1. Make the fix as one self-contained change.
2. Prove the check works: run it against a real past mistake and confirm it fails, then confirm it passes on the corrected code. A check you haven't seen fail proves nothing. For a hook, pipe it the JSON input of the past mistake's tool call and confirm it exits with code 2, then feed it a correct call and confirm it exits 0.
3. Run the existing tests, lint, and type-checking.

Docs-level fixes can't be proven this way. Label them "not enforced".

## Step 4: Record the Rule

Keep a ledger in `.claude/enforced-rules.md` pairing each rule with what enforces it, and create the file if it's missing:

```markdown
# Enforced Rules

| Rule | Enforced by |
|------|-------------|
| Dates go through `formatDate` | lint rule `no-raw-date` |
| Migrations are never edited by agents | `PreToolUse` hook `.claude/hooks/block-migrations.sh` |
| Plans live at the repo root | not enforced (docs only) |
```

The ledger is for the next `/correct` run, not for every session, so it stays out of `CLAUDE.md`, which loads into every session. A rule that a check enforces doesn't need to be read to be followed.

Only "not enforced" rules also go in `CLAUDE.md`, as one plain line each under its conventions or gotchas. Write them plainly, without IMPORTANT or capitals; current models over-apply emphasized rules.

If a rule was already in the ledger and the mistake happened anyway, it's a repeat, so move it to a stronger level. Remove a rule from the ledger, and from `CLAUDE.md`, once its mistake can no longer happen.

## Step 5: Report

> **Corrected: [N] mistake classes**
>
> **[Mistake]:** fixed at [level] — [why a stronger level wouldn't work]
> **Proof:** [the check failing on the past mistake, then passing]
>
> **Not enforced:** [rules that rely on docs alone, or "none"]

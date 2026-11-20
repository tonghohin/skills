# Claude Code Skills

A set of skills for structured development with Claude Code: plan before you build, revise plans as scope shifts, and review or debug existing code with repeatable workflows.

The core loop is plan → build → ship, with `/revise-plan` for mid-flight changes and `/open-pr` and `/address-review` to get the work merged. The remaining skills cover ticket writing, review, pull request review, debugging, learning a new codebase, and stopping recurring mistakes — each is a standalone workflow, usable on its own.

## How the plan → build loop works

```
/plan-feature add user notifications with email and in-app support
    │
    ├── Asks about each open decision that would change what gets built, and nothing else
    ├── Researches your codebase (structure, patterns, conventions)
    ├── Works out how to run and check the app
    ├── Writes Done When: the pass/fail checks that mean it's finished
    ├── Creates a detailed task breakdown
    └── Saves to .claude/plans/user-notifications.md

    ⏸️  You review the plan. Edit it. Sleep on it. Share it.
    ⏸️  Scope changed? → /revise-plan user-notifications <what to change>

/build-feature user-notifications
    │
    ├── Reads the saved plan
    ├── Executes every phase in one go, skipping tasks already marked [x]
    ├── Verifies each task for real and saves the evidence in the plan
    ├── Runs the plan's Done When checks
    ├── Runs an independent /review-feature and fixes Critical and Warning findings
    └── Updates the plan file with completion details

/open-pr user-notifications
    │
    ├── Re-runs the checks and scans the diff for things that shouldn't go up
    ├── Writes the PR description from the plan's results, gaps, and deviations
    └── Pushes and opens the PR only when you say so

/address-review
    │
    ├── Checks each review comment against the code
    ├── Fixes the right ones as local commits, drafts evidence-backed replies to the rest
    └── Pushes and posts replies only when you say so
```

## Skills

| Skill | Trigger | What it does |
|-------|---------|--------------|
| `/write-ticket` | "write a ticket", "file a bug", "create an issue" | Research codebase → interview you on the gaps → draft a ticket → file it in your tracker or hand back markdown |
| `/plan-feature` | "plan", "scope", "break down", "design" | Research codebase → create task plan → save to `.claude/plans/` |
| `/build-feature` | Manual: type `/build-feature` | Load plan from `.claude/plans/` → execute all phases in one go → verify → check Done When → independent review → summarize |
| `/revise-plan` | "change the plan", "add a task to", "swap X for Y" | Edit an existing plan; preserves `[x]` markers so in-flight builds resume cleanly |
| `/open-pr` | Manual: type `/open-pr` | Check the branch is ready → draft the PR title and description from the plan → push and open only when you say so |
| `/address-review` | Manual: type `/address-review` | Triage review comments on your PR → fix the right ones → draft replies with evidence → push and post only when you say so |
| `/review-feature` | "review", "check my code", "review against the plan" | In a fresh context, check a change against its plan and Done When, re-run the build's evidence, and report only proven correctness gaps |
| `/review-pr` | "review PR 123", "look at this PR", a pasted PR link | Read the intent → check design → verify each finding → draft a verdict and labeled inline comments → post to GitHub only when you say so |
| `/debug-issue` | "debug", "fix this bug", "why is this broken" | Structured debugging: reproduce → isolate → fix → verify |
| `/learn-codebase` | "walk me through", "onboard me", "explain this repo" | Deep dive: explore the repo → trace real flows → save a record to `.claude/learn/` |
| `/correct` | Manual: type `/correct` | Find repeated mistakes → fix each at the strongest level → prove the check fails on a past mistake |

`/build-feature` and `/correct` change code across a project, and `/open-pr` and `/address-review` push code and notify people, so these four only run when you type them. The others can also start from what you say.

## Install

This repo is a Claude Code plugin marketplace. Install the plugin once and it's available in all your projects:

```
/plugin marketplace add tonghohin/skills
/plugin install dev@tonghohin-skills
```

Or from the shell:

```bash
claude plugin marketplace add tonghohin/skills
claude plugin install dev@tonghohin-skills
```

Skills are namespaced by the plugin, so you can type either `/plan-feature` or `/dev:plan-feature`.

### Updating

After pushing changes to this repo, pull them in with:

```bash
claude plugin marketplace update tonghohin-skills
claude plugin update dev@tonghohin-skills
```

Then restart Claude Code. Bump `version` in `.claude-plugin/plugin.json` (and the matching entry in `.claude-plugin/marketplace.json`) when you release changes, so the update is picked up.

### Developing locally

To try edits without pushing, load the plugin straight from your checkout:

```bash
claude --plugin-dir /path/to/skills
```

### Sharing with a team

Add the marketplace and plugin to a project's `.claude/settings.json` so teammates get prompted to install it:

```json
{
  "extraKnownMarketplaces": {
    "tonghohin-skills": {
      "source": { "source": "github", "repo": "tonghohin/skills" }
    }
  },
  "enabledPlugins": {
    "dev@tonghohin-skills": true
  }
}
```

## Usage

### Write a ticket

```
/write-ticket exports time out for accounts with more than 10k invoices
```

Claude reads the relevant code first, then interviews you in short rounds about what only you know: why it matters, what "done" looks like, what happens on failure, and what's out of scope. Vague answers get pushed back on until they can be tested. It then drafts a bug, feature, task, or spike ticket with a problem statement, pass/fail acceptance criteria, and technical context that points to where the work starts and what must not change, without dictating the implementation. Before showing you the draft, it rereads it as an assignee who wasn't in the conversation and fixes whatever they would have to ask.

Nothing is saved to the repo. If `gh` or a tracker tool is available, Claude offers to file the ticket, and only does so when you say yes. Otherwise it hands back the title and body as markdown ready to paste. It uses your team's issue template if the repo has one.

### Plan a feature

```
/plan-feature add OAuth2 login with Google and GitHub providers
```

Claude checks each decision the feature needs and asks about the ones that would change what gets built and that neither your request nor the code settles, in short rounds with a suggested answer for each. A vague request gets questions; a fully specified one gets none, with a line saying which decisions your request settled. It then researches your codebase, produces a plan with a **Done When** checklist (the pass/fail checks that mean the feature is finished) and a task breakdown, and saves it to `.claude/plans/oauth-login.md`. Open that file, review the tasks, reorder them, edit descriptions, or remove things you don't want.

### Build from a plan

```
/build-feature oauth-login
```

Claude reads the plan and builds every phase in one go, without stopping between phases. For each task it runs the check, saves the command and output under the task as evidence, and moves on. If a check can't run, the task stays unchecked and is reported as unverified. Claude stops early only when something blocks the build: a fix fails twice, a task that later work depends on can't be verified, or the plan turns out to be wrong.

Once all phases are done, Claude runs the plan's Done When checks, then `/review-feature` reviews the build in a fresh context. Critical and Warning findings in the new code get fixed, and anything outside the plan's scope comes back for you to decide on. Claude then updates the plan file with a completion summary.

For a run nobody is watching, start it in auto mode. The Done When list also works as a [`/goal`](https://code.claude.com/docs/en/goal) condition.

If you run `/build-feature` without a name, it'll list available plans and let you pick one. If the plan has some tasks already marked `[x]`, it resumes from the first incomplete task.

### Revise a plan

```
/revise-plan oauth-login swap Google provider for Apple
```

Claude loads the existing plan, classifies the change (add / remove / modify / reorder / swap approach), and edits **surgically** — the plan always reads as current desired state, not a changelog. Git carries the history.

Unbuilt (`[ ]`) tasks are revised freely — bodies get rewritten, tasks get removed or inserted. No special handling, since no code exists yet.

Built (`[x]`) tasks need more care because the `[x]` is a claim that code exists matching the spec:

- **Modified task** — the task body gets rewritten in place. If it was `[x]`, Claude clears it to `[ ]` (the built code no longer matches the spec) and notes which existing files the rebuild should touch.
- **Removed task** — the task is deleted from the plan. If the built code should also go, Claude adds a **cleanup task** with specific files/functions to delete and a concrete Verify step (e.g., `grep -r 'OldHandler' returns no matches`).
- **Added task** — inserted unchecked, ready for the next build.

Then:

```
/build-feature oauth-login
```

Runs the unchecked tasks — including any cleanup tasks — so plan and code converge. If Claude is unsure whether built code should be removed or kept (e.g., a shared utility), it asks before adding the cleanup task.

### Open a pull request

```
/open-pr oauth-login
```

Claude re-runs the tests, lint, type-check, and the plan's Done When checks, then scans the diff for secrets, debug leftovers, and files that look unintended. It doesn't fix anything here: if something fails, it asks whether to fix it first, open a draft, or go ahead. It then writes the title and description from the plan: what changed and why, how it was tested, and a "Needs a closer look" section for deviations and anything unverified. With no plan, it works from the commits. It follows your repo's PR template and title conventions if it finds them, and adds no "Generated with Claude Code" line.

You see the full draft first. Claude pushes the branch and runs `gh pr create` only when you say so, and never force-pushes. If the branch already has an open PR, it offers to update that description instead.

### Address review comments

```
/address-review 123
```

With no argument, it picks up the PR for the current branch. Claude reads every open review thread and checks each comment against the code before acting on it. It then sorts them: fix, push back (with evidence), answer, your call, or already done. You confirm that triage and decide the judgment calls before any code changes.

Fixes become local commits, with a test when a comment points at a bug. Each thread gets a drafted reply: a short "fixed in `abc1234`", or a pushback that cites the line and the test. Claude never rewrites history already on the remote. It pushes and posts the replies only when you say so, and leaves threads for the reviewer to resolve.

### Review changes

```
/review-feature
```

Reviews the pending diff against its plan if one exists, or against the change's stated intent otherwise. It runs in a fresh context, so it never sees the reasoning that produced the code, and it can't edit files. With a plan, it checks every task, re-runs the build's evidence, and runs the Done When checks. It reports only Critical and Warning findings that affect correctness or the plan, and each one says how far it was proven, from a cited line up to a reproduction in the running app. Style and cleanup are left to the built-in `/simplify`, and a broad bug sweep to `/code-review`.

### Review a pull request

```
/review-pr 123
```

Takes a PR number, a URL, or a branch; with no argument it reviews the PR for the current branch. Claude reads the description, linked issue, CI status, and earlier review comments, then checks the PR's code out into a separate worktree so your working tree isn't touched. It reviews design first and lines second, and tries to disprove each finding before keeping it: confirmed problems become issues, unconfirmed ones become questions, and anything CI or another reviewer already covers is dropped.

The draft has a recommended verdict, a short summary that says what was and wasn't checked, and inline comments labeled blocking or non-blocking, in [Conventional Comments](https://conventionalcomments.org/) style unless your team has its own. Nothing is posted until you say so. When you do, it goes up as a single review under your account, with the verdict you chose. It needs the `gh` CLI logged in to read and post on GitHub; without it, Claude reviews the branch diff and hands back markdown.

Use `/review-feature` for your own uncommitted changes or a build from a plan, and `/review-pr` for a pull request.

### Debug an issue

```
/debug-issue the /api/users endpoint returns 500 when the body is empty
```

Runs a structured debugging workflow: reproduce → isolate → fix → verify. If a bug won't reproduce at first, Claude tries recreating the trigger or adding temporary logging before asking you.

### Correct a recurring mistake

```
/correct plans keep getting saved to the wrong folder
```

Finds where the mistake has happened before, then fixes it at the strongest level that works: architecture, types, a lint or CI check, a test, a Claude Code hook or permission rule, and docs only as a last resort. Each new check is proven against a real past mistake. The rule is recorded in `.claude/enforced-rules.md`, and only rules nothing can enforce go into `CLAUDE.md`, which loads into every session. With no argument, it searches for repeats and asks which to fix before changing anything.

### Learn a codebase

```
/learn-codebase how billing works
```

Explores the repo, traces the key execution flows, and writes up how it all works in `.claude/learn/billing.md` without stopping to ask questions. The argument is optional; without one it covers the whole project in `overview.md`. Each record notes the commit it was written against, and running the skill again updates the existing record. It doesn't change any code.

## The plans directory

Plans live in `.claude/plans/` in your project root:

```
.claude/
└── plans/
    ├── oauth-login.md          ← ready to build
    ├── user-notifications.md   ← completed (has ## Completed section)
    └── db-migration.md         ← ready to build
```

You can `.gitignore` this directory or commit it — up to you. Committing plans gives you a nice history of what was planned vs. what was built.

## Customization

All skills are plain markdown in `skills/` — edit them to fit your workflow (test with `claude --plugin-dir .`, then push and update):

- **Add project-specific conventions** (e.g., "always create a migration file", "run `pnpm test` after each task")
- **Change the plan format** to match your issue tracker
- **Adjust task granularity** (default targets 3-8 tasks per feature)
- **Add a verification step** to the plan skill (e.g., "always include a task for writing tests")

## Tips

- Works best when your project has a `CLAUDE.md` or `README.md` — the research phase reads these.
- For large features, review the plan carefully and split into multiple smaller plans if needed.
- You can edit the saved plan file manually before running `/build-feature` — Claude follows whatever's in the file.
- Plans from previous sessions still work — the build skill reads the file fresh each time.
- If scope shifts mid-build, use `/revise-plan` rather than editing by hand — it protects completed-task markers so resume still works.

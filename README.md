# Claude Code Skills

A set of skills for structured development with Claude Code: plan before you build, revise plans as scope shifts, and review, debug, or refactor existing code with repeatable workflows.

The core loop is plan → build, with `/revise-plan` for mid-flight changes. The remaining skills cover ticket writing, review, pull request review, debugging, refactoring, learning a new codebase, and stopping recurring mistakes — each is a standalone workflow, usable on its own.

## How the plan → build loop works

```
/plan-feature add user notifications with email and in-app support
    │
    ├── Researches your codebase (structure, patterns, conventions)
    ├── Works out how to run and check the app
    ├── Creates a detailed task breakdown
    └── Saves to .claude/plans/user-notifications.md

    ⏸️  You review the plan. Edit it. Sleep on it. Share it.
    ⏸️  Scope changed? → /revise-plan user-notifications <what to change>

/build-feature user-notifications
    │
    ├── Reads the saved plan
    ├── Executes each task in order, skipping any already marked [x]
    ├── Verifies each task for real and saves the evidence in the plan
    ├── Runs an independent /review-feature and fixes Critical and Warning findings
    └── Updates the plan file with completion details
```

## Skills

| Skill | Trigger | What it does |
|-------|---------|--------------|
| `/write-ticket` | "write a ticket", "file a bug", "create an issue" | Research codebase → interview you on the gaps → draft a ticket → file it in your tracker or hand back markdown |
| `/plan-feature` | "plan", "scope", "break down", "design" | Research codebase → create task plan → save to `.claude/plans/` |
| `/build-feature` | Manual: type `/build-feature` | Load plan from `.claude/plans/` → execute tasks → verify → independent review → summarize |
| `/revise-plan` | "change the plan", "add a task to", "swap X for Y" | Edit an existing plan; preserves `[x]` markers so in-flight builds resume cleanly |
| `/review-feature` | "review", "check my code", "audit" | Review changes for bugs, edge cases, security, consistency, and maintainability |
| `/review-pr` | "review PR 123", "look at this PR", a pasted PR link | Read the intent → check design → verify each finding → draft a verdict and labeled inline comments → post to GitHub only when you say so |
| `/debug-issue` | "debug", "fix this bug", "why is this broken" | Structured debugging: reproduce → isolate → fix → verify |
| `/refactor` | "refactor", "restructure", "split this file" | Plan and execute a refactor with a test baseline |
| `/learn-codebase` | "walk me through", "onboard me", "explain this repo" | Deep dive: explore the repo → trace real flows → save a record to `.claude/learn/` |
| `/correct` | Manual: type `/correct` | Find repeated mistakes → fix each at the strongest level → prove the check fails on a past mistake |

`/build-feature` and `/correct` change code across a project, so they only run when you type them. The others can also start from what you say.

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

Claude researches your codebase, produces a task plan, and saves it to `.claude/plans/oauth-login.md`. Open that file, review the tasks, reorder them, edit descriptions, or remove things you don't want.

### Build from a plan

```
/build-feature oauth-login
```

Claude reads the plan, executes each task, runs its check, saves the command and output under the task as evidence, and moves on. If a check can't run, the task stays unchecked and is reported as unverified. Once all phases are done, a fresh subagent reviews the build: Critical findings get fixed, and Warnings and Suggestions come back for you to decide on. Claude then updates the plan file with a completion summary.

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

### Review changes

```
/review-feature
```

Reviews the pending diff against a plan if one exists, or reviews standalone changes otherwise. Each Critical or Warning finding says how far it was proven, from a cited line up to a reproduction in the running app.

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

### Refactor

```
/refactor extract the notification dispatch logic from UserService
```

Establishes a test baseline, plans the refactor, and executes it without changing behavior.

### Correct a recurring mistake

```
/correct plans keep getting saved to the wrong folder
```

Finds where the mistake has happened before, then fixes it at the strongest level that works: architecture, types, a lint or CI check, a test, and docs only as a last resort. Each new check is proven against a real past mistake, and the rule goes in a table in `CLAUDE.md`. With no argument, it searches for repeats and asks which to fix before changing anything.

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

---
name: learn-codebase
description: "Deep dive into an unfamiliar codebase and write up how it works as a markdown record, saved to .claude/learn/. Traces real execution paths through the code rather than summarizing folders. Use when the user says 'learn this codebase', 'walk me through', 'onboard me', 'explain this repo', 'how does this project work', 'help me understand this code', 'I'm new to this codebase', 'give me a tour', 'document how this works', or wants to get up to speed on a project or subsystem, even if they don't say 'learn'. Runs without asking questions and does not change any code."
argument-hint: "[area or question to focus on]"
context: fork
background: false
---

# Learn Codebase — Explore → Trace → Write

You are studying a codebase and writing up how it works, so the user can read one document and reason about the code on their own. Run start to finish without stopping to ask questions; if something is ambiguous, pick the sensible reading and state it in the record. The document is the deliverable.

The only file you create or change is the learning record in `.claude/learn/`. Don't edit code, and don't commit anything. Running the tests or the app is fine if it confirms something and has no side effects; skip it if you're unsure.

## Step 1: Set the Scope

If the user named an area or question, focus on that and name the record after it (e.g., `billing.md`). Otherwise cover the whole project and use `overview.md`.

If that file already exists in `.claude/learn/`, read it, then update it to match the current code rather than creating a second record. Check its claims again; don't carry them over on trust.

## Step 2: Explore

Research before writing, so the record follows how the system really works and not just how the folders are laid out. Use subagents or direct exploration.

- **Docs:** README, CLAUDE.md, architecture docs, and contributing guides. Treat these as claims to check against the code, since they drift.
- **Stack and shape:** Languages, frameworks, key dependencies, and top-level directories. For a monorepo, how the packages relate.
- **Entry points:** Where execution starts, such as servers, CLI commands, workers, cron jobs, and UI routes.
- **Core domain:** The main data models and the schema, plus the words the code uses for them.
- **Boundaries:** Auth, databases, queues, and external APIs.
- **How to run it:** Install, dev, test, and build commands, plus config and environment variables.
- **Tests:** The framework, where tests live, and which ones best show intended behavior.
- **History:** Use `git log` to find the most-changed files and recent areas of work. Churn often reveals what matters and what's fragile.

Read the key files rather than inferring from names. Everything in the record should come from code you actually read.

## Step 3: Trace the Key Flows

Pick the 3-5 flows that matter most, or that best answer the user's focus. Choose real flows ("what happens when a user submits an order"), not directories. Following a flow shows how the parts cooperate; listing folders doesn't.

For each flow, follow the actual execution path from trigger to outcome. Note every stop with its `file:line`, how control or data reaches the next stop, the error paths, and how it's tested. When the reason for a design isn't obvious, check `git log` or `git blame`.

## Step 4: Write the Record

Save to `.claude/learn/<name>.md`, creating the directory if needed. Use this structure, leaving out any section that doesn't apply:

```markdown
# [Project or area]

> Written against commit `<short hash>` on <YYYY-MM-DD>. Scope: [whole project, or the focus area].

## What it is
[1-2 sentences — what this system does and for whom]

## Main pieces
[Each major component, its job, and where it lives]

## How they connect
[A small ASCII diagram of the main data or request flow]

## Key terms
[Domain words the code uses, with plain meanings]

## Running it
[Install, dev, test, and build commands; required config]

## Flows

### [Flow name]
**Trigger:** [what starts it] **Outcome:** [what it produces]

1. `path/to/file.ts:42` — [what happens, and how it reaches the next stop]
2. ...

**Why it's built this way:** [patterns in use and the reasoning]
**Watch out for:** [hidden side effects, implicit ordering, framework magic, error paths]
**Tests:** [the test to read, or the gap]

## Conventions
[The unwritten rules you saw in the code]

## Gotchas
[The main fragile areas and traps]

## Not covered
[What you skipped or couldn't verify]

## Good first steps
[2-3 concrete things to read, run, or change next]
```

Use short snippets only where the code itself is the point. The `file:line` references are what let a reader check the record later, so keep them accurate.

## Step 5: Report

Tell the user where the record is and give the short version:

> **Saved:** `.claude/learn/<name>.md`
>
> **The mental model:** [3-5 sentences tying it together]
>
> **Worth a look:** [the biggest gotcha, plus any bug or stale doc you found]

## Principles

- **Evidence over plausibility.** Every claim about the code should trace to something you read. If you couldn't verify something, put it under "Not covered". A confident wrong explanation does more harm than a gap, because the reader will build on it.
- **Separate fact from inference.** "This retries three times" is a fact. "This was probably done for rate limits" is a guess, so label it.
- **Prefer what's hard to rediscover.** Anyone can list a directory. Spend the words on traced behavior, reasoning, conventions, and traps.

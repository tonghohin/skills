---
name: plan-feature
description: "Research the codebase and create a detailed implementation plan for a new feature or project. Use when the user says 'plan', 'scope', 'break down', 'design', 'outline', 'how would I build', 'what would it take to', or wants to understand the approach before building. Also use when starting any new feature — planning should come before building. Produces a research summary and task breakdown, saves it to .claude/plans/ so /build-feature can pick it up later."
argument-hint: "<feature description>"
---

# Plan Feature — Research → Plan → Save

You are creating an implementation plan. You will NOT write any implementation code. Your job is to research, think, and produce a plan that `/build-feature` can execute later.

## Step 1: Understand the Request

Restate what the user wants in your own words in 2–3 sentences.

Then **always** run a structured clarification interview before doing any research or writing any plan. Your goal is to surface every assumption and decision point now, so you don't have to revise the plan later.

Ask about as many of these dimensions as apply to the feature — group related questions together, and skip any that are clearly irrelevant. Use the interview for decisions only the user can make. If the answer is a fact you can get by reading or running the code, find it out during research instead of asking.

- **Scope & success criteria:** What does "done" look like? What's explicitly out of scope?
- **User flow:** Walk through the happy path step by step. Who does what, in what order?
- **Edge cases & errors:** What happens when input is invalid / missing / malformed? What if a network call or DB write fails? Are there retry or rollback needs?
- **Data:** What data needs to be stored, read, or transformed? Any schema changes?
- **Integrations:** Does this touch auth, payments, emails, third-party APIs, background jobs, or any existing feature area?
- **UI/UX:** Any specific layout, loading states, empty states, or responsive behavior?
- **Permissions & access control:** Who can access this? Any role or ownership checks?
- **Performance & scale:** Any volume, latency, or caching concerns?
- **Testing expectations:** Are there specific scenarios or edge cases you want covered by tests?
- **Anything you're unsure about:** Is there any part of the feature you haven't fully decided yet?

Do NOT proceed to Step 2 until the user has answered. Once you have answers, briefly confirm your understanding in a short bullet-list summary, then ask: "Does this capture everything? Anything to add or change?" Only move forward after they confirm.

## Step 2: Research

Use subagents or direct exploration to understand the codebase. Be thorough — missing a relevant file now means a plan revision later.

- What is the project structure? (frameworks, languages, key directories)
- Are there existing patterns for similar features? (auth, routing, state, API calls, DB access, tests)
- What dependencies are relevant?
- Are there any CLAUDE.md, README, or architecture docs?
- What coding conventions does the project enforce or follow? (lint, format, and type-check config and the commands that run them; naming, error handling, and file layout in the code near the feature)
- What testing patterns exist? (framework, conventions, coverage)
- Check recent git history — are there in-flight changes or PRs touching the same areas? The plan should account for concurrent work.
- Are there any existing types, interfaces, or schemas the new feature must conform to?
- Are there any shared utilities, hooks, middleware, or services this feature should reuse?

If the feature involves a library, API, or technique you're unsure about, search docs or the web. Don't guess.

Then work out how an agent can check this work for real, because every Verify step depends on it:

- **Run:** How does the app start locally, and how do you tell it's ready? What ports, env vars, or seed data does it need?
- **Drive:** How can an agent use the feature the way a user would? Look for existing options first: test suites, Playwright specs, CLI commands, HTTP endpoints. If a `run` skill or a project verification skill is available, use it.
- **Observe:** What shows it worked? Command output, response bodies, logs, stored data, screenshots.

Try what you find, as long as it has no side effects, so the plan rests on commands you've seen work. Don't install anything or change data to do this; record what you couldn't try and why.

## Step 2.5: Think Before Writing

Before drafting the plan, reason through these questions internally:

- Are there any dependencies between tasks that could trip up the builder if ordered wrong?
- Are there any implicit requirements the user didn't mention but that must be true for the feature to work? (e.g., a new API route needs auth middleware, a DB query needs a migration first)
- Are there any edge cases from the clarification answers that don't yet have a task assigned to them?
- Is there anything that could break existing functionality?
- Are there any tasks that are larger than they appear and should be split?

If this reasoning surfaces anything significant, add it to the plan's Notes section and flag it clearly.

## Step 3: Create the Plan

Write a plan document with this structure:

```markdown
# Plan: [Feature Name]

> [One-line description of what this feature does]

## Research Summary

- **Stack:** [e.g., Next.js 14 App Router, TypeScript, Prisma, PostgreSQL]
- **Relevant patterns:** [e.g., existing auth uses NextAuth in src/lib/auth.ts]
- **Conventions:** [rules the builder must follow, e.g., named exports only, errors returned as Result, `npm run lint` and `npm run typecheck` must pass]
- **Key files:** [list the 3-8 most relevant existing files/dirs]
- **New dependencies:** [any new packages needed, or "none"]
- **Risks/Considerations:** [edge cases, breaking changes, migrations]

## Data Shape

[The core types, schemas, and state this feature adds or changes, and who owns each. Tasks build on these. Write "no new data" if nothing changes.]

## How to Verify

- **Run:** [command to start the app, and how to tell it's ready]
- **Drive:** [how to use it like a user: test command, CLI, HTTP call, browser tool]
- **Observe:** [where the proof comes from: output, logs, stored data, screenshots]
- **Clean up:** [how to stop what was started]
- **Tried during planning:** [what you ran and what happened, plus anything you couldn't try and why]

## Tasks

### Phase 1: [Phase title, e.g., Data Layer]

#### 1.1. [Short title]
- **What:** [1-2 sentence description]
- **Files:** [files to create/modify]
- **Verify:** [how to confirm it works — run test, check output, etc.]

#### 1.2. [Short title]
- **What:** ...
- **Files:** ...
- **Verify:** ...

### Phase 2: [Phase title, e.g., API]

#### 2.1. [Short title]
- **What:** ...
- **Files:** ...
- **Verify:** ...

[...more phases/tasks as needed. Small features may use a single phase.]

## Notes

[Anything the builder should know — open questions, alternative approaches considered, follow-up work]
```

Guidelines for tasks:
- **3-8 tasks per phase.** A small feature may have a single phase. For larger features, group tasks into phases (e.g., "Phase 1: Data Layer", "Phase 2: API", "Phase 3: UI"). Each phase should be a cohesive chunk of work that can be verified as a unit.
- Each task should touch **1-3 files** and be **independently verifiable**.
- Order by dependency — foundational phases first, then tasks within each phase.
- The verify step should be concrete: a command to run, a behavior to check, not "make sure it works." Name the result you'd actually see, such as the output, response, or screen state. When a task has a user-facing surface, passing tests alone aren't enough: check the real thing. Build each verify step on the commands in How to Verify, so an agent can run it without help.
- **Each phase should include a testing task.** Prefer unit tests over integration/e2e tests — they're faster and cheaper. Only add integration or e2e tests when the feature genuinely requires testing across boundaries (e.g., API + DB, multi-step UI flows). Don't write tests for trivial glue code or simple pass-through logic.

## Step 4: Save the Plan

Plans live in the **repository**, not the user's home directory. The plans directory is `.claude/plans/` at the **repo root** — resolve it with `git rev-parse --show-toplevel` (or use the project working directory if not a git repo). Never write to `~/.claude/` — that is the user's global Claude config folder, and plans saved there are invisible to `/build-feature` running in the repo. Always use the absolute path `<repo-root>/.claude/plans/` when creating and writing files.

1. Create the directory `<repo-root>/.claude/plans/` if it doesn't exist.
2. Save the plan as `<repo-root>/.claude/plans/[feature-name].md` using a kebab-case slug derived from the feature name (e.g., `user-notifications.md`, `oauth-login.md`). If a plan with the same name already exists, ask the user whether to overwrite it or use a different name.
3. Tell the user:

> **Plan saved to `.claude/plans/[feature-name].md`**
>
> Review it, edit anything you'd like to change, then run:
> ```
> /build-feature [feature-name]
> ```
> to start building.

## Principles

- **No implementation code.** You're planning, not building. Don't create source files, install packages, or run migrations.
- **Be specific.** "Update the API route" is a bad task. "Add a POST handler in `src/app/api/notifications/route.ts` that accepts `{userId, message, channel}` and writes to the notifications table" is a good task.
- **Match existing patterns.** The plan should follow the conventions already in the codebase, not introduce new ones.
- **Surface decisions, don't bury them.** If there's a meaningful choice (e.g., polling vs websockets), call it out in Notes so the user can weigh in before building.
- **Code quality defaults.** Prefer specific types over `any` or type casts; if a task introduces a new type/schema, call it out. For runtime validation at boundaries (API input, parsed config, external data), reach for the idiomatic schema library in the stack — **Zod** for TypeScript, **Pydantic** for Python.
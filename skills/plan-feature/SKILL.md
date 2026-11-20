---
name: plan-feature
description: "Research the codebase and create a detailed implementation plan for a new feature, project, or larger refactor. Use when the user says 'plan', 'scope', 'break down', 'design', 'outline', 'how would I build', 'what would it take to', 'plan a refactor', or wants to understand the approach before building. Also use when starting any new feature — planning should come before building. Produces a research summary, a Done When checklist, and a task breakdown, saved to .claude/plans/ so /build-feature can pick it up later."
argument-hint: "<feature description>"
---

# Plan Feature — Understand → Research → Plan → Save

You are creating an implementation plan. You don't write implementation code. Your job is to research, think, and produce a plan that `/build-feature` can execute later.

## Step 1: Understand the Request

Restate what the user wants in your own words in 2–3 sentences.

Then take a quick look at the code the feature touches, enough to propose answers rather than ask open questions. Full research comes in Step 2.

Interview the user only for what they alone can decide: intent, scope, priorities, and trade-offs. If the answer is a fact you can get by reading or running the code, find it out during research instead of asking.

- **Decide whether to ask by checking each decision, not by how the request reads overall.** List the decisions this feature needs, using the list below. For each one, ask two things: would a different answer change what gets built, and does the request or the code already settle it? Ask about every decision that would change the build and isn't settled. Don't ask about the rest.
  - A request that sounds simple can still leave a decision open: "add due-date reminders" doesn't say how a reminder reaches the user.
  - A short request can also settle everything: "add a `--version` flag that prints the version from `pyproject.toml`" has nothing to ask.
  - When no decision is open, don't ask anything. Say in one line which decisions the request settled.
- **Ask in short rounds** of 2–4 questions, most important first. Use the question tool if one is available. Offer your best guess with each question, drawn from what you saw in the code, so the user can confirm instead of composing an answer.
- **Don't accept an answer you couldn't test.** "Fast" needs a number. "Handle errors" needs to say which errors and what the user sees. Ask again, with an example of a testable answer.
- **When the user doesn't know,** don't push them to invent an answer. Record it as an open question in the plan's Notes.
- **Stop** when the remaining gaps wouldn't change the plan, or when the user says that's enough.

What to press on, skipping anything already answered or irrelevant:

- **Done and out of scope:** What will be observably true when it's finished? What would a reader assume is included that isn't?
- **Failure paths:** Invalid input, an empty state, a failed call or write. What should the user see?
- **Data:** What gets stored or changed? Any schema changes?
- **Access:** Who can use this? Any role or ownership checks?
- **When the feature has them:** UI states (loading, empty, error), integrations it touches (auth, payments, email, third-party APIs, background jobs), volume or latency limits, and scenarios the user wants tested.
- **Undecided parts:** Anything the user hasn't made up their mind about yet.

After the last round, move on to research. The plan itself is where the user confirms your understanding, so there's no separate confirmation round.

## Step 2: Research

Research the codebase thoroughly — missing a relevant file now means a plan revision later. Beyond the obvious, make sure you find:

- **The nearest existing feature like this one.** The plan should follow its patterns, not introduce new ones.
- **What to reuse:** shared utilities, middleware, services, and the types or schemas the feature must conform to.
- **The enforced conventions:** the lint, format, type-check, and test commands, plus how nearby code names things and handles errors.
- **Concurrent work:** recent commits and open PRs touching the same areas.

If the feature involves a library, API, or technique you're unsure about, search docs or the web. Don't guess.

Then work out how an agent can check this work for real, because every Verify step and the Done When list depend on it:

- **Run:** How does the app start locally, and how do you tell it's ready? What ports, env vars, or seed data does it need?
- **Drive:** How can an agent use the feature the way a user would? Look for existing options first: test suites, Playwright specs, CLI commands, HTTP endpoints. If `/verify`, `/run`, or a project run skill is available, use it to find and try the launch steps.
- **Observe:** What shows it worked? Command output, response bodies, logs, stored data, screenshots.

Try what you find, as long as it has no side effects, so the plan rests on commands you've seen work. Don't install anything or change data to do this; record what you couldn't try and why. If the app is hard to launch and the project has no run skill, suggest `/run-skill-generator` in the plan's Notes so `/verify` can drive the app later.

## Step 3: Create the Plan

Before writing, check that every edge case and failure path from Step 1 has a task, and that implicit requirements have one too (a new route needs auth, a new query needs its migration first).

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

## Done When

- [ ] [An observable result and the command that shows it, e.g. `POST /api/notifications` with a valid body returns 201 and the row appears in `notifications`]
- [ ] [At least one failure path, e.g. a body without `userId` returns 422 and nothing is stored]
- [ ] [The project's checks, e.g. `npm test`, `npm run lint`, and `npm run typecheck` pass]

## Data Shape

[The core types, schemas, and state this feature adds or changes, and who owns each. Tasks build on these. Write "no new data" if nothing changes.]

## How to Verify

- **Run:** [command to start the app, and how to tell it's ready]
- **Drive:** [how to use it like a user: test command, CLI, HTTP call, browser tool, `/verify`]
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

Guidelines for Done When:
- **It defines finished for the whole feature.** `/build-feature` runs every check before it reports done, `/review-feature` checks against it, and the list works as a `/goal` condition for an unattended run.
- **3–8 checks.** Each one is pass or fail, names the result you'd see, and can be run by an agent using the commands in How to Verify.
- **Results, not activities.** "The settings page saves the new email and shows it after reload" is a check. "Implement the settings form" is a task.
- **Cover failure.** Include at least one failure path; a list with only the happy path gets only the happy path built.

Guidelines for tasks:
- **3-8 tasks per phase.** A small feature may have a single phase. For larger features, group tasks into phases (e.g., "Phase 1: Data Layer", "Phase 2: API", "Phase 3: UI"). Each phase should be a cohesive chunk of work that can be verified as a unit.
- Each task should touch **1-3 files** and be **independently verifiable**.
- Order by dependency — foundational phases first, then tasks within each phase.
- The verify step should be concrete: a command to run, a behavior to check, not "make sure it works." Name the result you'd actually see, such as the output, response, or screen state. When a task has a user-facing surface, passing tests alone aren't enough: check the real thing. Build each verify step on the commands in How to Verify, so an agent can run it without help.
- **Each phase should include a testing task.** Prefer unit tests over integration/e2e tests — they're faster and cheaper. Only add integration or e2e tests when the feature genuinely requires testing across boundaries (e.g., API + DB, multi-step UI flows). Don't write tests for trivial glue code or simple pass-through logic.

For a refactor (restructuring code without changing behavior):
- The first task records the test baseline. If the code has no tests, the first phase writes tests for its current behavior; type-checking and linting don't count as a baseline.
- Order removal before reshaping: delete dead code, one-caller wrappers, and redundant checks first, then restructure what's left. Each reshape should remove branches or invalid states, not just add indirection.
- Done When includes "every test that passed at the baseline still passes" and a check that the result is easier to follow: fewer files, branches, or concepts to hold in your head. Behavior changes go in a separate plan.

## Step 4: Save the Plan

1. Plans live in `<repo-root>/.claude/plans/` (resolve the root with `git rev-parse --show-toplevel`, or use the working directory outside git), never in `~/.claude/`, where `/build-feature` can't see them. Create the directory if it doesn't exist.
2. Save the plan as `<repo-root>/.claude/plans/[feature-name].md` using a kebab-case slug derived from the feature name (e.g., `user-notifications.md`, `oauth-login.md`). If a plan with the same name already exists, ask the user whether to overwrite it or use a different name.
3. Tell the user:

> **Plan saved to `.claude/plans/[feature-name].md`**
>
> [If you asked nothing: the decisions the request settled, in a few bullets.]
>
> Review it, edit anything you'd like to change, then run:
> ```
> /build-feature [feature-name]
> ```
> to start building.

## Principles

- **No implementation code.** Don't create source files, install packages, or run migrations.
- **Be specific.** "Update the API route" is a bad task. "Add a POST handler in `src/app/api/notifications/route.ts` that accepts `{userId, message, channel}` and writes to the notifications table" is a good task.
- **Surface decisions, don't bury them.** If there's a meaningful choice (e.g., polling vs websockets), call it out in Notes so the user can weigh in before building.
- **Code quality defaults.** Prefer specific types over `any` or type casts; if a task introduces a new type/schema, call it out. For runtime validation at boundaries (API input, parsed config, external data), reach for the idiomatic schema library in the stack — **Zod** for TypeScript, **Pydantic** for Python.

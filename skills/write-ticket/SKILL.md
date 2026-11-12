---
name: write-ticket
description: "Turn a rough idea, bug, or request into a ticket someone else can pick up cold. Researches the codebase, then interviews the user to close the gaps before drafting. Use when the user says 'write a ticket', 'create an issue', 'file a bug', 'write this up', 'ticket this', 'draft a story', 'turn this into a ticket', or wants to hand work to a teammate or a coding agent, even if they don't say 'ticket'. Covers bugs, features, tasks, and spikes. Files the ticket in the tracker on request, or hands back markdown ready to paste. Not for implementation plans; use /plan-feature for those."
argument-hint: "<what the ticket is about>"
---

# Write Ticket — Research → Grill → Draft → Cold Read

You are writing a ticket. You will NOT write implementation code or fix the problem. Your job is to produce a ticket that someone who was not in this conversation can pick up and finish without asking the author anything.

Most bad tickets fail in one of two ways. Either they are a title and a hope, so the assignee has to rebuild the context, or they are a set of step-by-step instructions written without the code open, so the assignee follows them into a wall. This skill avoids both: gather the context, press the user on what is vague, and write down the problem, the finish line, and the constraints, leaving the route to the person doing the work.

## Step 1: Intake

Restate the request in 1–2 sentences, then settle three things. Infer them from the request and the repo where you can, and ask only for what you can't infer.

**Type.** It decides which sections the ticket has.

- **Bug:** shipped behavior is wrong.
- **Feature:** behavior a user will notice is added or changed.
- **Task:** internal work with no user-visible change, such as a migration, upgrade, or cleanup.
- **Spike:** the answer isn't known yet, and the output is a decision or a finding.

If nobody can say what "done" looks like because a question is still open, it's a spike. Write the spike and leave the build ticket until the spike is answered.

**Reader.** Who picks this up sets how much goes in (see Technical Depth).

- A teammate who knows the codebase.
- Someone new to this area, or another team.
- A coding agent, which has only the ticket and the repo.
- The user themselves, later.

**House style.** Check for `.github/ISSUE_TEMPLATE/`, a ticket template in the docs, or conventions in `CLAUDE.md`. If the `gh` CLI or a tracker tool is available, read a few recent tickets. Where the team already has a format, use its sections and wording, and apply the rules in this skill to what goes inside them.

## Step 2: Research

Read before you ask. Every fact you find yourself is a question the user doesn't have to answer, and the questions you do ask get sharper.

- Find where the behavior lives: the entry point, the main files, and the tests that cover it.
- Note what the current behavior is, from the code, not from the request.
- Look for an existing pattern the work should follow, and for anything nearby it could break.
- Check for duplicates or related work: open tickets, recent commits, and plans in `.claude/plans/`.
- For a bug, try to reproduce it if that is cheap and has no side effects. Record the exact steps and output. If you can't reproduce it, say so in the ticket. Don't chase the root cause; a ticket needs the symptom pinned down, and a cause only if you happened to find one.

Keep the research in proportion. A ticket needs the starting point and the hazards, not a full design.

## Step 3: Grill

Interview the user until you could fill every section without guessing. Ask only what the user alone can answer: intent, priority, and decisions. Anything the code can tell you belongs in Step 2.

How to run it:

- Ask in short rounds of 2–4 questions, most important first. Use the question tool if one is available.
- Offer your best guess with each question, drawn from the research, so the user can confirm instead of composing an answer.
- Don't accept an answer you couldn't test. "Fast" needs a number. "Handle errors" needs to say which errors and what the user sees. "Like the other page" needs to say which parts. Ask again, with an example of what a testable answer looks like.
- When an answer contradicts the code or an earlier answer, say so and ask which is right.
- When the user doesn't know, don't push them to invent an answer. Record it under Open Questions, or turn the ticket into a spike if the unknown blocks everything.

What to press on, skipping anything already answered or irrelevant:

- **Why now:** What is the evidence this matters, such as a user report, a number, or an incident? What happens if it's never done? Ask for the original wording of any user feedback so the ticket can quote it.
- **Who is affected:** Which users, how many, and how often?
- **Done:** What will be observably true when this is finished? How would someone check it?
- **Failure paths:** What should happen on invalid input, an empty state, a missing permission, or a failed call?
- **Out of scope:** What will a reader assume is included that isn't?
- **Constraints:** What has already been decided or must not change, such as a public API, a deadline, a library choice, or compatibility?
- **Dependencies:** What blocks this, and what does it block?
- **Bug only:** Exact steps, expected against actual, environment, how often it happens, when it started, and any workaround.

Stop when the remaining gaps wouldn't change what the assignee builds, or when the user says that's enough. Then confirm your understanding in a few bullets and wait for a yes before drafting.

**Size check.** A ticket should be one change that can be reviewed in one sitting. Propose a split when the criteria run past about eight, when parts could ship on their own, or when the work spans unrelated areas. List the tickets you'd write and let the user choose. If the work is big enough to need phases, suggest `/plan-feature` instead.

## Step 4: Draft

### Title

The title is read in a list, so it has to make sense with nothing around it.

- Bug: the symptom and where it happens. "Checkout returns 500 when the cart is empty", not "Checkout bug".
- Everything else: a verb and an object. "Add CSV export to the invoices page", not "CSV export".
- Leave out ticket-type prefixes, sprint numbers, and "we should".

### Body

Use the sections for the ticket's type. Drop a section that has nothing to say instead of filling it with "N/A", except Out of Scope and Acceptance Criteria, which are always present.

```markdown
# [Title]

**Type:** Bug | Feature | Task | Spike

## Problem
[What is wrong or missing, stated as something observable, and who it affects.
Then why it matters now. Quote user feedback word for word and link its source.
Describe the problem, not the fix.]

## Acceptance Criteria
- [ ] [An observable result that anyone could check and mark pass or fail]
- [ ] [At least one failure path: what happens when it goes wrong]

## Out of Scope
- [What a reader would assume is included but isn't, and where it's tracked if anywhere]

## Technical Context
- **Where:** [entry points and key files, as links or paths]
- **Constraints:** [decisions already made and things that must not change, each with its reason]
- **Follow:** [the existing pattern or code to reuse]
- **Watch out:** [traps found in research: hidden callers, migrations, flags, flaky tests]
- **Suggested approach:** [optional, labeled as a suggestion; see Technical Depth]

## How to Verify
[Commands to run or steps to take, and the result that proves it works]

## Open Questions
- [What is unresolved, and who can answer it]

## Links
[Designs, docs, related tickets, the thread where this was discussed]
```

Sections that change by type:

- **Bug:** replace Problem with the block below. Acceptance criteria are that the steps no longer reproduce the bug, plus a regression test where one is practical.

  ```markdown
  ## What Happens
  [The actual behavior, with the exact error text, a log excerpt, or a screenshot]

  ## What Should Happen
  [The expected behavior]

  ## Steps to Reproduce
  1. [Start from a known state. One action per step. The shortest path that triggers it.]

  ## Environment and Impact
  [Version or commit, platform, and any flags or data it depends on. How often it
  happens, who is affected, when it started, and any workaround. Say whether you
  reproduced it or are passing on a report.]
  ```

- **Spike:** replace Acceptance Criteria with **Question** (the one question to answer), **Time Box**, and **Output** (what gets handed back, such as a decision, a written comparison, or a prototype to throw away). Building the thing is out of scope.

### Acceptance Criteria

Each criterion describes a result, not an activity, and can be marked pass or fail by someone who didn't write it.

- Weak: "The form validates input." Strong: "Submitting an invalid IBAN returns a 422, and the error appears under the IBAN field."
- Weak: "Improve dashboard load time." Strong: "The dashboard's largest element paints in under 2 seconds on the 10k-row seed account."
- A criterion that names a function, table, or library is an implementation step. Move it to Constraints if it's a real requirement, and delete it if it isn't.
- Cover what happens on failure. A list with only the happy path gets only the happy path built.
- Use Given/When/Then only if the team already does. A plain sentence that can be checked is enough.

### Technical Depth

Write down what the assignee can't find out cheaply or must not get wrong. Leave out what they will decide better with the code open.

**Put in:**

- Where to start: entry points and files, as pointers.
- Decisions already made, with the reason. A constraint without a reason gets argued with or quietly ignored.
- Contracts that others depend on: API shapes, schemas, event names, and anything public.
- Hazards from the research that aren't visible from the starting point.
- The existing pattern to follow.

**Leave out:**

- Step-by-step instructions, and names for functions and files that don't exist yet.
- Code, beyond a short snippet that pins down a contract or an error.
- Facts the reader gets by opening the file you pointed to.
- Copies of specs or designs. Link them, because a pasted copy goes stale.

Keep constraints and suggestions apart. "Must use the existing job queue, because ops only monitors that one" is a constraint. "One way to do this is a nightly job" is a suggestion, and it goes under Suggested approach where the assignee is free to overrule it.

Then adjust for the reader and the risk:

- **Teammate who knows the area:** pointers and constraints, with no suggested approach unless the user has one.
- **Someone new to the area:** add how the pieces fit together in a few sentences, the pattern to copy, and how to run it locally.
- **Coding agent:** the ticket is its whole briefing, so be explicit about what a person would pick up from the team. Name the files likely to change, the commands that must pass, and what must not be touched. Don't lean on "as discussed" or anything that lives outside the ticket and the repo.
- **Risky or hard to undo** (data migrations, public APIs, auth, billing): more depth whoever the reader is. State the rollout and rollback expectations, and make the contract exact.

If you find yourself writing the design, stop. Either the ticket is really a plan, which belongs in `/plan-feature`, or the design isn't settled, which calls for a spike.

## Step 5: Cold Read

Before showing the draft, read it as the assignee who has seen none of this conversation. Go through it and list every question you would have to ask the author. Then fix each one from what you know, ask the user, or add it to Open Questions.

Check that:

- The title makes sense alone in a list.
- The problem is stated without reference to the fix, and says why it matters.
- Every criterion can be marked pass or fail, and at least one covers a failure path.
- Out of Scope names what a reader would otherwise assume.
- Nothing depends on this conversation: no "as discussed", no "the usual way", no undefined abbreviations.
- Every path, command, and name in the ticket exists. Check them against the repo instead of trusting your memory of the research.
- No instruction is presented as a requirement unless the user confirmed it is one.
- A bug's reproduction steps start from a known state, and the ticket says whether you ran them.
- The ticket is short enough to be read. Cut anything that restates another section.

## Step 6: Review and File

Show the draft to the user, along with anything from the cold read that still needs their answer. Revise until they approve it.

The ticket's home is the team's tracker, so don't save it to a file in the repo. Once it's approved, get it there one of two ways:

- **File it.** If the `gh` CLI or a tracker tool is available, offer to file it, and say where it will go before you do. Filing publishes the ticket and notifies people, so do it only when the user says to. Set only the labels, assignees, and priority the user gave you. The title goes in the tracker's title field, so leave the `# [Title]` line and any field the tracker holds separately, such as Type, out of the body.
- **Hand it over.** Otherwise, give the final title on its own line and the body in a single fenced markdown block, so the user can copy each into the tracker without picking it out of the conversation.

Then tell the user:

> **Filed:** [URL] *(or "Not filed. The ticket is above, ready to paste.")*
>
> **Open questions:** [count, or "none"]
>
> To plan the implementation, run:
> ```
> /plan-feature [ticket URL or title]
> ```

## Principles

- **Write for the reader who wasn't there.** The test for every sentence is whether the assignee needs it and could not get it faster elsewhere.
- **Describe the problem and the finish line, and leave the route.** The person doing the work will know more than the ticket's author by the second hour.
- **Don't fill gaps with guesses.** An unknown written down as an open question is useful. An unknown papered over with a plausible detail gets built.
- **Short is a feature.** A ticket that takes ten minutes to read gets skimmed. Link out for depth.
- **One ticket, one outcome.** If the title needs "and", it is probably two tickets.

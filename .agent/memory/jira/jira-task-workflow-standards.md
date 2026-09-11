---
type: reference
tags: [jira, process, workflow, team, naming-convention, definition-of-done]
created: 2026-08-21
agent: main
---

Team standard for defining, managing and updating Jira issues (source doc given by user, Vietnamese). Goal: consistent naming, complete info before work starts, clear owner, timely progress updates, early risk reporting.

## Hierarchy

`Epic -> Task/Story -> Sub-task`

- Epic: large feature / initiative / goal needing many tasks.
- Task/Story: one concrete deliverable with a single main owner.
- Sub-task: smaller slice of a Task/Story.

## Naming convention

Rules: short but shows the goal, prefix with module/domain, use an action verb. Banned generic titles: "Fix bug", "Update API", "Change UI", "Do payment", "Support backend".

| Type | Format | Example |
|---|---|---|
| Epic | `[Module] Business Goal / Feature` | `[Payment] Online Payment Integration` |
| Task | `[Module] Action + Object` | `[Payment] Implement payment callback API` |
| Bug | `[Bug][Module] Issue / Incorrect Behavior` | `[Bug][Payment] Transaction status is not updated after callback` |
| Sub-task | `[Role/Component] Action + Object` | `[BE] Implement callback endpoint`, `[FE] Integrate payment result screen`, `[QA] Verify successful and failed payment cases` |

## Mandatory fields

Epic: Title, Description (Background *optional*, Objective, Expected Result *optional*, Related Documents), Owner, Start Date, Due Date, Priority.

Task: Title, Description (Requirement; Acceptance Criteria *optional*), Assignee (exactly one, never work a task with no assignee), Start Date, Due Date, Priority, Epic Link / Parent, Dependency (Blocked by / Blocks / Related).

Example Acceptance Criteria: API returns response per doc; all input fields validated; error cases handled; unit tests added; QA can verify on test env.

## Splitting rule

Split a Task when: multiple people involved; mixes BE/FE work; runs many days but has independent outputs; each part needs its own progress tracking.

Anti-pattern: one giant `Implement payment feature` held by one person for weeks with no visible progress. Instead: Task `[Payment] Implement online payment` + sub-tasks `[BE] Implement payment initiation API`, `[BE] Implement payment callback`, `[FE] Implement payment checkout UI`, `[FE] Handle payment result`, `[QA] Verify payment flows`.

## Definition of Ready

Clear title; complete description; assignee set; start date; due date; priority; docs attached; member understands and can start. If the requirement is unclear, member asks Lead first - never guess an important requirement.

## Definition of Done

Requirement fully implemented; acceptance criteria met; code reviewed (if applicable); tests done; no remaining blocker; docs updated; working note updated; Jira status moved to the correct state.

## Lead / BA responsibilities

Owns defining and preparing work. May create Epics/Tasks, define requirements, write descriptions, set acceptance criteria, priority, assignee, start/due dates, dependencies, adjust scope, re-prioritize.

Must: define the task clearly before assigning (what, deadline, where the requirement lives); guarantee all mandatory fields; track progress (near-due, blocked, at-risk, dependency-stuck members); handle blockers (decide direction, coordinate resources, work with other teams, adjust scope/priority/deadline); keep Jira updated when requirements change - never leave an important requirement only in chat.

## Team member responsibilities

On start: read title + description, check acceptance criteria, check start/due dates, check dependencies, confirm the requirement is clear, then move status to In Progress immediately (never leave To Do while actually working).

During work, keep updated: status, progress, description/working note of the sub-task, done / doing / remaining parts, blockers, new issues, dependencies, anything affecting the deadline.

Working note shape:
```
Done: implement payment callback endpoint; add request validation.
Doing: handle failed transaction cases.
Remaining: add unit tests; update API document.
Issue: waiting for sandbox credential from payment provider.
```

## Due date / at-risk reporting

Member owns their due date. Never report a problem only after the due date passed. Raise risk to Lead as early as possible with: Current Status, Issue/Blocker, Impact, Proposed Action, Expected Completion.

Example: "Task ABC-123 may miss due date 25/08. Current: API implementation ~70%. Issue: blocked by API from Team X. Impact: cannot finish integration test. Proposal: Team X delivers API before 23/08 or adjust deadline."

## Six general rules

1. Jira is the single source of truth for requirement, progress, blocker, deadline, scope change.
2. The owner tracks their own task; the Lead does not chase people daily.
3. Report risk before the deadline, not on it.
4. Status must match reality: no To Do while working, no In Progress while blocked for days, no In Progress when already finished, no Done when acceptance criteria are unmet.
5. Every task needs an Assignee and a Due Date, otherwise it is not manageable.
6. Nothing important stays only in chat - Slack/Teams/meeting decisions that change requirement, scope or deadline must be written back to Jira.

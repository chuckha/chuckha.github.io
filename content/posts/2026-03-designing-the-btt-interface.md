---
title: "Designing the BTT Interface: Why a Simplified Jira for AI Agents"
date: 2026-03-22
tags: [ai, automation, tooling, design, go]
description: "The design decisions behind BTT's interface — why AI agents need tasks, epics, statuses, and state machines to do multi-step work reliably."
draft: true
---

AI agents don't need Jira. They need 10% of Jira.

[Part 1](/posts/2026-03-building-btt-part-1-the-thesis) showed what happens when you throw 23 tasks at stateless agents — locally correct output that doesn't compose into anything coherent. [Part 2](/posts/2026-03-building-btt-part-2-making-it-real) walked through BTT's implementation. This post is about the design decisions. Why does BTT look like a simplified project management tool? Why tasks, epics, statuses, state machines?

Because freeform prompting breaks the moment your work has more than one step.

## Freeform Prompting Breaks at Scale

Here's how most people use AI for complex work:

```
"Research Kubernetes networking, write a blog post about it,
make sure it matches our style guide, verify all links work,
and make it production-ready."
```

That's five distinct activities crammed into one prompt. What happens: the agent under-researches because it's eager to start writing. The draft ignores the style guide because it's buried under research context. Links go unchecked because the agent ran out of budget. There's no feedback loop — the first draft is the final draft.

This worked fine for the incremental game's individual tasks. Each task was small enough for a single agent to handle. But the *system* fell apart because there was no structure connecting the tasks — no quality gates, no feedback, no state tracking.

The missing piece wasn't a better model. It was the structure around it.

## Tasks: Why Structure the Input

A raw prompt tells you what to do. It doesn't tell the system *how* to route the work.

BTT's planner takes a plain-English request and produces structured tasks:

```go
type Task struct {
    ID          string
    EpicID      string    // parent epic, if any
    Title       string
    Description string
    Context     []string  // accumulated feedback from revisions
    Kind        WorkKind  // task, bugfix, epic, won't-do, infra
    Effort      string    // low, medium, high
    Order       int       // execution sequence
    Status      Status    // pending → planned → implementing → complete/failed
    Attempt     int       // current attempt number
    MaxAttempts int       // default 6
}
```

Every field exists to enable a routing decision:

- **Kind** determines which workflow runs. A regular task goes through the full pipeline. A bugfix skips research and goes straight to implementation. An infra task might use a restricted execution mode.
- **Effort** determines which model handles it. Low-effort tasks get Haiku. High-effort tasks get Opus. You're not paying Opus prices for a typo fix.
- **Context** accumulates feedback across retries. On attempt 3, the agent sees what attempts 1 and 2 got wrong.
- **Order** controls execution sequence within an epic.
- **Attempt** and **MaxAttempts** prevent infinite retries.

None of this is possible with a raw prompt. You can't route a string to different models based on complexity. You can't retry a string with accumulated feedback. You can't track the status of a string.

The task struct is the minimum viable description of a unit of work that an orchestrator can reason about.

## Epics: Grouping Related Work

Sometimes one request produces multiple tasks that depend on each other. "Add a comment system to the blog" breaks into: design the data model, build the storage backend, create the submission UI, add moderation. Four tasks, strict ordering, one logical unit of work.

```go
type Epic struct {
    ID          string
    Title       string
    Description string
    TaskIDs     []string  // ordered child tasks
    Status      Status    // computed from children
}
```

Without epics, those four tasks are independent. The planner might reorder them. Task 3 might run before Task 2. Task 4 might run even if Task 2 failed.

Epic status is computed, not set:

```go
func DecideEpicStatus(tasks []Task) Status {
    if len(tasks) == 0 {
        return StatusPending
    }
    for _, t := range tasks {
        if t.Status == StatusFailed || t.Status == StatusCancelled {
            return StatusFailed
        }
    }
    // ... if all complete, return StatusComplete
    return StatusPlanned
}
```

If any child fails, the epic fails. You get fail-fast for free — no point building the submission UI if the storage backend can't be built.

Real Jira has story points, sprints, components, labels, assignees, custom fields. BTT epics have: title, description, ordered task list, computed status. AI agents don't need sprint planning. They need ordered execution with aggregate tracking.

## The State Machine: Why Tasks Need Statuses

BTT tasks move through nine statuses:

```go
const (
    StatusPending           // queued
    StatusPlanned           // planner has structured it
    StatusImplementing      // workflow is executing
    StatusInReview          // in a review step
    StatusComplete          // terminal: done
    StatusChangesRequested  // reviewer said revise
    StatusFailed            // terminal: gave up
    StatusCancelled         // terminal: user cancelled
    StatusWontDo            // terminal: planner rejected
)
```

The action function maps statuses to what happens next:

```go
func DecideTaskAction(task Task) TaskAction {
    switch task.Status {
    case StatusPlanned:
        return TaskImplement
    case StatusImplementing, StatusInReview:
        return TaskReview
    case StatusChangesRequested:
        if task.Attempt < task.MaxAttempts {
            return TaskReimplement
        }
        return TaskFail
    case StatusComplete, StatusWontDo:
        return TaskComplete
    default:
        return TaskNoop
    }
}
```

It's a state machine instead of a human dragging cards across a Kanban board. Without it:

- You can't resume after interruption. Which step was the task on?
- You can't enforce transitions. A task shouldn't go from "pending" to "complete" without passing through implementation.
- You can't limit retries. How many times has this task been through the revise loop?
- You can't answer "what state is this task in?" without inspecting the agent's context window.

AI agents are unreliable enough that you need the same tracking machinery humans use. The difference is nobody has to drag the card.

## The Workflow: Why Route Through Discrete Steps

This blog uses a five-step workflow:

```yaml
workflows:
  post:
    entry_step: research
    max_step_visits:
      draft: 6
    steps:
      research:  { mode: full,      transitions: { success: draft, failed: stop } }
      draft:     { mode: full,      transitions: { success: edit, already-done: done, failed: stop } }
      edit:      { mode: full,      transitions: { approved: qa, revise: draft, failed: stop } }
      qa:        { mode: full,      transitions: { passed: review, revise: draft, failed: stop } }
      review:    { mode: read-only, transitions: { approved: done, revise: draft, failed: stop } }
```

Why not let one agent do everything in a single pass? Because separation of concerns produces better results.

**Research → Draft.** An agent doing both tends to start writing before it has enough material. Separating them means the research step gathers everything first, and the drafter works from complete notes.

**Draft → Edit.** The editor is a different persona with different instructions. The drafter writes; the editor tightens. One agent doing both produces mediocre results at both — it's too attached to its own prose to cut it.

**Edit → QA.** QA is mechanical: do the links work? Does the code compile? Is the front matter correct? The editor focuses on voice and structure. Different skills, different prompts.

**QA → Review.** The final gate. And it's `read-only` — more on that below.

Each step gets a focused prompt with only the context it needs. The research step doesn't see the style guide. The QA step doesn't need the research notes. The workflow engine manages what each step sees, and that focus is what makes each step effective.

## The Revise Loop: Why Feedback Is Structural

The revise transition is the most important design decision in BTT. When any evaluating step — edit, QA, or review — returns `revise`, the workflow routes back to the draft step with the feedback appended to the task's context:

```go
if result.Status == "revise" && result.Feedback != "" {
    task.Context = append(task.Context,
        fmt.Sprintf("%s feedback: %s", stepName, result.Feedback))
    task.Attempt++
}
```

Without this, a bad draft means either accepting it (no quality gate), restarting the entire pipeline (wasting the research step's work), or having a human intervene.

With revise loops, the editor returns something like: "the intro doesn't match the style guide — too much preamble." That feedback gets appended to the task context. The draft step runs again, and this time it sees exactly what the previous attempt got wrong.

Two things make this work:

**Feedback accumulates.** On attempt 3, the drafter sees feedback from both the editor and the QA step. Fixing the editor's complaint doesn't re-introduce the QA issue because both are visible in the context.

**Revise always goes back to the producing step.** In this blog's workflow, edit, QA, and review all route `revise` back to `draft`. The draft step produces; the other steps evaluate. Same as human editing — the editor doesn't rewrite the piece, they send it back with notes.

This is the feature the incremental game was missing. The shell script had no mechanism for one step's output to inform a retry.

## Circuit Breakers: Why max_step_visits and MaxAttempts

Revise loops can go infinite. The reviewer sends it back, the drafter makes the same mistake, the reviewer sends it back again. Without a circuit breaker, this burns API credits until someone kills the process.

```yaml
max_step_visits:
  draft: 6
```

When the draft step exceeds 6 visits, the task fails:

```go
run.stepVisits[stepName]++
if max, ok := wf.MaxStepVisits[stepName]; ok && run.stepVisits[stepName] > max {
    task.Status = StatusFailed
    return
}
```

The limit is per-step, not per-workflow. A draft step might need 6 attempts to converge. A research step should succeed on the first try — if it doesn't, something is fundamentally wrong.

Two layers of defense: `max_step_visits` limits visits within a single workflow run, `task.MaxAttempts` limits retries across runs. The first prevents infinite loops. The second prevents retrying a fundamentally broken task forever.

Why 6? Enough to converge through feedback, not so many that you're burning money on something clearly stuck. Most tasks complete in 1-3 attempts. Hitting 6 means the prompt needs human attention.

## Read-Only Review: Why the Last Gate Can't Edit

The review step is `read-only`:

```go
const (
    ModeFull     ExecutionMode = "full"      // read + write + shell
    ModeGitOnly  ExecutionMode = "git-only"  // read + write + git only
    ModeReadOnly ExecutionMode = "read-only" // read only
)
```

The reviewer can read code, search for patterns, navigate the codebase. It cannot change a single file.

Without this constraint, reviewers "helpfully" fix small issues themselves instead of sending back feedback. Then they approve their own fixes. Who reviews the reviewer's changes? Nobody.

Read-only forces feedback to be the only output. The reviewer's only option when something is wrong is to return `revise` with clear notes. This forces specificity — "the intro spends 3 paragraphs before reaching the point; start with the problem statement" beats "improve the intro."

It's the same reason code review works the way it does. The reviewer comments on the PR. They don't push commits to your branch.

## The Trade-off

More steps mean more API calls and more latency. A 5-step blog workflow costs more than a single-shot prompt. That's real.

The argument is that quality justifies it. A blog post that goes through research, drafting, editing, QA, and review is better than a single-shot draft. The feedback loops catch problems a single agent wouldn't notice about its own work.

The simplified Jira model — tasks, epics, statuses, state machines, feedback loops, circuit breakers — is the minimum viable structure for reliable multi-step AI work. Every piece is load-bearing. Remove the state machine and you lose resumability. Remove the revise loop and you lose quality gates. Remove the circuit breaker and you lose cost control. Remove read-only review and you lose separation between producing work and evaluating it.

It's not project management for the sake of process. It's the smallest interface that makes AI agents produce work you'd actually ship.

---
title: "Building Build The Thing, Part 1: The Thesis"
date: 2026-03-22
tags: [ai, automation, tooling, go]
description: "The origin story of BTT: an experiment with AI-generated code that worked mechanically but failed structurally, and what that revealed about using AI as a content production system."
---

# Building Build The Thing, Part 1: The Thesis

Here's the idea: figure out how to get AI to generate something good, then use AI to generate *a lot* of content you find good. Code, prose, blog posts — doesn't matter. Get the recipe right, then scale it.

This isn't copilot-style autocomplete. Autocomplete is a human writing code with AI suggestions. I'm talking about the other direction: AI writes the code, and the human defines what "good" means.

That distinction matters. Autocomplete helps you type faster. What I wanted was a system that produces finished work — work I'd actually ship — without me writing every line.

So I ran an experiment.

## The Incremental Game

The setup: three AI agents orchestrated by a 55-line shell script. A TODO list with 23 tasks describing an incremental/idle game in Go. No human intervention once it started.

The three agents followed a TDD cycle:

1. **TDD Agent** — reads the next task, writes failing tests
2. **Implementor Agent** — makes those tests pass
3. **Refactor Agent** — cleans up the code

Each agent was a stateless `claude --print` call. The script processed tasks one at a time, ran the three agents in sequence, then marked the task done.

```bash
# The core loop (simplified)
claude --print --system-prompt "$(cat agents/tdd.md)" \
  "Process the next unchecked task from TODO.md. The task is: $TASK_DESC"

claude --print --system-prompt "$(cat agents/implement.md)" \
  "Make all failing tests pass. Do not modify test files."

claude --print --system-prompt "$(cat agents/refactor.md)" \
  "Review and refactor the implementation code. Do not touch tests."

sed -i '' "s/^- \[ \] .../- [x] .../" TODO.md
```

The agents built a real game: resource system, generators with exponential cost scaling, prestige mechanics, achievements, save/load, terminal UI. About 2,700 lines of tests across 22 test files. All tests pass.

That's the part that looks impressive. Now here's where it falls apart.

## Nine Problems

I grouped these after the fact, but they all showed up during the experiment as things that felt off.

### Coherence Problems

**Stateless agents can't maintain coherence.** Each `claude --print` invocation starts fresh. No agent reasons about the project's trajectory — only the snapshot it sees right now. Task 15 doesn't know what task 3 decided, except through whatever code happens to exist on disk.

**Isolated tasks produce incoherent systems.** The input handler (task 10) hardcodes buying a "Miner" at 10 gold. The catalog (task 11) defines a "gold mine" at 50 gold. Two purchase systems that never integrate.

```go
// input_handler.go (task 10)
gen := NewGenerator("Miner", "gold", 1.0)
cost := NewResource("gold", 10.0)

// catalog.go (task 11) — different name, different price, no connection
```

Both pass their tests. Neither knows the other exists.

**No entry point.** 23 tasks completed, all tests green, and no `main.go`. A thoroughly tested library that can't actually run. No agent noticed because "make the game runnable" wasn't a TODO item.

### Rigidity Problems

**Tests lock in bad API decisions.** The TDD agent wrote `HandleKeyPress` returning `interface{}`. Once tests encoded that contract, no subsequent agent could change the return type without breaking them. The one remaining TODO is a direct artifact of this — it's still unfixed.

**Refactoring can't cross the test boundary.** The refactor agent can't touch tests, so any structural improvement requiring an API change is off-limits. Reasonable rule in isolation, catastrophic in practice.

**Agents can't push back on the plan.** The TODO list is the fixed source of truth. If a task conflicts with existing code, there's no mechanism to flag it. The agent tries anyway and either produces something weird or silently fails.

### Quality Problems

**No verification gate.** The script marks tasks done unconditionally. If an agent fails, or produces garbage, or the tests don't actually compile — doesn't matter, the task is checked off and we move on.

**The three-agent split is artificial.** Real TDD is one mind holding context across test, implementation, and design. Splitting that across three stateless processes removes the judgment that makes TDD work. You get the ceremony of TDD without the substance.

**Shallow test quality.** Tests verify isolated behavior but miss integration. "Does buying work?" passes. "Does the catalog work?" passes. "Does buying through the input handler use the catalog?" Never tested.

## What the Experiment Proved

The thesis isn't wrong. It's incomplete.

AI can generate working code from structured instructions. 23 tasks, real game mechanics, comprehensive tests — the capability is real.

But "working" and "good" aren't the same thing. The code works locally — each unit does what it's supposed to. It fails globally — the units don't compose into a coherent system.

The missing ingredients:

- **Feedback loops.** When a reviewer spots a problem, someone needs to fix it. The incremental game had no reviewer and no loop back.
- **State between steps.** Each agent starts from zero context. Prior decisions, trade-offs, architectural choices — all lost.
- **Quality gates.** Something needs to verify that the output actually meets a bar before moving on. "The tests pass" isn't enough when the tests themselves might be wrong.

The structure around the AI matters as much as the AI itself.

## Enter BTT

That experiment was the direct motivation for Build The Thing. Six days later, I had a working tool that addressed each of those failures.

BTT is a workflow engine for AI agents. You define steps in YAML, connect them with transitions, and let the engine manage state, feedback, and quality gates.

The simplest workflow — the one for software engineering tasks — looks like this:

```
implement ──success──▶ review ──approved──▶ done
    ▲                    │
    └────── revise ──────┘
```

Two steps, one feedback loop. If the reviewer says "revise," the implementor gets the feedback and tries again. State carries forward. The reviewer is a real quality gate, not a checkbox.

Every step returns a structured envelope:

```json
{
  "status": "success | revise | failed",
  "summary": "what happened",
  "feedback": "what to improve (on revise)",
  "artifact": "the content produced"
}
```

The status drives transitions. The feedback accumulates across attempts. The artifact carries content forward. This is what the incremental game was missing — a way for the output of one step to inform the next.

Because workflows are YAML-defined, the same engine handles different domains. A book-writing workflow adds plot and editor steps. A blog workflow adds research and style-checking. Different YAML, different prompts, no code changes.

The initial commit was about 1,950 lines of Go with a single external dependency: `gopkg.in/yaml.v3`. By the end of that first week, it supported multiple AI providers, effort-based model routing, and three domain templates.

Part 2 will walk through how BTT actually works — the workflow engine, the prompt system, and what it looks like to define a new domain from scratch.

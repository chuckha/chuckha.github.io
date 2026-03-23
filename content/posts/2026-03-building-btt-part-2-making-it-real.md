---
title: "Building Build The Thing, Part 2: Making It Real"
date: 2026-03-22
tags: [ai, automation, tooling, go, architecture]
description: "How BTT went from a hardcoded 3-agent pipeline to a general-purpose YAML workflow engine in 7 days — the key decisions, pivots, and what actually mattered."
---

Part 1 laid out the pitch: BTT is a YAML-driven workflow engine for AI agents. The result envelope, the feedback loop, the quality gate — introduced but not explained.

This post is the explanation. Seven days, 48 commits, three major pivots.

## Day 1: The Hardcoded Pipeline

The first commit: about 1,950 lines of Go with one external dependency (`yaml.v3`). Three agents, hardcoded:

- `planner.go` — breaks a feature request into epics and tasks
- `implementor.go` — writes code
- `reviewer.go` — reviews the code

Fixed pipeline: plan → implement → review → approve or reject. The implementor ran Claude with `--dangerously-skip-permissions` because it needed to write files. The reviewer could approve or send it back.

Same day I added multi-epic planning, token counting from Claude CLI stream events, and standalone tasks that didn't need the epic wrapper. Ship a feature request, get code out.

Simple enough to build in a day. That simplicity was also its ceiling.

## Day 2: Adding Agents, Hitting a Wall

Day 2 was a feature spree. Five new capabilities layered onto the fixed pipeline:

**Git-aware review.** A `SourceControl` interface that captures HEAD before and after implementation, so the reviewer inspects actual diffs instead of guessing what changed.

**QA agent.** Runs after review when a `.btt/qa.md` file exists. Separate quality check with its own instructions.

**Bug reproduction agent.** Runs before implementing bugfixes. Tries to reproduce the bug first so the implementor has a concrete failure to fix.

**QA instruction auto-refresher.** Rewrites `qa.md` when it detects the instructions are stale. This one was too clever by half — more on that later.

**Web dashboard.** SSE-driven real-time UI showing agent progress. 1,876 new lines.

By end of day 2 I had five agent types, a web UI, and a problem. Every new agent meant a new Go file, a new result type, new branches in the pipeline logic. Adding a domain that wasn't software engineering — say, blog writing — would mean writing more Go.

The architecture was fighting the thesis. New workflows shouldn't require new code.

## The Pivot: YAML Workflow Engine

Day 3. The single largest commit: 1,577 insertions, 1,014 deletions across 23 files.

I deleted every hardcoded agent:

- `internal/agent/implementor.go` — gone
- `internal/agent/reviewer.go` — gone
- `internal/agent/qa.go` — gone
- `internal/agent/qaupdater.go` — gone
- `internal/agent/reproduce.go` — gone

Replaced them with four generic files:

- `workflow.go` — YAML config parsing, step definitions with transitions
- `template.go` — Go `text/template` rendering for prompts
- `step.go` — universal step executor
- `prompt.go` — data struct for template variables

Five agent types became one. What makes a step an implementor vs. a reviewer is the prompt template and the execution mode, not the Go code.

The SWE feature workflow:

```yaml
workflows:
  feature:
    entry_step: implement
    max_step_visits:
      implement: 6
    steps:
      implement:
        mode: full
        transitions:
          success: review
          already-done: done
          failed: stop
      review:
        mode: read-only
        transitions:
          approved: done
          revise: implement
          failed: stop
```

Same diagram as Part 1, but as data. Adding a step means adding YAML, not Go.

Four more commits that day tightened the design:

- Auto-discover prompt templates from a `prompts/` directory (convention over configuration)
- Instructions and codebase map as template variables
- `missingkey=error` on all templates — if a variable is missing, it blows up instead of silently rendering empty
- Pre-formatted optional sections that return empty strings instead of nil

That last one deserves its own section.

## The Result Envelope

Every step returns a structured result. The shape is fixed:

```go
type StepResult struct {
    Status   string `json:"status"`
    Summary  string `json:"summary"`
    Feedback string `json:"feedback"`
    Artifact string `json:"artifact"`
}
```

Status drives transitions. Summary is for logging. Feedback accumulates across revision attempts. Artifact carries content forward between steps (a draft, a plan, a diff).

The schema is built dynamically per step, constraining status to the values that step's transitions allow:

```go
func buildStepResultSchema(allowedStatuses []string) string {
    statusProp := map[string]any{"type": "string"}
    if len(allowedStatuses) > 0 {
        statusProp["enum"] = allowedStatuses
    }
    schema := map[string]any{
        "type": "object",
        "properties": map[string]any{
            "status":   statusProp,
            "summary":  map[string]any{"type": "string"},
            "feedback": map[string]any{"type": "string"},
            "artifact": map[string]any{"type": "string"},
        },
        "required":             []string{"status", "summary", "feedback", "artifact"},
        "additionalProperties": false,
    }
    b, _ := json.Marshal(schema)
    return string(b)
}
```

If a review step allows `approved`, `revise`, and `failed`, the schema enforces that. The AI can't return `success` or `maybe` or anything else. You know exactly which states are reachable from each step.

The `additionalProperties: false` and `required` on all fields were added later for OpenAI Codex compatibility. Claude was more lenient about schema strictness. Two bug-fix commits just for that.

## Prompt Templates

Each step has a markdown prompt template. Here's what the implement step looks like:

```markdown
# Project Guidelines
{{ .Instructions }}

# Codebase Map
{{ .CodebaseMap }}

You are a senior software engineer implementing a task.

Task: {{ .Task.Title }}

Description:
{{ .Task.Description }}

{{ .ContextSection }}
{{ .LatestOutputSection }}
{{ .ActionItemsSection }}

Implement the task. Write production-quality code.
Commit your changes when done with a clear commit message.

You must set status to one of: {{ .AllowedStatuses }}
```

Sections like `ContextSection` and `ActionItemsSection` are pre-formatted in the orchestrator:

```go
func formatContextSection(ctx []string) string {
    if len(ctx) == 0 {
        return ""
    }
    var b strings.Builder
    b.WriteString("Context (relevant files, packages, review feedback):\n")
    for _, item := range ctx {
        fmt.Fprintf(&b, "- %s\n", item)
    }
    return b.String()
}
```

When there's no context, the function returns `""`. The template renders an empty string. No stray headers, no `{{ if }}` guards.

The early version had `{{ if .Context }}` guards wrapping every optional section. Switching to pre-formatted sections with `missingkey=error` made prompt authoring much simpler. Write the template once, every variable is always present, and a typo in a variable name fails loudly instead of rendering empty.

## The Workflow Loop

The orchestrator is a `for` loop with a map lookup. Here's the core, simplified:

```go
run := &workflowRun{
    workflow:    wf,
    currentStep: wf.EntryStep,
    stepVisits:  make(map[string]int),
}

for {
    stepName := run.currentStep
    stepCfg := wf.Steps[stepName]

    // Prevent infinite loops
    run.stepVisits[stepName]++
    if max, ok := wf.MaxStepVisits[stepName]; ok && run.stepVisits[stepName] > max {
        task.Status = StatusFailed
        return
    }

    // Render prompt, execute step
    templateData := o.buildPromptData(task, run, stepCfg, beforeSHA, "")
    prompt := o.renderStepPrompt(stepName, stepCfg, templateData)
    result, meta := o.executor.RunStep(ctx, prompt, stepCfg.Mode, ...)

    run.history = append(run.history, result)

    // Resolve transition
    nextStep, done, err := ResolveTransition(stepCfg, result)
    if err != nil {
        task.Status = StatusFailed
        return
    }
    if done {
        task.Status = StatusComplete
        return
    }

    // Accumulate feedback on revise
    if result.Status == "revise" && result.Feedback != "" {
        task.Context = append(task.Context,
            fmt.Sprintf("%s feedback: %s", stepName, result.Feedback))
        task.Attempt++
    }

    run.currentStep = nextStep
}
```

The transition resolver:

```go
func ResolveTransition(step StepConfig, result StepResult) (string, bool, error) {
    target, ok := step.Transitions[result.Status]
    if !ok {
        return "", false, fmt.Errorf("unrecognized status %q", result.Status)
    }
    switch target {
    case "done":
        return "", true, nil
    case "stop":
        return "", false, nil
    default:
        return target, false, nil
    }
}
```

The feedback accumulation is what makes revision cycles work. When a reviewer returns `revise` with feedback like "the error handling doesn't cover the timeout case," that gets appended to the task context. Next implementation attempt, the implementor sees all prior feedback in its prompt. Attempt 3 knows what attempts 1 and 2 got wrong.

This is the feedback loop the incremental game was missing. The shell script had no mechanism for one step's output to inform a retry. BTT makes it structural.

## Multiple Domains, No Code Changes

The same engine runs different workflows by swapping YAML. The blog post workflow:

```yaml
workflows:
  post:
    entry_step: research
    max_step_visits:
      draft: 6
    steps:
      research:
        mode: full
        transitions:
          success: draft
          failed: stop
      draft:
        mode: full
        transitions:
          success: edit
          already-done: done
          failed: stop
      edit:
        mode: full
        transitions:
          approved: qa
          revise: draft
          failed: stop
      qa:
        mode: full
        transitions:
          passed: review
          revise: draft
          failed: stop
      review:
        mode: read-only
        transitions:
          approved: done
          revise: draft
          failed: stop
```

Five steps instead of two. Research feeds into drafting, editing can kick it back, QA checks style and accuracy, review is the final gate. Different graph, different prompts, same engine.

The book-writing workflow adds a plot step and replaces QA with an editor:

```yaml
workflows:
  chapter:
    entry_step: plot
    max_step_visits:
      write: 6
    steps:
      plot:
        mode: full
        transitions:
          success: write
          failed: stop
      write:
        mode: full
        transitions:
          success: editor
          already-done: done
          failed: stop
      editor:
        mode: full
        transitions:
          approved: review
          revise: plot
          failed: stop
      review:
        mode: read-only
        transitions:
          approved: done
          revise: plot
          failed: stop
```

Three domain templates ship with BTT — `swe`, `blog`, and `book` — embedded in the binary via `go:embed`:

```go
//go:embed templates/*
var templatesFS embed.FS
```

`btt init blog` copies the template into your project's `.btt/` directory. From there, everything is customizable — workflow YAML, prompt templates, instructions. Zero Go changes to support a new domain.

## Execution Modes

Each step declares what the AI is allowed to do:

```go
const (
    ModeFull     ExecutionMode = "full"      // read + write + shell
    ModeGitOnly  ExecutionMode = "git-only"  // read + write + git only
    ModeReadOnly ExecutionMode = "read-only" // read only
)
```

Review steps use `read-only`. The reviewer can inspect code but can't "helpfully" fix things itself — without this, you get reviewers that approve their own edits, defeating the point of a separate review step.

Implementation steps use `full`. Infrastructure tasks use `git-only` — can commit files but can't run arbitrary shell commands.

## Provider Abstraction

By day 6 I wanted different models for different tasks. The runner interface:

```go
type Runner interface {
    Run(ctx context.Context, prompt string, streamLog io.Writer) (string, RunMeta, error)
    RunJSON(ctx context.Context, prompt string, schema string, dest any, streamLog io.Writer) (RunMeta, error)
}
```

The factory picks a runner based on config:

```go
func NewRunner(provider string, model string, stderr io.Writer, extraArgs ...string) (Runner, error) {
    switch provider {
    case "claude", "":
        return NewCLIRunner("claude", stderr, args...), nil
    case "codex":
        return NewCodexRunner(stderr, args...), nil
    default:
        return nil, fmt.Errorf("unknown provider %q", provider)
    }
}
```

Effort levels map task complexity to provider and model:

```yaml
effort_levels:
  low:
    name: claude
    model: haiku
  medium:
    name: claude
    model: sonnet
  high:
    name: claude
    model: opus
```

Resolution priority: step-level override beats task effort level beats default provider. A review step can use Haiku while implementation uses Opus.

One surprise: Claude sometimes puts the structured result in a `StructuredOutput` tool use block instead of the result event's `structured_output` field. The runner scans for both. Without that fallback, about 10% of runs failed to parse.

## What Worked, What Didn't, What Surprised

**The YAML pivot was the single best decision.** Five hardcoded agents to a generic step executor on day 3 — everything after that (multiple domains, init templates, provider abstraction) happened without touching the core engine. If I'd kept adding Go agent files, the blog and book templates wouldn't exist.

**Structured contracts between steps beat shared context.** BTT's agents are stateless. Each starts fresh with a templated prompt. State moves through the result envelope and accumulated task context, not shared memory or conversation history. You can look at a prompt template and know exactly what information the agent has.

**The things I scrapped were too clever.** The QA auto-updater that rewrote instruction files? Gone — replaced by doc files the user edits. The `{{ if }}` template guards? Gone — always-present variables. A `no-shell` execution mode? Gone — three modes covered everything.

**One dependency is a feature.** `gopkg.in/yaml.v3` and the Go standard library. No framework, no ORM, no HTTP router (just `net/http`). Not a philosophical stance — it's what happened when I only added dependencies I needed. YAML parsing is the only thing the stdlib doesn't handle well.

**The hardcoded approach hit its wall faster than expected.** Two days. Five agent types and it was clear that six would be painful. The architecture forced the pivot early — a gift. A more flexible initial design might have limped along for weeks before the same realization hit.

## The Numbers

48 commits over 7 days. The workflow engine is about 150 lines — a map lookup and a for loop. The orchestrator is about 690 lines, most of it prompt template rendering and logging. The step executor is 67 lines. The Claude CLI runner is 235 lines.

The complexity isn't in the engine. It's in the prompts. Getting a reviewer to give actionable feedback instead of vague approval took more iteration than any Go code. The engine is boring infrastructure. The prompts are where the craft is.

The thesis from Part 1 — that the structure around AI matters as much as the AI itself — turned out to be the operating principle for the entire build. The engine is a for loop and a map. The prompts are the product.

# Architecture

## Overview

Eval Insights is not a fixed pipeline. It is a **TrueForge agent** driven by the
**Eval Investigation Skill**, given autonomy to decide what evidence it needs
and in what order to gather it. The diagram below shows the systems it can
draw on, not a linear sequence of steps.

```text
                       USER

             "Investigate keyword_overlap"

                         │
                         ▼

              ┌─────────────────────┐
              │      TrueForge      │
              │                     │
              │ Eval Insights Agent │
              └──────────┬──────────┘
                         │
                         │
                Eval Investigation
                      Skill
                         │
            ┌────────────┴────────────┐
            │                         │
            ▼                         ▼

     ┌─────────────┐           ┌─────────────┐
     │  Langfuse   │           │   GitHub    │
     │     MCP     │           │     MCP     │
     ├─────────────┤           ├─────────────┤
     │ Scores      │           │ Grader code │
     │ Traces      │           │ Agent prompt│
     │ Feedback    │           │ Tools       │
     │ Tool calls  │           │ Dataset     │
     │ Metadata    │           │ Middleware  │
     └──────┬──────┘           └──────┬──────┘
            │                         │
            └────────────┬────────────┘
                         │
                         ▼

               Hypothesis-driven
                  investigation

                         │
                         ▼

                Root-cause model

                         │
                         ▼

          ┌────────────────────────────┐
          │ Interactive Investigation  │
          │           Report           │
          ├────────────────────────────┤
          │ Grader explanation         │
          │ Failure categories         │
          │ Failure percentages        │
          │ Trace drill-down           │
          │ Langfuse trace links       │
          │ Code evidence              │
          │ Agent health               │
          │ Grader health              │
          │ Recommended fixes          │
          └────────────────────────────┘
```

## Components

### 1. TrueForge (agent harness and runtime)

TrueForge is the environment the investigator runs in — it is not a thin
wrapper around a single LLM call. It provides:

- the autonomous reasoning/tool-call loop
- the Eval Investigation Skill definition
- MCP tool access (Langfuse, GitHub)
- persistent investigation context across many tool calls
- the ability to delegate work to subagents for larger investigations
- the artifact-building capability used to render the final report

We deliberately do **not** hard-code a workflow such as
`get_scores() → get_traces() → inspect_grader() → generate_report()`.
Instead, the skill defines the objective, the evidence sources, the
root-cause taxonomy, and the standard of evidence — and TrueForge lets the
agent decide the actual investigation path per case.

### 2. Eval Investigation Skill (`SKILL.md`)

Packages the investigation methodology: how to reason about a low score,
which root-cause categories to consider, when runtime evidence is enough vs.
when to go read source code, how to cluster recurring failures, and what the
finished report must contain. See `SKILL.md` for the full definition.

### 3. Langfuse MCP — runtime evidence

Gives the agent access to:

- grader scores per trace
- full traces (input, tool calls, model generations, final response)
- grader feedback/explanations
- metadata needed to compare passing vs. failing traces

This is where an investigation typically starts: which traces are failing,
and what do they have in common?

### 4. GitHub MCP — implementation evidence

Gives the agent access to the connected repository so it can follow the
code path behind a failure:

```text
Agent Prompt → Agent Implementation → Tool Definition → Tool Wrapper
→ Middleware → Runtime/Harness → Final Response → Grader
```

This is where the agent answers questions Langfuse traces alone can't:
Is the grader implementation correct? Is the "missing" behavior actually
enforced somewhere else (e.g. a runtime approval gate the grader can't see)?
Is the reference/dataset wrong?

### 5. Hypothesis-driven investigation engine

The agent maintains competing explanations for a failure (see `SKILL.md`'s
hypothesis taxonomy — H1–H8) and chooses its next tool call based on which
evidence would best distinguish between them, rather than following a fixed
checklist.

### 6. Root-cause model

Once evidence is gathered, each failure is assigned:

- a **root-cause category** (Agent Prompt, Agent Behavior, Tool/Data,
  Grader Implementation, Grader Design, Grader Rubric Misalignment, Grader
  Observability Gap, or Reference/Dataset)
- a **responsible layer** (who actually owns the fix)
- a **confidence** level, grounded in the evidence collected

### 7. Failure clustering

Individual failing traces are grouped into recurring failure modes with
counts and percentage-of-failures, so the report surfaces "what to fix
first" instead of a flat list of failed traces.

### 8. Interactive Investigation Report (renderer)

Built with TrueForge's web-artifact capability. Structure:

1. **Grader explanation** — what the grader measures, how, and its
   observable scope, before any failures are shown.
2. **Failure category distribution** — clickable table of clusters with
   root cause, trace count, % of failures, confidence, severity.
3. **Category → trace drill-down** — every trace belonging to a category.
4. **Trace detail** — input, response, score, feedback, expectation,
   diagnosis, evidence, responsible layer, confidence, recommended fix,
   and a link to the real Langfuse trace (never fabricated).
5. **Grader health vs. agent health** — kept as separate sections so a
   poorly-designed grader doesn't get "fixed" by degrading a healthy agent.
6. **Recommended actions** — each tagged with a target (`FIX AGENT`,
   `FIX GRADER`, `FIX DATASET`, `FIX TOOL`, `FIX INFRASTRUCTURE`, or
   `NO CHANGE REQUIRED`), priority, evidence, confidence, and source file
   when available.

See `examples/sample-output.json` for the structured shape and
`examples/sample-report.html` for the rendered form.

## Scaling: subagent fan-out

For small trace sets, the lead investigator does everything directly. For
larger evaluation runs, TrueForge subagents can parallelize evidence
gathering while the lead investigator stays responsible for reconciling
findings and producing the final diagnosis:

```text
                   Lead Investigator
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
          Trace         Grader         Code
       Investigator     Auditor    Investigator
          (traces)    (grader/     (prompt/tools/
                       rubric/       middleware/
                       reference)    runtime)
             │            │            │
             └────────────┼────────────┘
                          │
                          ▼
                  Lead Investigator
                          │
                          ▼
                 Failure clustering
                          │
                          ▼
                  Root-cause report
```

## Design principles

- **A low score is a symptom, not a verdict.** The agent never assumes the
  evaluated agent is at fault; it investigates who actually is.
- **Evidence over assumption.** Every claim in the report must be backed by
  a real trace, score, feedback string, or source file — never invented.
- **Clusters over lists.** Failures are grouped by root cause, not
  enumerated one by one.
- **Grader health and agent health are tracked separately** so fixes land
  on the right target.
- **Never fabricate a Langfuse trace URL.** Only link to traces the agent
  actually retrieved.

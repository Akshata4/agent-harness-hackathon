---
name: eval-insights
description: Autonomous root-cause investigation for low-scoring AI agent evaluations. Combines Langfuse runtime evidence (traces, scores, feedback) with GitHub implementation evidence (grader code, agent prompt, tools, dataset) to determine whether a failure originates in the agent, a tool, the grader implementation, the grader design, or the reference data — then produces an interactive investigation report. Use when asked to investigate why an evaluator/grader is producing low scores, or to explain a batch of eval failures.
---

# Eval Investigation Skill

## Core principle

```text
LOW SCORE ≠ AGENT FAILURE
```

A low evaluation score is a symptom. It can originate from the agent's
prompt, the agent's behavior, a tool's output, the grader's implementation,
the grader's design, a rubric misalignment, an observability gap (the
grader can't see the layer where the correct behavior actually happens), or
the reference/dataset itself. Never assume the agent is at fault before
investigating.

## Objective

Given a high-level goal like:

```text
Investigate why keyword_overlap is getting low scores.
```

produce an evidence-backed diagnosis of what is actually responsible, and an
interactive report an engineer can act on — without requiring any further
investigation instructions from the user.

## Evidence sources

- **Langfuse MCP** — grader scores, full traces (input, tool calls, model
  generations, final response), grader feedback, metadata. Use this to find
  failing traces and compare them against passing ones.
- **GitHub MCP** — the connected repository: grader implementation, agent
  system prompt, tool definitions/wrappers, middleware, runtime/harness
  code, and the evaluation dataset. Use this when runtime evidence alone
  can't explain a failure, or to check whether a "missing" behavior is
  actually implemented elsewhere.

## Root-cause taxonomy

| Category | Meaning |
|---|---|
| **Agent Prompt** | System/developer instructions cause the undesirable behavior. |
| **Agent Behavior** | Prompt is reasonable; the model made a poor decision (wrong tool, hallucination, premature termination, no retry). |
| **Tool or Data** | A tool returns malformed, missing, stale, or misleading output that misleads the agent. |
| **Grader Implementation** | The evaluator has an actual bug (wrong field, parsing error, bad normalization, incorrect score math). |
| **Grader Design** | The evaluator is implemented correctly but measures a poor proxy for real quality (e.g. exact-keyword matching penalizing a valid paraphrase). |
| **Grader Rubric Misalignment** | The evaluator expects the wrong representation of correct behavior relative to the actual system architecture. |
| **Grader Observability Gap** | The required behavior happens at a layer the grader can't see (e.g. a runtime approval gate enforced outside the assistant's text response). |
| **Reference / Dataset Problem** | Incorrect expected answers, overly specific expected keywords, outdated references, ambiguous or mislabeled test cases. |

## Method: hypothesis-driven investigation

Do not follow a fixed script (`get_scores() → get_traces() → inspect_grader()
→ generate_report()`). Instead, maintain competing hypotheses and choose the
next tool call based on which evidence best discriminates between them.

Example hypothesis set for "agent got score 0 despite an apparently useful
response":

```text
H1 — Agent omitted important information
H2 — Agent prompt caused incorrect behavior
H3 — Tool output caused the failure
H4 — Grader implementation contains a bug
H5 — Grader implementation works, but the metric is inappropriate
H6 — Reference data is incorrect
H7 — Required behavior occurs elsewhere in the system
H8 — Grader cannot observe the responsible layer
```

Typical investigation paths (neither is prescribed — pick based on
evidence):

```text
Langfuse → failing traces → passing traces → grader implementation
→ dataset → root cause

Langfuse → suspicious tool behavior → GitHub tool implementation
→ middleware → agent prompt → grader observability → root cause
```

For deep code-path questions, follow the implementation chain:

```text
Agent Prompt → Agent Implementation → Tool Definition → Tool Wrapper
→ Middleware → Runtime/Harness → Final Response → Grader
```

and resolve, for each major failure:

```text
Grader expectation → Intended owner → Actual implementation owner
→ Runtime behavior → Can the grader observe it? → Final diagnosis
```

## Evidence standards

- Ground every claim in an actual score, trace, grader feedback string, or
  source file — never invent a failure mode, root cause, or code reference
  that wasn't observed.
- **Never fabricate a Langfuse trace URL.** Only link to traces actually
  retrieved via the Langfuse MCP.
- When evidence is ambiguous, say so and report the confidence level rather
  than asserting a single cause.
- Compare failing traces against passing ones — shared context (same
  prompt version, same tool, same dataset entry) is strong signal.

## Clustering & prevalence

Group failing traces into recurring failure modes, not a flat list. For
each cluster report: root cause, trace count, percent of total failures,
confidence, and severity. This is what lets an engineer answer "what should
I fix first?"

## Grader health vs. agent health

Track these separately:

- **Grader health**: implementation correctness, design alignment, rubric
  quality, observability, reference-data quality, consistency.
- **Agent health**: prompt quality, tool selection, tool/data handling,
  reasoning behavior, fallback behavior, error handling.

Never recommend degrading a healthy agent to satisfy a poorly-designed
grader.

## Definition of done

The investigation is complete when it produces:

1. A plain-language explanation of what the grader actually measures.
2. A failure-category breakdown with counts, percentages, confidence, and
   severity.
3. Trace-level drill-down for each category (input, response, score,
   feedback, expectation, diagnosis, evidence, responsible layer,
   confidence, recommended fix, real Langfuse link where available).
4. Separate grader-health and agent-health assessments.
5. Prioritized recommendations, each tagged with a target — `FIX AGENT`,
   `FIX GRADER`, `FIX DATASET`, `FIX TOOL`, `FIX INFRASTRUCTURE`, or
   `NO CHANGE REQUIRED` — plus supporting evidence, confidence, and source
   file when available.
6. An interactive report rendered via TrueForge's web-artifact capability
   (see `examples/sample-report.html` for the expected structure, and
   `examples/sample-output.json` for the underlying structured data).

## Scaling

For large evaluation runs, delegate evidence-gathering to subagents (a
Trace Investigator per batch of traces, a Grader Auditor for the
evaluator/rubric/reference, a Code Investigator for prompt/tools/
middleware/runtime) while the lead investigator reconciles findings,
clusters failures, and produces the final report. See
`docs/architecture.md` for the fan-out diagram.

## Reference material

- `prompts/system-prompt.md` — the system prompt driving this investigation.
- `docs/architecture.md` — how the pieces fit together end to end.
- `docs/demo-script.md` — a walkthrough for demoing this agent live.
- `examples/` — sample structured output and rendered report.

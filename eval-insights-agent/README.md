# 🔍 Eval Insights

### Autonomous Root-Cause Analysis for AI Agent Evaluations

> **Eval platforms tell you what failed. Eval Insights investigates why.**

Eval Insights is an autonomous evaluation-debugging agent built on **TrueForge**.

It investigates low-scoring AI agent evaluations by combining:

- runtime evidence from **Langfuse**
- implementation evidence from **GitHub**
- autonomous reasoning and tool orchestration through **TrueForge**
- a reusable **Eval Investigation Skill**
- interactive investigation reports generated directly by the agent

Instead of requiring an engineer to manually inspect dozens of traces, grader outputs, prompts, tools, and source files, Eval Insights performs the investigation end-to-end and identifies **what is actually responsible for the failure**.

---

# 🚨 The Problem

Teams building AI agents increasingly rely on evaluation pipelines containing:

- deterministic graders
- LLM-as-a-judge evaluators
- reference-answer checks
- tool-use graders
- safety graders
- instruction-following graders

Platforms such as Langfuse make it easy to collect:

- traces
- grader scores
- evaluator feedback
- tool calls
- model generations

But a low score still leaves an important question unanswered:

> **Why did this evaluation fail?**

Consider an evaluator dashboard:

| Grader | Score |
|---|---:|
| Correctness | 0.72 |
| Tool Selection | 0.64 |
| Safety | 0.91 |
| Instruction Following | 0.68 |

Knowing that `Tool Selection = 0.64` is useful.

But an engineer still needs to determine:

- Which traces are responsible?
- Do those traces share a common pattern?
- Did the agent actually behave incorrectly?
- Did the system prompt cause the behavior?
- Was a tool response responsible?
- Is the grader implementation buggy?
- Is the grader simply too strict?
- Is the reference answer wrong?
- Is the grader evaluating the wrong system layer?
- Is the supposedly missing behavior already enforced elsewhere?

Answering those questions usually requires manually moving between:

**evaluation platform → traces → grader implementation → dataset → agent prompt → tools → application code**

This becomes slow and repetitive as evaluation suites grow.

---

# 💡 The Idea

Eval Insights acts as an **autonomous evaluation investigator**.

An engineer provides a high-level goal such as:

```text
Investigate why keyword_overlap is getting low scores.
```

That is the only investigation instruction required.

The agent then determines what evidence it needs and investigates autonomously.

For example, it may decide to:

1. inspect grader scores in Langfuse
2. examine low-scoring traces
3. compare them against successful traces
4. inspect grader feedback
5. search GitHub for the grader implementation
6. inspect expected/reference values
7. inspect the evaluated agent's prompt
8. follow relevant tool or middleware code
9. form competing root-cause hypotheses
10. validate those hypotheses against runtime and code evidence
11. cluster recurring failure modes
12. calculate their prevalence
13. recommend the correct engineering fix
14. generate an interactive investigation report

The investigation path is **not hard-coded**.

The agent chooses what to inspect next based on the evidence it discovers.

---

# 🧠 A Low Score Does Not Mean the Agent Is Wrong

This is the core principle behind Eval Insights.

```text
LOW SCORE ≠ AGENT FAILURE
```

A low evaluation score could originate from many places.

Eval Insights distinguishes between categories including:

### Agent Prompt

The system/developer instructions cause undesirable behavior.

Example:

```text
"If internal search fails, use web search."
```

The agent interprets an empty result as a failure and unnecessarily switches to web search.

---

### Agent Behavior

The prompt is reasonable, but the model makes a poor decision.

Examples:

- wrong tool selected
- premature termination
- hallucination
- unnecessary tool calls
- failure to retry

---

### Tool or Data

The agent behaves incorrectly because a tool returns:

- malformed output
- missing fields
- stale information
- misleading empty results
- errors

---

### Grader Implementation

The evaluator itself contains an implementation bug.

Examples:

- incorrect score calculation
- wrong field evaluated
- parsing bug
- normalization error

---

### Grader Design

The evaluator implementation works correctly, but what it measures is a poor proxy for actual agent quality.

Example:

```text
Expected keywords:
["outside", "scope", "phone help"]
```

The agent gives a perfectly reasonable response:

```text
I can't book the ticket directly, but here's how you can
safely book it using Safari...
```

The response may be useful and correct but receive:

```text
keyword_overlap = 0
```

because it does not contain the exact expected wording.

The evaluator works as implemented.

The **evaluation design is the problem**.

---

### Grader Rubric Misalignment

The evaluator expects the wrong representation of correct behavior.

For example:

```text
Grader:
"The assistant must explicitly say the request is outside its scope."
```

But the application architecture may already classify and safely route unsupported requests without requiring that exact phrase.

Changing the production agent to satisfy the evaluator would optimize the product for the benchmark rather than improve the product.

---

### Grader Observability Gap

Sometimes the required behavior happens outside the layer visible to the grader.

Example:

```text
Agent
   ↓
delete_user()
   ↓
TrueForge / Runtime
   ↓
⚠ Human approval required
   ↓
Execution
```

A grader examining only the assistant response might report:

```text
FAIL: Agent did not request confirmation.
```

But confirmation is already enforced by the runtime.

Eval Insights follows the implementation path before deciding who is responsible.

---

### Reference / Dataset Problems

Evaluation failures can also originate from:

- incorrect expected answers
- overly specific expected keywords
- outdated references
- ambiguous test cases
- mislabeled examples

---

# 🔎 Deep Code-Path Investigation

Eval Insights does not stop after reading a trace.

When runtime evidence is insufficient, the agent can inspect the connected GitHub repository.

It may investigate:

```text
Agent Prompt
     ↓
Agent Implementation
     ↓
Tool Definition
     ↓
Tool Wrapper
     ↓
Middleware
     ↓
Runtime / Harness
     ↓
Final Response
     ↓
Grader
```

This allows the investigator to answer an important question:

> **Is the behavior flagged by the grader actually missing, or is it implemented somewhere else?**

For every major failure, the agent can determine:

```text
Grader expectation
        ↓
Intended owner
        ↓
Actual implementation owner
        ↓
Runtime behavior
        ↓
Can the grader observe it?
        ↓
Final diagnosis
```

This prevents recommendations such as modifying an agent prompt when the actual problem is the evaluator.

---

# 🏗️ Architecture

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

---

# ⚙️ How TrueForge Is Used

TrueForge is not used as a thin wrapper around an LLM.

It is the **agent harness and runtime** for Eval Insights.

TrueForge provides the environment in which the investigator:

- maintains the autonomous reasoning/tool loop
- uses the Eval Investigation Skill
- calls external MCP tools
- queries Langfuse
- searches GitHub
- follows evidence across systems
- maintains investigation context
- can delegate analysis to subagents
- generates the final interactive artifact

We intentionally do **not** hard-code a workflow such as:

```python
get_scores()
get_traces()
inspect_grader()
inspect_prompt()
generate_report()
```

Instead, the skill defines:

- the investigation objective
- available evidence
- root-cause taxonomy
- evidence standards
- investigation quality requirements
- definition of done

TrueForge allows the agent to determine the investigation path dynamically.

For one failure, the agent may follow:

```text
Langfuse
→ failing traces
→ passing traces
→ grader implementation
→ dataset
→ root cause
```

Another investigation might require:

```text
Langfuse
→ suspicious tool behavior
→ GitHub tool implementation
→ middleware
→ agent prompt
→ grader observability
→ root cause
```

The next action depends on the evidence.

---

# 🧩 Eval Investigation Skill

The investigation methodology is packaged as a reusable TrueForge skill:

```text
SKILL.md
```

The skill teaches the agent:

- how to reason about evaluation failures
- not to blindly trust grader scores
- how to form competing hypotheses
- when runtime evidence is insufficient
- how to investigate implementation evidence
- how to follow code paths
- how to identify responsibility ownership
- how to detect grader observability gaps
- how to compare passing vs failing traces
- how to cluster recurring failures
- how to calculate failure prevalence
- how to generate evidence-backed recommendations
- how to produce the final interactive investigation artifact

The skill defines **how to investigate well**, not a fixed sequence of tool calls.

---

# 🔄 Hypothesis-Driven Investigation

Eval Insights continuously considers competing explanations.

For example:

```text
Observed:
Agent receives score 0 despite apparently useful response.
```

Possible hypotheses:

```text
H1 — Agent omitted important information

H2 — Agent prompt caused incorrect behavior

H3 — Tool output caused the failure

H4 — Grader implementation contains a bug

H5 — Grader implementation works,
     but the metric is inappropriate

H6 — Reference data is incorrect

H7 — Required behavior occurs elsewhere

H8 — Grader cannot observe the responsible layer
```

The agent determines what evidence would best distinguish these hypotheses and chooses its next tool call accordingly.

---

# 📊 Failure Clustering

Eval Insights converts individual trace failures into recurring failure modes.

Instead of:

```text
Trace 12 failed
Trace 19 failed
Trace 24 failed
Trace 31 failed
...
```

the engineer sees:

| Failure Category | Root Cause | Traces | % of Failures |
|---|---|---:|---:|
| Valid paraphrase penalized | Grader Design | 11 | 61% |
| Over-specific reference | Dataset | 4 | 22% |
| Actual agent failure | Agent Behavior | 2 | 11% |
| Tool/data issue | Tool/Data | 1 | 6% |

This helps answer:

> **What should I fix first?**

---

# 🖥️ Interactive Investigation Report

The final investigation is rendered as an interactive engineering report using TrueForge's artifact-building capabilities.

The report begins with:

## What Does This Grader Actually Do?

Before presenting failures, Eval Insights explains:

- grader purpose
- grader type
- scoring mechanism
- inputs
- reference data
- meaning of high/low scores
- observable system layers
- limitations

This ensures developers understand the evaluator before interpreting its scores.

---

## Failure Category Distribution

The report then shows:

```text
Failure Category
Root Cause
Trace Count
% of Failures
Confidence
Severity
```

Failure categories are clickable.

---

## Category → Trace Drill-Down

Clicking a category reveals all analyzed traces belonging to that failure mode.

Example:

```text
Valid Paraphrase Penalized
11 traces · 61% of failures

Trace dad-014     Score 0.00
Trace dad-021     Score 0.33
Trace dad-028     Score 0.33
...
```

---

## Trace Details

Each trace can show:

- user input
- agent response
- grader score
- grader feedback
- grader expectation
- diagnosed failure mode
- runtime evidence
- code evidence
- responsible system layer
- whether behavior exists elsewhere
- whether the grader can observe it
- confidence
- recommended fix

---

# 🔗 Langfuse Trace Drill-Down

When a verified Langfuse trace URL is available, the report provides:

```text
Open in Langfuse
```

This lets an engineer move directly from:

```text
aggregate diagnosis
→ failure category
→ individual trace
→ original runtime evidence
```

The investigator never fabricates trace URLs.

---

# 🩺 Grader Health vs Agent Health

Eval Insights deliberately separates:

## Grader Health

- implementation correctness
- design alignment
- rubric quality
- observability
- reference-data quality
- consistency

from:

## Agent Health

- prompt quality
- tool selection
- tool/data handling
- reasoning behavior
- fallback behavior
- error handling

This prevents engineers from changing a healthy agent simply because the evaluator is poorly designed.

---

# 🛠️ Recommended Actions

Recommendations identify the correct change target.

Examples:

```text
FIX AGENT
FIX GRADER
FIX DATASET
FIX TOOL
FIX INFRASTRUCTURE
NO CHANGE REQUIRED
```

Each recommendation includes:

- priority
- target
- suggested change
- supporting evidence
- confidence
- relevant source file when available

---

# 📈 Scaling to Larger Evaluation Sets

Small trace sets can be investigated directly.

For larger evaluation workloads, Eval Insights can use TrueForge subagents.

```text
                   Lead Investigator
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
          Trace         Trace        Trace
       Investigator  Investigator  Investigator
          Batch A       Batch B       Batch C
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

Potential specialized investigators include:

### Trace Investigator

Analyzes batches of failing and passing traces.

### Grader Auditor

Investigates:

- grader implementation
- rubric
- scoring logic
- reference data
- observability

### Code Investigator

Investigates:

- agent prompt
- tools
- middleware
- routing
- runtime behavior
- code paths

Subagents provide evidence.

The parent Eval Investigator remains responsible for:

- reconciling findings
- validating aggregate claims
- clustering failure modes
- determining root causes
- producing the final report

---

# 🧪 Example Investigation

Suppose:

```text
Evaluator: keyword_overlap
```

Langfuse shows several low scores.

An engineer asks:

```text
Investigate why keyword_overlap is getting low scores.
```

Eval Insights investigates.

It discovers that the grader uses deterministic expected-keyword matching.

A trace contains:

```text
User:
"Book my train ticket to Munich for tomorrow."
```

The agent refuses to perform the unsupported action but provides useful guidance.

The grader returns:

```text
Score: 0
Matched 0 of 3 expected keywords.
```

The investigator searches the repository and discovers that the dataset expects wording such as:

```text
outside
scope
phone help
```

The grader implementation correctly calculates keyword coverage.

Therefore:

```text
GRADER_IMPLEMENTATION: HEALTHY

GRADER_DESIGN: PROBLEM

REFERENCE DATA: BRITTLE

AGENT: MOSTLY HEALTHY
```

The recommended action is **not**:

```text
Modify the agent to repeat the expected keywords.
```

Instead:

```text
Replace/supplement lexical matching with semantic evaluation
and improve the reference criteria.
```

This is the distinction Eval Insights is designed to discover.

---

# 🆚 Traditional Eval Debugging vs Eval Insights

### Traditional workflow

```text
See low score
    ↓
Open trace
    ↓
Read grader feedback
    ↓
Open another trace
    ↓
Compare responses
    ↓
Find grader code
    ↓
Find dataset
    ↓
Find agent prompt
    ↓
Inspect tools
    ↓
Guess root cause
    ↓
Decide what to change
```

### Eval Insights

```text
"Investigate this grader."
           ↓
      Eval Insights
           ↓
   Interactive report
           ↓
   Root cause + evidence
           ↓
      Recommended fix
```

---

# 🧰 Tech Stack

| Component | Purpose |
|---|---|
| **TrueForge** | Agent harness and autonomous runtime |
| **Eval Investigation Skill** | Investigation methodology |
| **Langfuse** | Agent traces, grader scores, runtime evidence |
| **Langfuse MCP** | Agent access to evaluation data |
| **GitHub MCP** | Source-code and implementation evidence |
| **OpenAI** | Reasoning/model inference |
| **TrueForge Web Artifact Skill** | Interactive investigation report |

---

# 🏆 Hackathon Alignment

This project was built for the **Agent Harness Hackathon**.

The primary target is:

## Best Use of the Agent Harness

The project intentionally makes TrueForge central to the workflow.

The harness is responsible for:

```text
reason
→ call Langfuse
→ inspect evidence
→ form hypotheses
→ call GitHub
→ inspect implementation
→ follow evidence
→ investigate further
→ synthesize root cause
→ generate artifact
```

TrueForge is therefore not sitting underneath a thin chatbot wrapper.

The harness performs the actual investigation.

---

# 🚀 Demo

The demo begins with one instruction:

```text
Investigate why keyword_overlap is getting low scores.
```

No further investigation instructions are required.

The demo shows:

1. TrueForge receiving the investigation goal
2. Langfuse tool calls
3. GitHub/source-code investigation
4. autonomous hypothesis-driven analysis
5. generated interactive report
6. failure-category distribution
7. category → trace drill-down
8. direct Langfuse trace access
9. root-cause diagnosis
10. recommended fix

---

# 📁 Repository Structure

```text
eval-insights-agent/
│
├── README.md
├── SKILL.md
│
├── prompts/
│   └── system-prompt.md
│
├── examples/
│   ├── sample-report.html
│   └── sample-output.json
│
├── docs/
│   ├── architecture.md
│   └── demo-script.md
│
├── assets/
│   └── screenshots/
│
└── LICENSE
```

---

# 🔮 Future Work

Potential next steps include:

### Automatic Regression Case Generation

Convert discovered production failures into new evaluation cases.

```text
Failure discovered
      ↓
Generate regression case
      ↓
Human approval
      ↓
Add to evaluation dataset
```

### Automatic Fix Generation

Generate evidence-backed changes to:

- agent prompts
- grader rubrics
- tool descriptions
- reference data

### Before / After Validation

Apply a proposed fix in a safe environment and rerun affected evaluations.

```text
Before: 0.64
After:  0.86
```

### Evaluation Blind-Spot Detection

Analyze:

```text
agent capabilities
+
existing graders
```

and identify important behaviors that currently have no evaluation coverage.

### Larger-Scale Investigation

Use hierarchical TrueForge subagents to investigate hundreds or thousands of traces efficiently.

---

# 🎯 Vision

Evaluation infrastructure has become very good at answering:

> **What score did my agent get?**

But engineers ultimately need to know:

> **Why?**

Eval Insights turns evaluation telemetry into an autonomous engineering investigation.

Instead of manually moving between scores, traces, prompts, graders, datasets, and source code, engineers can ask:

```text
Investigate this evaluator.
```

and receive an evidence-backed explanation of:

- what failed
- why it failed
- how common it is
- where the problem originates
- whether the agent is actually responsible
- whether the evaluator itself is wrong
- what should be changed first

---

# Eval Insights

### From eval score → root cause.

**Langfuse tells you what failed.
Eval Insights investigates why.**

# Demo Script

A walkthrough for demoing Eval Insights live (~4-6 minutes).

## Setup

- A Langfuse project connected via the Langfuse MCP, with at least one
  grader (e.g. `keyword_overlap`) showing a run with mixed scores.
- A GitHub repository connected via the GitHub MCP containing the grader
  implementation, the evaluated agent's prompt, its tools, and the dataset.
- `examples/sample-output.json` and `examples/sample-report.html` open as a
  canned fallback in case live data isn't available.

## Script

1. **Set the scene (30s)**
   - "Eval platforms like Langfuse tell you a grader scored 0.64. They
     don't tell you why. Today an engineer has to manually hop between
     traces, grader code, the dataset, and the agent prompt to figure that
     out. Eval Insights does that investigation autonomously."

2. **Give the single instruction (15s)**
   - Type the one instruction the agent needs:
     ```text
     Investigate why keyword_overlap is getting low scores.
     ```
   - Emphasize: no further investigation instructions are provided.

3. **Narrate the investigation as it runs (90-120s)**
   - TrueForge receiving the investigation goal and loading the Eval
     Investigation Skill.
   - Langfuse MCP calls: pulling scores, pulling low-scoring traces,
     pulling a few passing traces for comparison.
   - The agent forming competing hypotheses out loud (grader bug? grader
     design? agent behavior? dataset?).
   - GitHub MCP calls: opening the grader implementation, then the
     dataset, then (if needed) the agent's system prompt.
   - The moment it resolves the hypothesis — e.g. "the grader implementation
     is correct, but it's doing exact-keyword matching against an
     overly-specific reference answer."

4. **Walk through the generated report (90-120s)**
   - Open the interactive report and go top to bottom:
     - **Grader explanation** — what `keyword_overlap` actually measures.
     - **Failure category distribution** — click into the largest cluster
       (e.g. "Valid paraphrase penalized," 61% of failures).
     - **Trace drill-down** — open one trace, show the agent's actually
       reasonable response next to the grader's score of 0 and its
       feedback.
     - **Langfuse link** — click through to the real trace in Langfuse.
     - **Grader health vs. agent health** — point out the agent is mostly
       healthy; the grader design is the problem.
     - **Recommendations** — read the top one aloud, note it's tagged
       `FIX GRADER`, not `FIX AGENT`.

5. **Land the point (20-30s)**
   - "The naive fix would have been to change the agent's prompt to repeat
     magic keywords — that optimizes the agent for the benchmark, not for
     users. Eval Insights caught that the evaluator itself was the
     problem, with the evidence to prove it, in one instruction instead of
     twenty minutes of manual digging."

## Fallback

If live Langfuse/GitHub access isn't available during the demo, fall back
to `examples/sample-output.json` and `examples/sample-report.html` — they
represent a realistic finished investigation (the same `keyword_overlap` /
train-ticket scenario) and support the same narration.

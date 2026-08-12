# Introspection Self-Improving Agent MVP

## Agent-Readable Research Context, Architecture, Experimental Protocol, and Implementation Direction

**Status:** MVP design\
**Date:** 2026-08-12\
**Primary goal:** Demonstrate a genuinely self-improving agent harness
using Introspection's native operational and improvement primitives,
with an external benchmark providing an immutable objective.

------------------------------------------------------------------------

## 0. Executive Summary

We want to demonstrate **self-improvement through the Introspection
framework**, not merely build a generic self-improvement loop and host
the target agent on Introspection.

The MVP uses:

-   **τ-bench / τ-Knowledge `banking_knowledge`** as the external task
    environment and immutable objective evaluator.
-   An intentionally simple **target agent implemented as an
    Introspection recipe**.
-   **Introspection** as the evidence and execution substrate: tasks,
    conversations, traces, tool calls, observations, patterns, metrics,
    judgements, and runtime lineage.
-   **Claude Code + the Introspection plugin** as the **Improvement
    Orchestrator**.
-   The plugin's **`operate`** skill for evidence gathering, signal
    discovery, and diagnosis.
-   The plugin's **`improve`** skill for hypothesis-driven modifications
    to the target agent's repository-owned harness.
-   Repeated benchmark rounds to determine whether the resulting harness
    changes improve performance on unseen tasks.

The key architectural insight is:

> **Do not predefine the useful diagnostic signals for the orchestrator.
> Let Claude discover them from benchmark outcomes plus Introspection
> execution evidence.**

The orchestrator therefore has two core responsibilities:

1.  **Learn from execution:** inspect evidence, discover recurring and
    actionable signals, compare failures against successful controls,
    identify the earliest meaningful divergence, and formulate a causal
    hypothesis.
2.  **Act on what it learned:** propose and eventually implement the
    smallest coherent harness change expected to improve the target
    agent, then validate the prediction experimentally.

The immutable τ evaluator remains the ultimate source of truth.
Diagnostic signals, observations, patterns, and even new evals/judges
may evolve, but the objective benchmark cannot.

The core loop is:

``` text
τ tasks
  ↓
Target Agent H_n
  ↓
Introspection execution evidence + τ outcome
  ↓
Claude Code + Introspection plugin
  ├─ operate  → discover evidence/signals/diagnosis
  └─ improve  → hypothesis + harness mutation
  ↓
Candidate H_(n+1)
  ↓
Validation
  ↓
accept / reject
  ↓
repeat
```

The research question is:

> **Can an LLM Improvement Orchestrator use Introspection's operational
> evidence to autonomously discover actionable failure signals,
> formulate hypotheses about an agent's behavior, and evolve its harness
> such that performance improves on unseen τ-Knowledge tasks?**

------------------------------------------------------------------------

# 1. What We Are Trying to Demonstrate

The project should demonstrate more than:

> Claude Code can edit an agent.

It should demonstrate:

> **Given an external objective and empirical execution evidence, an LLM
> orchestrator can discover what aspects of an agent are limiting
> performance, formulate an intervention, modify the agent harness
> through Introspection's repository workflow, and produce measurable
> improvement on unseen tasks.**

This distinction matters.

A weak demonstration would be:

``` text
benchmark failure
  ↓
human-defined label: "retrieval failure"
  ↓
Claude told to improve retrieval
  ↓
Claude edits prompt
```

The intended demonstration is:

``` text
benchmark failure
  ↓
raw execution evidence
  ↓
Claude investigates
  ↓
Claude discovers a recurring phenomenon
  ↓
Claude tests whether that phenomenon plausibly explains failures
  ↓
Claude formulates a hypothesis
  ↓
Claude chooses the owning harness layer
  ↓
Claude proposes a minimal intervention
  ↓
candidate is evaluated
  ↓
objective score determines whether intervention worked
```

The **discovery of the useful signal is part of self-improvement**.

------------------------------------------------------------------------

# 2. Core Design Principles

## 2.1 Introspection must be structurally essential

The target agent is an Introspection recipe.

The Improvement Orchestrator uses Introspection's own plugin and
operational surfaces to inspect the agent and propose improvements.

The project is therefore not:

``` text
generic self-improver
  ↓
Introspection used as hosting
```

It is:

``` text
Introspection Agent
  ↓
Introspection evidence
  ↓
Introspection operate/improve workflow
  ↓
Introspection recipe change
  ↓
new Introspection Agent version
```

τ-bench sits outside this loop as the independent reality check.

## 2.2 Objective and diagnostics must remain separate

There are two measurement layers.

### Immutable objective

``` text
τ-bench / τ-Knowledge reward
```

This answers:

> Did the agent actually perform the task correctly?

The orchestrator must never modify:

-   benchmark tasks;
-   gold answers/state;
-   evaluator;
-   reward aggregation;
-   held-out split;
-   benchmark adapter in a way that changes semantics.

### Evolvable diagnostics

Examples:

-   Introspection traces;
-   tool-call evidence;
-   observations;
-   patterns;
-   aggregate metrics;
-   judge outputs;
-   custom evals;
-   discovered failure clusters.

These answer:

> What might explain the objective outcome?

The orchestrator may learn to use or eventually create/refine diagnostic
instrumentation, but diagnostics can never replace the external
objective.

Formally:

\[ `\text{Objective}`{=tex} `\neq `{=tex}`\text{Diagnostics}`{=tex} \]

and:

\[ `\text{Success}`{=tex}(H) =
`\text{immutable benchmark evaluation of }`{=tex} H \]

## 2.3 Open-code evidence before imposing a taxonomy

Do **not** begin with a hand-designed failure ontology such as:

``` text
retrieval_failure
policy_failure
tool_failure
planning_failure
communication_failure
```

These may ultimately emerge, but humans should not hand them to the
Improvement Orchestrator as its initial gradient.

The current Introspection `improve` skill explicitly instructs the agent
to **open-code the evidence before imposing a taxonomy**.

This principle is central to the MVP.

## 2.4 Keep the model fixed

For the MVP, improvement means **harness improvement**, not model
improvement.

Freeze:

-   target model;
-   model version/provider;
-   sampling configuration;
-   benchmark;
-   task splits;
-   relevant execution budgets;
-   evaluator.

Allow the orchestrator to modify the harness.

This gives the experiment a clean interpretation:

\[ `\Delta `{=tex}`\text{performance}`{=tex}
`\approx `{=tex}f(`\Delta `{=tex}`\text{harness}`{=tex}) \]

rather than:

\[ f(`\Delta `{=tex}`\text{harness}`{=tex},
`\Delta `{=tex}`\text{model}`{=tex},
`\Delta `{=tex}`\text{benchmark}`{=tex},
`\Delta `{=tex}`\text{budget}`{=tex}) \]

## 2.5 Prefer minimal, hypothesis-driven changes

Do not encourage complete rewrites after every failed round.

Preferred process:

``` text
Evidence
  ↓
Signal
  ↓
Hypothesis
  ↓
Prediction
  ↓
Small coherent mutation
  ↓
Validation
  ↓
Accept / reject
```

This improves causal interpretability and makes the evolutionary history
of the harness useful research data.

------------------------------------------------------------------------

# 3. The Four System Roles

## 3.1 τ-bench / τ-Knowledge: Task Oracle

Responsibilities:

-   provide realistic conversational tasks;
-   provide the banking environment;
-   provide knowledge documents;
-   provide transactional tools/state;
-   provide objective evaluation;
-   remain immutable from the Improvement Orchestrator's perspective.

It does **not** diagnose the target agent for Claude.

Conceptually, τ provides:

``` text
task
environment
objective outcome
```

not:

``` text
failure cause
recommended fix
```

## 3.2 Target Agent: Subject Being Improved

The target is an Introspection recipe (H_n).

It should begin intentionally simple.

Initial responsibilities:

-   interact with the simulated user;
-   retrieve relevant knowledge;
-   reason over policies/procedures;
-   use banking tools;
-   communicate the result.

Initially avoid sophisticated scaffolding such as:

-   multi-query retrieval;
-   reranking;
-   explicit planning frameworks;
-   specialized policy compilers;
-   verification subagents;
-   elaborate retries;
-   large collections of hand-authored skills.

We want sufficient headroom for the Improvement Orchestrator to discover
useful structure.

## 3.3 Introspection: Evidence and Execution Substrate

Introspection supplies the empirical surface through which the
orchestrator understands what happened.

Relevant surfaces include:

-   tasks/runs;
-   conversations;
-   traces;
-   model calls;
-   tool calls and results;
-   observations;
-   patterns;
-   metrics;
-   costs;
-   runtime/version information;
-   feedback;
-   judgements;
-   experiments where justified;
-   repository/runtime lineage.

Introspection also supplies the repository mechanism that allows an
agent recipe to be changed through ordinary Git/PR workflows.

## 3.4 Claude Code + Introspection Plugin: Improvement Orchestrator

This is the learning/control component.

**There is no additional orchestrator agent to implement.** Throughout this document, "Improvement Orchestrator" means **Claude Code operating with the Introspection plugin**. The repository's `contract/` directory constrains this orchestrator; it does not implement it.

Its contract is approximately:

\[ O(E_n, H_n) `\rightarrow `{=tex}(S_n, `\Delta `{=tex}H_n) \]

where:

-   (E_n): execution evidence from the current generation;
-   (H_n): current target harness;
-   (S_n): discovered signal/diagnosis/hypothesis;
-   (`\Delta `{=tex}H_n): proposed harness modification.

Internally:

\[ O = O\_{`\text{operate}`{=tex}} + O\_{`\text{improve}`{=tex}} \]

------------------------------------------------------------------------

# 4. Why `operate` and `improve` Are the Correct Introspection Primitives

## 4.1 `operate`: empirical investigation

The current Introspection plugin describes `operate` as the workflow for
inspecting/explaining/changing **live Introspection state without
changing the agent recipe**.

Relevant capabilities include investigation of:

-   tasks;
-   failed/stuck/cancelled executions;
-   conversations;
-   traces;
-   observations;
-   patterns;
-   metrics;
-   costs;
-   runtimes;
-   live judge state.

Important behaviors encoded in `operate`:

1.  Start task diagnosis from the task row and execution state.
2.  Only move to conversation evidence after understanding the
    task-level result.
3.  Inspect tool failures explicitly rather than assuming a conversation
    succeeded because model calls succeeded.
4.  Distinguish individual evidence from population-level prevalence.
5.  Use aggregate telemetry before claiming that a pattern is common or
    rare.
6.  Treat absence of an asynchronous pattern as insufficient proof that
    a problem does not exist.
7.  Hand behavioral recipe changes to `improve`.

For this project:

> **`operate` is the orchestrator's empirical interface.**

It answers:

-   What happened?
-   Where did it happen?
-   How frequently?
-   Which failures share a behavior?
-   How do failures differ from successful controls?
-   What evidence supports or falsifies a suspected cause?

## 4.2 `improve`: intervention

The current plugin describes `improve` as the workflow for proposing or
implementing repository-owned changes to an existing Introspection agent
recipe.

Its scope includes:

-   behavior;
-   prompts;
-   tools;
-   configuration;
-   tests;
-   evals;
-   judge definitions.

Its methodology is unusually well aligned with this experiment.

It directs the agent to:

-   begin from evidence;
-   verify that evidence and local/deployed code refer to the same
    target;
-   inspect recurring patterns and exact supporting conversations;
-   include controls;
-   seek falsifying as well as supporting evidence;
-   open-code evidence before imposing a taxonomy;
-   identify the earliest meaningful divergence;
-   identify the actual owning layer before choosing a remedy;
-   establish an unchanged baseline;
-   change one coherent mechanism at a time;
-   freeze relevant comparison configuration;
-   rerun affected cases and non-regression controls;
-   inspect traces behind score changes;
-   avoid creating an eval or experiment for every failure.

For this project:

> **`improve` is the orchestrator's intervention interface.**

## 4.3 Handoff boundary

The conceptual handoff is:

``` text
OPERATE
inspect → compare → aggregate → discover → diagnose
                         │
                         ▼
                    hypothesis
                         │
                         ▼
IMPROVE
identify owner → propose mechanism → mutate → prove
```

The distinction should remain explicit even if a single Claude Code
session drives both skills.

------------------------------------------------------------------------

# 5. Why τ-Knowledge `banking_knowledge`

The preferred MVP benchmark is the text-based `banking_knowledge` domain
from τ-bench / τ-Knowledge.

Current τ-bench documentation describes the domain as containing:

-   **97 tasks**;
-   **698 policy/procedure documents**;
-   account-management, credit-card, dispute, and transfer workflows;
-   configurable retrieval;
-   transactional tools plus knowledge retrieval.

The retrieval framework supports multiple mechanisms including:

-   BM25;
-   cosine-similarity retrieval;
-   grep;
-   embedding-backed retrieval;
-   reranking;
-   agentic terminal/shell retrieval.

This is valuable because target-agent failures can emerge across several
harness dimensions:

``` text
user intent
   ↓
knowledge retrieval
   ↓
cross-document reasoning
   ↓
policy/procedure interpretation
   ↓
planning
   ↓
tool selection
   ↓
tool arguments
   ↓
state transition
   ↓
communication
```

Crucially, we do **not** tell the Improvement Orchestrator that these
are the failure categories. They are only examples of phenomena it may
discover.

τ-Knowledge is preferable to a simple classification or QA benchmark
because the agent harness has real architectural surface to improve.

------------------------------------------------------------------------

# 6. τ-bench Versioning Requirement

Use a current τ-bench version and pin it exactly.

At minimum:

``` text
tau2-bench >= 1.0.1
```

Preferably record an exact Git commit.

Reason: τ-bench v1.0.1 changed `banking_knowledge` grading and task
data. The project explicitly warns that results from versions before and
after v1.0.1 are not directly comparable.

A self-improvement curve is meaningless if the evaluator changes during
the experiment.

Record for every run:

``` yaml
benchmark:
  repository: sierra-research/tau2-bench
  version: ">=1.0.1"
  commit: "<exact SHA>"
  domain: banking_knowledge
```

------------------------------------------------------------------------

# 7. Why Not Use Harbor as the First Benchmark?

Harbor is highly relevant but is not the preferred first task
environment.

Harbor is a framework for:

-   evaluating arbitrary agents;
-   creating benchmarks/environments;
-   running isolated environments at scale;
-   generating rollouts for RL/optimization.

It becomes valuable later as a **benchmark portability layer**.

A terminal/SWE benchmark has a much larger failure surface:

-   repository navigation;
-   shell interaction;
-   planning;
-   coding;
-   testing;
-   dependency management;
-   debugging;
-   environment issues.

That makes early causal interpretation harder.

Recommended progression:

``` text
MVP:
Introspection + τ-Knowledge

Later:
same Improvement Orchestrator
        │
        ├── τ-Knowledge
        ├── Harbor / Terminal tasks
        └── custom benchmarks
```

A later research question becomes:

> Does the same Introspection-based improvement process generalize
> across agent domains?

------------------------------------------------------------------------

# 8. What We Borrow from SIA

SIA provides an important conceptual reference.

SIA separates roles around:

-   meta/feedback agent;
-   target/task agent;
-   benchmark feedback;
-   successive generations.

It explores both:

-   harness updates;
-   weight updates.

For this MVP we deliberately choose only:

``` text
HARNESS UPDATES
```

and hold weights/model fixed.

What we borrow:

1.  **Generational improvement.**
2.  **External benchmark signal.**
3.  **Separation between target agent and improvement logic.**
4.  **Preservation of per-generation artifacts.**
5.  **Evaluation of whether improvements generalize rather than merely
    fit observed cases.**

What we do **not** initially borrow:

-   weight updates;
-   RL/post-training;
-   simultaneous model and harness evolution.

This keeps the result attributable to the Introspection
harness-improvement process.

------------------------------------------------------------------------

# 9. What We Borrow from Prime Agent

Prime Agent is useful conceptually because its refinement philosophy
favors incremental changes grounded in trajectory evidence rather than
uncontrolled rewrites.

The relevant principle for this MVP is:

> **Prefer the smallest harness mutation that follows from the evidence
> and has a testable predicted effect.**

The orchestrator should not respond to:

``` text
score = 42%
```

with:

``` text
rewrite the entire agent
```

It should instead produce reasoning of the form:

``` text
Observation:
A recurring behavior appears in failed trajectories.

Control:
Successful trajectories differ at a specific point.

Hypothesis:
Mechanism X plausibly explains the divergence.

Intervention:
Change one harness mechanism.

Prediction:
Specific cases should improve without regressions in controls.

Validation:
Run unchanged baseline and candidate.

Decision:
Accept or reject.
```

------------------------------------------------------------------------

# 10. What We Borrow from SquareDiff

SquareDiff's public philosophy is highly aligned with the project:

``` text
define → improve → run → evolve
```

Its platform describes autonomous experimentation across prompts,
models, and tools using evaluation performance and traces.

The main connection is **autonomous harness experimentation**.

Our MVP differs by deliberately emphasizing:

-   an external research benchmark;
-   a fixed model;
-   interpretable generational changes;
-   open evidence-to-hypothesis records;
-   Introspection as the native evidence and modification substrate.

------------------------------------------------------------------------

# 11. The Final MVP Architecture

``` text
                    IMMUTABLE OUTER REALITY
              ┌───────────────────────────────┐
              │           τ-BENCH             │
              │                               │
              │ banking_knowledge tasks       │
              │ environment / user simulator  │
              │ objective evaluator           │
              └───────────────┬───────────────┘
                              │
                              ▼
                   ┌─────────────────────┐
                   │    TARGET AGENT     │
                   │                     │
                   │ Introspection H_n   │
                   └──────────┬──────────┘
                              │
                           executes
                              │
                              ▼
             ┌──────────────────────────────────┐
             │          INTROSPECTION           │
             │                                  │
             │ tasks / conversations / traces   │
             │ tool calls / results             │
             │ observations / patterns          │
             │ metrics / feedback / judgements  │
             │ runtime + repository lineage     │
             └────────────────┬─────────────────┘
                              │
                           evidence
                              │
                              ▼
       ┌────────────────────────────────────────────┐
       │          IMPROVEMENT ORCHESTRATOR          │
       │          Claude Code + Plugin              │
       │                                            │
       │  ┌──────────────────────────────────────┐  │
       │  │ OPERATE                              │  │
       │  │                                      │  │
       │  │ inspect                              │  │
       │  │ compare failures / successes         │  │
       │  │ measure prevalence                   │  │
       │  │ open-code evidence                   │  │
       │  │ discover actionable signal           │  │
       │  │ identify earliest divergence         │  │
       │  └───────────────────┬──────────────────┘  │
       │                      │                     │
       │               learning record              │
       │                      │                     │
       │                      ▼                     │
       │  ┌──────────────────────────────────────┐  │
       │  │ IMPROVE                              │  │
       │  │                                      │  │
       │  │ determine owning layer               │  │
       │  │ formulate causal hypothesis          │  │
       │  │ predict expected effect              │  │
       │  │ propose minimal mutation             │  │
       │  │ baseline → candidate → validation    │  │
       │  └───────────────────┬──────────────────┘  │
       └──────────────────────┼─────────────────────┘
                              │
                              ▼
                            H_n+1
                              │
                              └──────────► next round
```

------------------------------------------------------------------------

# 12. Experimental Dataset Split

Do not use a simple train/test split if we want the orchestrator to
investigate failures repeatedly without contaminating the final
evaluation.

Use three sets.

``` text
                     τ task pool
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
      DISCOVERY      VALIDATION        TEST
          D              V              T
```

Suggested initial sizes depend on implementation cost, but a reasonable
MVP could begin around:

``` text
Discovery:   25–35 tasks
Validation:  10–20 tasks
Test:        15–25 tasks
```

The exact split should preserve task diversity.

## 12.1 Discovery set

Claude may inspect:

-   task outcome;
-   conversation;
-   trace;
-   model calls where available;
-   retrieval/tool calls;
-   observations;
-   patterns;
-   metrics;
-   relevant diagnostic judgements.

This is the dataset from which the orchestrator learns.

## 12.2 Validation set

Used to decide whether a candidate change generalizes beyond the cases
that motivated it.

Prefer limiting information returned to the orchestrator.

At minimum provide aggregate/objective outcomes.

If candidate diagnosis requires additional evidence, define a controlled
policy for exposing validation failures rather than silently turning
validation into more discovery data.

## 12.3 Test set

The Improvement Orchestrator must not inspect test tasks or trajectories
during optimization.

Use only at predetermined checkpoints or final evaluation.

This is the strongest evidence that the harness genuinely improved.

------------------------------------------------------------------------

# 13. Baseline Target Agent

Generation 0 should be intentionally simple but legitimate.

Conceptual architecture:

``` text
simulated user
     ↓
target LLM
     ↓
knowledge-search capability
     ↓
banking documents
     ↓
target LLM
     ↓
banking transactional tools
     ↓
response / state
```

The baseline should have only enough instruction to perform the task:

-   act as a banking support agent;
-   use available knowledge when needed;
-   follow policies;
-   use available banking tools when appropriate;
-   communicate results accurately.

Avoid prematurely giving it:

-   query decomposition;
-   synonym expansion;
-   reranking;
-   explicit retrieve-plan-execute loops;
-   specialized policy extraction;
-   self-verification;
-   multi-agent decomposition;
-   learned failure-specific rules.

Those are candidate structures the Improvement Orchestrator may
discover.

------------------------------------------------------------------------

# 14. Mutable vs Immutable Surfaces

## 14.1 Immutable during an experiment

The Improvement Orchestrator must not modify:

``` text
target model
target model version
sampling configuration
τ benchmark version
τ evaluator
τ task definitions
dataset split
gold states/answers
objective reward aggregation
benchmark semantic adapter
held-out/test data
comparison budgets
```

## 14.2 Mutable target-agent harness

Potentially mutable:

``` text
system prompt
agent instructions
skills
tool descriptions
retrieval behavior
retrieval implementation
planning/orchestration logic
retry logic
context management
verification logic
tests
diagnostic eval definitions
diagnostic judge definitions
```

The exact permission envelope should be explicit.

## 14.3 Important distinction: diagnostic evolution

The orchestrator may discover that a recurring behavioral risk deserves
permanent instrumentation.

For example:

``` text
G0:
τ reward only

experience:
Claude discovers recurring retrieval mismatch

G1:
target harness receives a retrieval intervention

possibly:
a retrieval-quality diagnostic eval is introduced
```

This is allowed because the τ objective remains unchanged.

The system can therefore co-evolve:

\[ (H_n, E_n) \]

where:

-   (H_n): target harness;
-   (E_n): diagnostic/instrumentation model.

But only (H_n)'s performance under the immutable external objective
determines success.

------------------------------------------------------------------------

# 15. One Improvement Generation

A complete generation should be treated as an empirical cycle.

## Phase A --- Execute

Run the current target harness (H_n) on the selected discovery cases.

Capture:

-   τ objective results;
-   Introspection task identifiers;
-   conversations;
-   traces;
-   retrieval/tool activity;
-   relevant metrics;
-   observations/patterns.

## Phase B --- Operate

Claude invokes the `operate` workflow.

Required behavior:

1.  Resolve the exact target runtime/version.
2.  Inspect objective failures.
3.  Inspect corresponding Introspection task rows.
4.  Inspect conversations/traces/tool evidence.
5.  Inspect successful controls.
6.  Use aggregate telemetry to estimate prevalence.
7.  Open-code observed phenomena.
8.  Search for the earliest meaningful divergence.
9.  Distinguish correlation from plausible cause.
10. Seek counterexamples/falsifying evidence.

Output: one or more candidate actionable signals.

Example:

``` text
Observed:
Failed cases frequently retrieve policy documents that match
the user's literal vocabulary but not the operational banking
concept required by the task.

Controls:
Successful cases often contain vocabulary overlap with the
correct policy document.

Candidate signal:
Surface-vocabulary mismatch between user language and policy
terminology may be suppressing relevant-document retrieval.
```

Notice that the human never supplied the label "retrieval failure."

## Phase C --- Hypothesis

The orchestrator converts evidence into a testable hypothesis.

Example:

``` text
If the target agent reformulates user language into likely
banking-policy terminology before retrieval, relevant-document
recall should improve on the affected class of tasks without
degrading unrelated tasks.
```

The hypothesis should include:

-   evidence;
-   counterevidence;
-   owning layer;
-   expected affected cases;
-   expected non-regressions;
-   confidence;
-   predicted measurable effect.

## Phase D --- Improve

Hand off to `improve`.

The orchestrator determines the narrowest appropriate mutation.

Examples that may emerge:

-   prompt clarification;
-   new skill;
-   retrieval query reformulation;
-   reranking;
-   improved tool description;
-   prerequisite validation;
-   plan-before-action step;
-   post-action verification;
-   deterministic test;
-   justified diagnostic eval.

Do not prescribe these categories to the orchestrator upfront.

## Phase E --- Baseline and Candidate

Preserve an unchanged baseline.

Then create candidate (H'\_n).

Freeze comparison variables.

Run:

``` text
baseline H_n
vs.
candidate H'_n
```

on the same relevant evaluation protocol.

## Phase F --- Validation

Evaluate candidate on validation tasks.

Possible result:

``` text
candidate improves:
accept → H_(n+1)

candidate does not improve:
reject/revert → retain H_n
```

Inspect traces behind score changes. A higher aggregate score alone is
not sufficient evidence that the intended mechanism improved.

## Phase G --- Record

Persist the complete learning record.

------------------------------------------------------------------------

# 16. Learning Record Schema

Every attempted mutation should produce a machine-readable artifact.

Example:

``` yaml
generation: 3
candidate: query-reformulation-v1

target:
  recipe_commit: "<sha>"
  runtime_version: "<id>"
  model: "<fixed-model>"

objective_before:
  discovery_pass1: 0.47
  validation_pass1: 0.43

evidence_examined:
  failed_tasks: 16
  successful_controls: 8
  task_ids:
    - ...
  conversations:
    - ...

discovered_signal:
  description: >
    Failed cases frequently retrieve documents matching
    surface terminology while missing the operational
    banking concept required by the policy.
  prevalence: "<measured, not guessed>"
  supporting_cases:
    - ...
  counterexamples:
    - ...

earliest_divergence:
  layer: retrieval
  description: >
    ...

hypothesis:
  statement: >
    Reformulating user language into domain-policy terminology
    before retrieval should increase relevant-document recall.
  confidence: "<calibrated qualitative or numeric value>"
  expected_effect:
    - "improve affected discovery cases"
    - "improve validation pass rate"
  expected_non_regression:
    - "no increase in invalid banking tool calls"

proposed_change:
  owner: target-agent-harness
  mechanism: "<description>"
  files:
    - ...

comparison:
  frozen:
    - model
    - model_version
    - benchmark_commit
    - cases
    - scorer
    - execution_budget

results:
  discovery_before: ...
  discovery_after: ...
  validation_before: ...
  validation_after: ...
  cost_before: ...
  cost_after: ...

trace_review:
  intended_mechanism_observed: true
  regressions_observed: []

decision: accept
reason: >
  ...
```

This artifact is part of the research output, not incidental logging.

------------------------------------------------------------------------

# 17. Metrics

## 17.1 Primary metric

Start with the native τ objective.

Primary:

``` text
pass^1 / task success rate
```

Do not initially create a complicated composite objective.

## 17.2 Reliability

Where budget allows, repeated trials can measure reliability rather than
single lucky successes.

Relevant concepts from τ-bench include pass\^k-style reliability.

## 17.3 Efficiency metrics

Track but do not necessarily optimize initially:

``` text
tokens/task
model calls/task
retrieval calls/task
tool calls/task
latency
cost/task
conversation turns
```

## 17.4 Improvement-process metrics

These are especially interesting for this project:

``` text
candidate acceptance rate
validation gain per accepted mutation
regression rate
cost per accepted improvement
generations to improvement
number of files/mechanisms changed
diagnostic signals discovered
diagnostic signals that led to successful interventions
```

A useful meta-measure is:

\[ P(`\Delta `{=tex}R \> 0 `\mid `{=tex}S_n
`\rightarrow `{=tex}`\Delta `{=tex}H_n) \]

Informally:

> How often does a discovered signal lead to an intervention that
> actually improves objective performance?

------------------------------------------------------------------------

# 18. What Counts as Self-Improvement Here?

The human supplies:

``` text
goal
benchmark
environment
permission boundaries
```

The Improvement Orchestrator discovers:

``` text
failure structure
useful signals
causal hypotheses
owning harness layer
intervention
validation evidence
```

This is stronger than a system in which humans label every failure and
tell the meta-agent which component to optimize.

The target agent itself does not necessarily execute the improvement
reasoning. The self-improving **system** consists of:

``` text
Target Agent
+
Introspection evidence substrate
+
Improvement Orchestrator
+
mutable target recipe
+
objective feedback loop
```

The critical property is closure:

\[ `\text{experience}`{=tex} `\rightarrow`{=tex}
`\text{diagnosis}`{=tex} `\rightarrow`{=tex}
`\text{harness mutation}`{=tex} `\rightarrow`{=tex}
`\text{new experience}`{=tex} \]

with minimal human specification of the intermediate diagnosis.

------------------------------------------------------------------------

# 19. Repository and Permission Architecture

Prefer strict separation.

Conceptually:

``` text
Repository A — target agent recipe
  orchestrator: read/write through approved workflow

Repository B — benchmark integration
  orchestrator: read-only where possible

Benchmark/evaluator
  orchestrator: immutable

Held-out test data
  orchestrator: inaccessible during optimization
```

Introspection's repository model is directly relevant.

Its documentation states that a recipe is a repository and can be
explicitly granted to an agent. With elevated `contents` and
`pull-requests` permissions, an agent can propose changes to the
prompts, skills, and tools defining its own behavior.

Important security principle:

> Agent repository write access is a code-execution capability.

For an initial research MVP, use PRs and branch protection.

Recommended first workflow:

``` text
Claude discovers improvement
  ↓
Claude proposes recipe diff
  ↓
PR
  ↓
review / controlled acceptance
  ↓
candidate runtime
  ↓
benchmark
```

A later stage can close more of the loop automatically.

------------------------------------------------------------------------

# 20. Human-in-the-Loop vs Fully Closed Loop

The current `improve` skill is explicitly human-in-the-loop and requires
confirmation before repository edits/PR work.

Therefore distinguish:

## MVP-A: research loop with approval

``` text
run
↓
operate
↓
diagnose
↓
improve proposal
↓
human approval
↓
edit / PR
↓
candidate
↓
evaluate
```

This already demonstrates automated signal discovery and automated
improvement design.

## MVP-B: progressively closed loop

After the experimental protocol is trustworthy, automate additional
transitions.

Potential later loop:

``` text
run
↓
operate
↓
diagnose
↓
propose candidate
↓
candidate branch
↓
automatic validation
↓
accept/reject under explicit policy
↓
next generation
```

Do not weaken Introspection's current permission/confirmation boundaries
merely to claim full autonomy.

------------------------------------------------------------------------

# 21. Expected Evolution --- Hypotheses, Not Instructions

We expect that the target harness **might** evolve along dimensions such
as:

``` text
baseline
  │
  ├─ better retrieval discipline
  │
  ├─ query reformulation
  │
  ├─ policy extraction
  │
  ├─ prerequisite checking
  │
  ├─ plan-before-write behavior
  │
  └─ post-action verification
```

But these should **not** be given to the orchestrator as a prescribed
roadmap.

They are researcher hypotheses.

The scientifically interesting result is which mechanisms the
orchestrator actually discovers from evidence.

------------------------------------------------------------------------

# 22. Threats to Validity

## 22.1 Benchmark leakage

Risk:

Claude sees held-out answers/tasks and memorizes them.

Mitigation:

-   strict discovery/validation/test separation;
-   inaccessible final test trajectories;
-   immutable benchmark repo;
-   no target-specific hardcoding.

## 22.2 Judge gaming / Goodharting

Risk:

The orchestrator improves diagnostic scores rather than task
performance.

Mitigation:

-   τ reward is immutable and authoritative;
-   diagnostic evals cannot redefine success;
-   final test uses external evaluator.

## 22.3 Model drift

Risk:

Provider/model changes contaminate generational comparisons.

Mitigation:

-   pin model/provider/version;
-   record exact configuration;
-   freeze it for the experiment.

## 22.4 Evaluator drift

Risk:

τ-bench changes during experiment.

Mitigation:

-   pin exact τ commit;
-   use \>=1.0.1;
-   store evaluator version in every run.

## 22.5 Budget drift

Risk:

Later agents simply consume more tokens/tools.

Mitigation:

-   freeze or explicitly track budgets;
-   report efficiency metrics;
-   later introduce constrained optimization if useful.

## 22.6 Overfitting discovery tasks

Risk:

Changes improve cases Claude inspected but not unseen cases.

Mitigation:

-   validation split;
-   final hidden test;
-   preserve per-generation generalization curves.

## 22.7 Uninterpretable multi-change generations

Risk:

Claude changes prompt, retrieval, tools, and planning simultaneously.

Mitigation:

-   one coherent mechanism at a time;
-   explicit hypothesis and prediction;
-   baseline/candidate comparison;
-   inspect traces behind score changes.

## 22.8 False causal stories

Risk:

Claude produces plausible narratives that are not causally responsible
for failure.

Mitigation:

-   require successful controls;
-   require counterexamples/falsifying evidence;
-   identify earliest divergence;
-   require intervention prediction;
-   validate against unseen cases.

------------------------------------------------------------------------

# 23. Suggested Project Layout

There is **no separately implemented orchestrator agent in this MVP**.

**Claude Code itself is the Improvement Orchestrator**, using the Introspection plugin. In particular:

```text
Claude Code
    +
Introspection Plugin
    │
    ├── operate  → inspect live evidence, discover signals, diagnose
    ├── improve  → propose/implement repository-owned harness changes
    └── deploy   → activate a changed recipe/runtime when deployment is required
```

Therefore, the repository must not contain an `orchestrator/` directory that could be mistaken for another agent implementation.

The project should instead separate:

1. the **target agent** being improved;
2. the **benchmark integration** providing the external objective;
3. the **contract** defining the immutable rules and permission boundaries under which Claude operates;
4. the **results/artifacts** produced by successive improvement generations.

Conceptually:

```text
introspection-self-improver/
│
├── target-agent/
│   ├── <Introspection recipe files>
│   ├── prompts/
│   ├── skills/
│   ├── tools/
│   └── tests/
│
├── benchmark/
│   ├── tau_adapter/
│   ├── split_manifest.yaml
│   └── benchmark_lock.yaml
│
├── contract/
│   ├── protocol.md
│   ├── constraints.md
│   └── learning_record.schema.yaml
│
├── results/
│   ├── generation_000/
│   ├── generation_001/
│   └── ...
│
└── README.md
```

Do not invent exact Introspection recipe filenames until implementation checks the current platform specification.

## 23.1 Why `contract/`, not `orchestrator/` or `experiment/`

`orchestrator/` is incorrect because it suggests that the repository contains another software agent responsible for self-improvement.

It does not.

The Improvement Orchestrator is:

```text
Claude Code + Introspection Plugin
```

`experiment/` is also less precise because the directory does not primarily contain experimental runs or results. Those belong under `results/`.

`contract/` contains the **experimental and operational contract** that constrains Claude's improvement process.

It defines things such as:

```text
TARGET
- Improve the banking-support target agent.

OBJECTIVE
- Improve performance under the immutable τ-Knowledge evaluator.

FIXED
- target model and version
- τ-bench version/commit
- evaluator
- discovery/validation/test split
- comparison budgets

DISCOVERY
- Claude may inspect discovery trajectories and Introspection evidence.
- Claude may use the Introspection operate workflow to investigate them.

VALIDATION
- Validation is used to test whether a candidate generalizes.
- Validation must not silently become unrestricted discovery data.

TEST
- Test tasks and trajectories are inaccessible during improvement.
- Test is used only at predetermined checkpoints/final evaluation.

MUTABLE
- target recipe
- prompts
- skills
- tools/tool descriptions
- retrieval behavior
- orchestration
- tests
- justified diagnostic evals/judges

FORBIDDEN
- modifying the τ evaluator
- modifying benchmark tasks or gold state
- changing the target model during the main experiment
- changing the task split
- hardcoding benchmark answers
- redefining the external objective
```

These are not instructions implementing another meta-agent. They are invariants supplied to **Claude Code**, which is already the Improvement Orchestrator.

The relationship is:

```text
                   contract/
                       │
               defines boundaries
                       │
                       ▼
             ┌──────────────────┐
             │   Claude Code    │
             │        +         │
             │ Introspection    │
             │     Plugin       │
             └────────┬─────────┘
                      │
             Improvement Orchestrator
                      │
          ┌───────────┴────────────┐
          ▼                        ▼
       operate                  improve
          │                        │
          ▼                        ▼
  Introspection evidence      target-agent/
                                   │
                                   ▼
                              candidate recipe
```

## 23.2 Avoid duplicating the Introspection plugin

The contract should **not reimplement or unnecessarily duplicate the methodology already encoded by the Introspection plugin**.

For example, the current `improve` workflow already provides methodological guidance around:

- beginning from evidence;
- checking controls;
- seeking falsifying evidence;
- open-coding before imposing a taxonomy;
- finding the earliest meaningful divergence;
- determining the owning layer;
- establishing a baseline;
- changing one coherent mechanism at a time;
- freezing comparison configuration;
- inspecting traces behind score changes.

Those behaviors should remain owned by the plugin.

The `contract/` directory should contain only **project-specific invariants, goals, permissions, benchmark boundaries, and reproducibility requirements** that the generic plugin cannot know.

This separation is important:

```text
Introspection plugin
    = HOW Claude investigates and improves agents

contract/
    = WHAT Claude is allowed/required to optimize in this experiment

τ-bench
    = WHETHER the resulting target agent actually improved
```

## 23.3 `operate`, `improve`, and `deploy`

The inner loop should also distinguish recipe modification from deployment.

Conceptually:

```text
operate
   ↓
understand evidence
   ↓
improve
   ↓
modify/propose target recipe
   ↓
candidate recipe/version
   ↓
deploy, only when activation is required
   ↓
execute candidate
   ↓
operate
   ↓
inspect resulting evidence
```

`improve` should not be treated as synonymous with production deployment.

If the MVP can evaluate a candidate recipe directly without activating it as a production runtime, `deploy` does not need to participate in every inner-loop iteration.

Use deployment only when required by the actual Introspection execution path.

# 24. Generation Artifact Layout

Each generation should be reproducible.

Example:

``` text
results/generation_003/
│
├── manifest.yaml
├── benchmark-results/
│   ├── discovery.json
│   └── validation.json
├── evidence/
│   └── introspection-identifiers.json
├── learning-record.yaml
├── candidate/
│   ├── base_commit.txt
│   ├── candidate_commit.txt
│   └── diff.patch
└── decision.md
```

The exact raw Introspection evidence can remain in Introspection; local
artifacts should retain stable identifiers/URLs necessary to recover it.

------------------------------------------------------------------------

# 25. MVP Success Criteria

The MVP succeeds if all of the following are demonstrated.

## Functional

1.  τ `banking_knowledge` tasks execute against the Introspection target
    agent.
2.  Objective τ outcomes are captured.
3.  Introspection captures useful execution evidence.
4.  Claude Code can use `operate` to inspect that evidence.
5.  Claude discovers at least one nontrivial actionable signal without
    being given its failure category.
6.  Claude uses `improve` to formulate a repository-owned harness
    change.
7.  The candidate can be evaluated under the same frozen configuration.
8.  The change can be accepted/rejected from validation evidence.
9.  The process can repeat for multiple generations.

## Scientific

1.  At least one accepted change improves validation performance.
2.  Final harness outperforms G0 on held-out test tasks.
3.  The improvement cannot be attributed to model/evaluator/budget
    drift.
4.  Every accepted change has an evidence → hypothesis → mutation →
    result record.
5.  At least some discovered signals are predictive of successful
    interventions.

## Showcase

A viewer should be able to understand:

``` text
what the target did
↓
what Claude observed
↓
what Claude inferred
↓
what Claude changed
↓
why it expected improvement
↓
what happened on unseen tasks
```

------------------------------------------------------------------------

# 26. Recommended Initial Experimental Sequence

## Generation 0 --- baseline

-   create minimal Introspection target recipe;
-   freeze target model;
-   integrate τ `banking_knowledge`;
-   run discovery baseline;
-   run validation baseline;
-   preserve final test.

## Generation 1+

For each generation:

1.  Run current harness on discovery tasks.
2.  Give Claude the objective outcomes plus access to Introspection.
3.  Invoke `operate`.
4.  Require evidence gathering and controls.
5.  Let Claude discover the highest-value actionable signal.
6.  Produce a learning record.
7.  Handoff to `improve`.
8.  Require one coherent proposed mechanism.
9.  Preserve unchanged baseline.
10. Create candidate after approval.
11. Evaluate candidate on relevant discovery cases and controls.
12. Evaluate on validation.
13. Inspect traces behind the score delta.
14. Accept or reject.
15. Record the decision.
16. Repeat.

Run hidden test only at predetermined checkpoints, not after every
speculative mutation.

------------------------------------------------------------------------

# 27. Later Experiments

These are explicitly **not MVP requirements**.

## 27.1 Cross-model transfer

After obtaining (H_0 `\rightarrow `{=tex}H_k), freeze (H_k) and change
the target model.

Compare:

\[ M_A + H_0 `\quad `{=tex}`\text{vs.}`{=tex} `\quad `{=tex}M_A + H_k \]

then:

\[ M_B + H_0 `\quad `{=tex}`\text{vs.}`{=tex} `\quad `{=tex}M_B + H_k \]

Question:

> Did the orchestrator discover general harness engineering knowledge or
> model-specific tricks?

## 27.2 Human-designed vs self-improved harness

Compare:

``` text
A. minimal baseline
B. human-engineered strong agent
C. autonomously improved agent
```

with the same model and benchmark.

## 27.3 Improvement-budget scaling

Measure:

\[ `\text{performance}`{=tex} =
f(`\text{meta-agent improvement budget}`{=tex}) \]

Examples:

-   meta-agent tokens;
-   number of candidate experiments;
-   wall-clock time;
-   dollar cost.

## 27.4 Harbor transfer

Reuse the same Improvement Orchestrator protocol on a Harbor benchmark.

Question:

> Does evidence-driven Introspection harness improvement transfer beyond
> knowledge-grounded customer support?

## 27.5 Weight evolution

Only after harness-only behavior is understood, consider SIA-like
model/weight updates.

At that point separate:

\[ `\Delta `{=tex}H \]

from:

\[ `\Delta `{=tex}W \]

and their interaction.

------------------------------------------------------------------------

# 28. Research Questions

Primary:

> **Can an LLM Improvement Orchestrator use Introspection's operational
> evidence to autonomously discover actionable failure signals,
> formulate hypotheses about an agent's behavior, and evolve its harness
> such that performance improves on unseen τ-Knowledge tasks?**

Secondary:

1.  What signals does the orchestrator discover without a human-defined
    failure taxonomy?
2.  Which discovered signals lead to interventions that generalize?
3.  Which harness layers are modified most often?
4.  Does harness complexity monotonically increase, or does the
    orchestrator also remove unnecessary scaffolding?
5.  How often do plausible diagnoses fail experimental validation?
6.  How many generations are required before improvements saturate?
7.  Do improvements raise pass¹ while reducing or increasing
    reliability?
8.  What is the cost/performance frontier across generations?
9.  Do discovered diagnostic evals become useful predictors of objective
    τ performance?
10. Do learned harness improvements transfer across target models?
11. Do they transfer across benchmarks/domains?

------------------------------------------------------------------------

# 29. Conceptual Contribution

The project is not simply:

> "Use Claude to optimize a prompt."

The intended contribution is:

> **An evidence-driven self-improvement loop in which an LLM
> autonomously learns both what is wrong with an agent and how to change
> the harness, using Introspection as the empirical and intervention
> substrate and an external benchmark as the immutable objective.**

There are effectively two coupled learning processes:

\[ H_0 `\rightarrow `{=tex}H_1 `\rightarrow `{=tex}H_2
`\rightarrow `{=tex}`\dots`{=tex} \]

and:

\[ E_0 `\rightarrow `{=tex}E_1 `\rightarrow `{=tex}E_2
`\rightarrow `{=tex}`\dots`{=tex} \]

where (H) is the target harness and (E) is the orchestrator's evolving
diagnostic model/instrumentation.

The external objective remains fixed:

\[ R\_`\tau`{=tex}(H_n) \]

This lets us distinguish:

-   **learning how to act**;
-   **learning how to diagnose**;
-   **actually improving according to external reality**.

------------------------------------------------------------------------

# 30. References and Source Material

## Introspection

-   Documentation: https://docs.introspection.dev
-   Work with repositories / self-improving agents:
    https://docs.introspection.dev/guides/work-with-repositories
-   Introspection plugin:
    https://github.com/introspection-org/introspection-plugin
-   `operate` skill source:
    https://github.com/introspection-org/introspection-plugin/blob/main/skills/operate/SKILL.md
-   `improve` skill source:
    https://github.com/introspection-org/introspection-plugin/blob/main/skills/improve/SKILL.md

Key verified facts from current plugin/docs:

-   `operate` owns inspection of live Introspection state and explicitly
    hands repository behavior changes to `improve`.
-   `improve` owns repository changes to behavior, prompts, tools,
    configuration, tests, evals, and judge definitions.
-   `improve` explicitly says to open-code evidence before imposing a
    taxonomy.
-   `improve` asks for falsifying as well as supporting evidence.
-   `improve` calls for an unchanged baseline and one coherent mechanism
    at a time.
-   Introspection recipes are repositories; explicit repository write
    grants can allow an agent to propose changes to its own prompts,
    skills, and tools.
-   Repository write access should be treated as a security-sensitive
    capability and normally gated through review/branch protection.

## τ-bench / τ-Knowledge

-   Repository: https://github.com/sierra-research/tau2-bench
-   τ-Knowledge paper: https://arxiv.org/abs/2603.04370
-   Current `banking_knowledge` domain: 97 tasks, 698 policy/procedure
    documents.
-   v1.0.1 contains `banking_knowledge` grading/task corrections;
    pre-v1.0.1 and \>=v1.0.1 scores are not directly comparable.

## SIA

-   Repository: https://github.com/hexo-ai/sia
-   Paper: https://arxiv.org/abs/2605.27276
-   Relevant concept: generational self-improvement using a
    feedback/meta-agent, with harness updates and model-weight updates.
    This MVP isolates harness updates.

## Harbor

-   Repository: https://github.com/harbor-framework/harbor
-   Relevant concept: portable agent evaluation and optimization
    environments; useful as a later benchmark abstraction.

## Prime Agent

-   Article: https://www.primeintellect.ai/blog/prime-agent
-   Relevant concept: trajectory-grounded incremental harness refinement
    and keeping changes focused enough to validate.

## SquareDiff

-   Platform: https://www.squarediff.com/platform
-   Thesis: https://www.squarediff.com/thesis
-   Relevant concept: autonomous harness experimentation using
    evaluation performance and agent traces.

------------------------------------------------------------------------

# 31. Implementation Guardrail for Coding Agents

When implementing this specification:

1.  **Read the current Introspection documentation and plugin skill
    sources first.** Do not assume APIs, CLI syntax, recipe layout, or
    permission behavior from this document when current upstream
    documentation differs.
2.  **Pin τ-bench before generating results.**
3.  **Do not expose held-out test data to the Improvement
    Orchestrator.**
4.  **Do not pre-label benchmark failures with a human-created
    taxonomy.**
5.  **Do not let the orchestrator modify the immutable
    objective/evaluator.**
6.  **Do not change the target model during the main MVP experiment.**
7.  **Preserve stable identifiers linking every conclusion to actual
    Introspection evidence.**
8.  **Never fabricate an observed signal.** A signal must be grounded in
    inspected executions.
9.  **Require controls and counterevidence before promoting a
    correlation into a causal hypothesis.**
10. **Prefer one coherent harness mutation per candidate.**
11. **Run an unchanged baseline under the same configuration before
    claiming improvement.**
12. **Inspect trajectories behind score changes.**
13. **Record rejected hypotheses and failed mutations as first-class
    research results.**
14. **Treat recipe repository write access as privileged.**
15. **Keep the first implementation simple enough that the origin of
    performance changes remains interpretable.**

------------------------------------------------------------------------

# 32. Short Form

If an agent needs the project reduced to one paragraph:

> Build a minimal knowledge-grounded banking support agent as an
> Introspection recipe and evaluate it on a pinned subset of τ-bench
> `banking_knowledge`. Keep the target model, benchmark, evaluator, task
> splits, and comparison budget fixed. Use Claude Code plus the
> Introspection plugin as an Improvement Orchestrator. Claude first uses
> `operate` to inspect τ failures and Introspection tasks,
> conversations, traces, tool calls, observations, patterns, and
> aggregate metrics; it must discover useful failure signals from
> evidence rather than receive a predefined failure taxonomy. It then
> uses `improve` to identify the owning harness layer, formulate a
> falsifiable hypothesis, and propose one coherent change to the target
> recipe's prompts, skills, tools, retrieval, or orchestration. Compare
> the unchanged baseline with the candidate under frozen configuration,
> validate on unseen validation tasks, inspect traces behind score
> changes, and accept or reject the mutation. Keep a hidden test set
> inaccessible during optimization. Preserve an evidence → signal →
> hypothesis → mutation → result record for every generation. The τ
> evaluator is immutable and remains the ultimate measure of success;
> Introspection diagnostics may evolve but cannot redefine the
> objective.

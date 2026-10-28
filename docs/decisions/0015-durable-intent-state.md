# ADR-0015: Durable Intent State, Reconciliation, and Context-Budgeted Handoffs

**Status:** Proposed  
**Date:** 2026-10-27  
**Scope:** ALICE control plane, projects, tasks, agent gateway, context management, observability, federation  
**Related:** [ADR-0012: Persistent Task Intent](0012-persistent-task-intent.md), [ADR-0013: User-Directed Intent Compilation](0013-user-directed-intent-compilation.md), [ADR-0014: Facepalm Feedback](0014-facepalm-feedback.md), [System Architecture](../ARCHITECTURE.md), [Design Principles](../DESIGN-PRINCIPLES.md)

## Context

ALICE is the intended workbench. Twinny is an experimental stepping stone used to develop and test ideas, not ALICE's architectural foundation or a required runtime.

Today's agents generally rely on long conversational context, increasingly aggressive summarization, and re-injection of a previous context block whenever an inference engine, model, session, or agent changes. This approach has a fundamental weakness: a summary of *what was said* is not necessarily a faithful representation of *what must remain true*. It can mix authoritative user requirements with provisional agent conclusions, lose negative constraints, preserve obsolete plans, and omit the evidence needed to tell success from failure.

This becomes particularly visible with smaller usable context windows or frequently pruned contexts. A compacted history can lose the objective while retaining incidental details. Repeatedly pasting an ever-growing instructions file into the next request is also a fragile form of continuity.

ADR-0012 establishes durable task records; ADR-0013 introduces optional, user-directed intent compilation; ADR-0014 captures explicit feedback when execution loses the plot. The next architectural step is to make **intent reconciliation and context projection ongoing control-plane functions**, not merely a one-time prompt rewrite or a passive archive.

## Decision

**ALICE shall maintain durable, inspectable, versioned intent state outside any individual model's conversation or KV cache, and shall reconcile proposed execution against that state throughout a task's lifecycle.**

An agent or model is an interchangeable participant in execution. The ALICE control plane maintains continuity across model/provider changes, context resets, compaction, restarts, agent handoffs, and recoverable failures. It supplies each participant with an appropriate, bounded *projection* of intent and working state rather than assuming that replaying old conversation text will preserve purpose.

**Guiding distinction:**

- **Context:** what a model is currently given or remembers within its active inference session.
- **Intent:** the authorized objective, constraints, accepted decisions, and success conditions that ALICE must preserve across sessions.
- **Execution:** the proposed and observed actions taken toward that objective.
- **Evidence:** what independently supports or contradicts claims of progress, completion, or recovery.

This extends ADR-0012; it does not replace its persistent task record. It also does not make model-generated summaries authoritative or grant agents new permissions.

## Scope and ownership

ALICE's existing entities remain separate:

- A **Project** supplies project-level purpose, policies, resources, standing conventions, and inherited constraints under authorized ownership.
- A **Task** carries its specific original request, accepted interpretation, exclusions, objective, evaluation criteria, status, and revision history under a stable `task_id`.
- A **Session**, agent runtime, inference provider, or model is an execution participant, never the source of truth for the task's authority.
- A **Principal** and trusted policy broker determine who may authorize changes or operations. An intent document cannot impersonate a grant.

Task-specific decisions may override applicable project defaults only where project policy permits. A conflict between a user request and higher-priority policy must be represented and resolved, not silently flattened into one prompt.

## Intent representation: three practical layers

The layers are logical views over durable task state and referenced artifacts, not a mandate for three databases.

### 1. Intent kernel: small, stable, high-value

Retain, with authority and provenance:

- The **original user request** (unaltered, recoverable) and any user-approved revision.
- The **current authorized objective**, rationale, scope, exclusions, and non-negotiable constraints.
- Explicit **accepted decisions**, superseded decisions, and their effective revisions.
- **Acceptance criteria** and critical known conditions, including expected failures where relevant.
- References to trusted **policy/grants** without copying or bestowing authority.
- A small set of **must-not-forget invariants**, such as preserving existing archives, avoiding unapproved external writes, or maintaining independent provider selections.

The kernel is not a universal immutable prompt. It is a compact, versioned view of authoritative records, updated only through an authorized change process.

### 2. Working state: current phase and next useful action

Retain the plan and phase, completed and pending steps, affected resources, unresolved questions, current hypotheses, checkpoints, relevant tool capabilities, and the next verification step. This layer changes often; stale plans must be marked as stale rather than promoted to rules.

### 3. Evidence and history: available, not always in context

Retain bounded observations and links to source messages, tool calls, diffs, logs, artifacts, tests, hashes, timestamps, and execution provenance. Fetch these when pertinent. The full chat transcript may be one evidence source, but is neither the default handoff format nor the authority for current task state.

A suggested implementation is an append-only revision/event journal with a materialized current snapshot and content-addressed or securely referenced evidence where appropriate. Exact schemas and stores remain open and versioned.

## Continuous reconciliation and feedback

Intent compilation (ADR-0013) can produce a proposed starting interpretation. **Intent reconciliation continues after execution begins.** A reconciliation check compares proposed actions, observations, and changes in working state with the current authorized intent, known conditions, and acceptance criteria.

Recommended boundaries for reconciliation include:

1. **Before execution:** validate the accepted task interpretation, scope, exclusions, and expected evidence.
2. **Before consequential actions:** check destructive, externally visible, privileged, or difficult-to-reverse operations against the current scope and trusted policy. An LLM judgment never replaces the authorization check.
3. **After material observations:** classify expected errors, unexpected blockers, partial progress, and evidence that invalidates an assumption.
4. **At phase transitions or checkpoints:** update working state and verify that completed steps truly advance the objective.
5. **On explicit user feedback:** attach 👍, 👎, or 🤦 to the relevant span (ADR-0014), flag disputed outcomes, and propose repairs or corrections without automatically rewriting user intent.
6. **Before compaction, handoff, or provider/model switching:** materialize a fresh bounded projection and include unresolved contradictions rather than burying them in a summary.

Reconciliation should be event-driven and proportionate to task risk and available resources, not an unconditional extra model call after every token or tool operation. Deterministic checks should be used where adequate; a reasoning model may assist with proposed classifications but cannot be the final authority for permissions or evidence integrity.

### Changes to intent: proposals are not authority

Keep distinct records for: **explicit user instructions**, **authorized accepted interpretations/decisions**, **agent inferences or suggestions**, **tool observations**, and **trusted policy decisions**. Every material change records its source, revision, authorizing principal when required, and the superseded value. Material changes to objective, exclusions, acceptance criteria, or permission boundaries require the applicable trusted approval. Agents may refine working plans within authorized scope but must not silently expand that scope.

Negative constraints deserve first-class representation. “Do not modify files” or “preserve archived binaries” must not disappear simply because a newer plan emphasizes a positive goal.

The system must detect and surface **contradictions and uncertainty**. It must not manufacture an apparently consistent story by silently dropping one side.

## Context-budgeted projections and small-model operation

ALICE shall be able to construct a **bounded task-state projection** suitable for a selected inference context budget. The projection is generated from durable state, not from unrestricted summarization of the previous prompt.

A projection should normally prioritize, in order:

1. Source and authority labels, original/accepted objective, and critical constraints.
2. Current task phase, accepted decisions, immediate target, and required verification.
3. Relevant known failures and unresolved contradictions.
4. Small, necessary excerpts of evidence or references for retrieval.
5. Optional background conversation only when useful and within budget.

It shall preserve explicit exclusions and unresolved issues ahead of incidental conversational details. If necessary information will not fit safely, the agent must request retrieval or signal an insufficient-context condition rather than guess. Users and developers should be able to inspect what was included, omitted, or truncated.

The budget and packing strategy may vary by model, provider, task, and risk level. Avoid requiring a particular token count, tokenizer, prompt dialect, or inference API. Preserve the underlying canonical intent and provenance independent of each projection.

**KV cache clarification:** this architecture does **not** preserve a model's KV cache across incompatible engines or pruning events. It preserves *semantic task state* outside inference and reintroduces that state as new context. It may reduce re-prompting overhead and intent loss for small-context deployments, but those benefits must be measured rather than assumed.

## Model, provider, and agent handoff

A handoff shall carry at least the stable task identity; revision of accepted intent; current scoped projection; working checkpoint; outstanding acceptance checks; evidence references; relevant environment/resource identities; and any explicitly unresolved conflicts. The receiving participant must not infer it has authorization solely from this handoff.

After switching models or providers, ALICE should verify that the new participant can consume the projection and has the required declared capabilities. It should not assume a different model has the same tools, context budget, reasoning format, or understanding of a previous model's prose. The handoff must make unavailable capabilities and skipped evidence visible.

A handoff may occur between distinct specialized agents, or from a small local model to a larger one and back. No part of this decision requires Twinny, Mycroft, proprietary hosting, or a specific inference server.

## Trust, privacy, and recoverability

- ALICE's trusted control plane stores the durable record; model-generated text is untrusted proposed state or evidence, not a policy instruction.
- Repository files, web content, messages, and tool results must retain provenance and trust labels. Prompt injection must not be promoted into task authority during consolidation.
- Do not embed credentials or unnecessary private reasoning in a compact state. Enforce project-specific retention, redaction, export, and provider-disclosure policies.
- A provider switch must not transfer sensitive task evidence to a newly selected model endpoint without applicable authorization and disclosure.
- Do not mistake a restated task, a renamed file, or a model's report of success for verified recovery. Compare observable state with acceptance criteria and, where applicable, hashes or immutable artifact identities.
- Durable snapshots must support revision-aware updates and safe recovery after interruption. Concurrent or federated edits need conflict detection and explicit reconciliation.
- Store enough event and revision metadata for audit without indiscriminately duplicating every prompt, cache, tool payload, or artifact.

## Relationship to existing ALICE architecture and APIs

This belongs in ALICE's project/task control-plane responsibilities and can be exposed through existing authorized **task**, **event**, and **agent gateway** contracts. Keep storage, projection, compiler, reconciler, and execution roles conceptually distinct and replaceable; do not introduce a single privileged model process.

**Do not create or reserve a concrete `/intent` route as part of this ADR.** The `/intent` name remains available for possible future ALICE control-surface design. Endpoint naming, CLI syntax, wire schema, and UI are separate decisions. The Twinny `intent` provider type is an experiment in optional pre-execution compilation, not this runtime state's API or security boundary.

## Implementation sequence (ALICE, not Twinny)

1. Extend task records with versioned authoritative intent, source labels, negative constraints, acceptance checks, and revision history (building on ADR-0012).
2. Generate inspectable bounded projections and deterministic handoff packages; support a no-LLM reference implementation.
3. Add reconciliation events at checkpoints, before risky actions, and after observations; keep proposed corrections separate from approved changes.
4. Connect optional Intent Compilation (ADR-0013) to the record as a proposal, not as task authority.
5. Feed explicit feedback (ADR-0014) into dispute/review/recovery workflows with traceable evidence.
6. Evaluate context-budget strategies with small and large models; tune projections based on measured task fidelity rather than token count alone.

Each step must work on a standalone, self-hosted ALICE node and remain compatible with future federation and replaceable backends.

## Verification and acceptance criteria

A prototype meets this ADR's intent when the following tests pass:

1. **Compaction continuity:** after aggressive compaction, a small-context agent retains the original objective, critical exclusions, pending checks, and current phase even when a conversation-summary baseline loses them.
2. **Distinct-engine handoff:** switching between independently configured model/provider engines preserves task identity, provenance, accepted intent, and evidence requirements without merely pasting an entire old chat transcript.
3. **No silent intent drift:** an agent-proposed revision that contradicts an explicit prohibition is surfaced and remains unapplied without authorized acceptance.
4. **Negative-constraint guard:** a proposed destructive operation that conflicts with a recorded preservation requirement is blocked or sent for the appropriate authorization/review; policy enforcement remains independent of model judgment.
5. **Evidence-aware recovery:** copying a new binary under an archived filename does not qualify as restoring that archive without matching provenance or acceptable verification evidence.
6. **Feedback loop:** 🤦 marks a specific disputed result and initiates review or a proposed correction without auto-retraining, auto-deleting, or rewriting the user's original intent.
7. **Expected-error handling:** a known failure reproduced as part of an experiment is recorded as an expected observation, not automatically treated as a blocker or proof that the entire task passed.
8. **Provenance isolation:** tool output containing instructions cannot become authoritative task constraints when a projection is rebuilt.
9. **Resource and privacy limits:** projection respects the selected model's context budget and project disclosure policy, reporting omissions or blocked handoffs clearly.
10. **Restart/revision safety:** state survives process restart; concurrent modifications cannot silently overwrite the latest accepted task revision.

Benchmarks should report independently verified task success, constraint violations, context tokens sent, latency and cost of reconciliation, handoff/recovery success, and how often users must correct intent drift. Small-model benefits are hypotheses to evaluate, not guaranteed results.

## Open questions

- Exact schema and migration rules for kernels, projections, events, and task snapshots.
- How project standing intent and task-specific intent compose under different principal permissions.
- Which deterministic checks can verify intent preservation before involving a model.
- Projection packing, retrieval, confidence, and budget policies for very small contexts.
- Best revision and concurrency mechanisms across offline or federated nodes.
- How to render an accessible future intent inspection/control surface without exposing private data or confusing model proposals with accepted state.
- How to evaluate long-horizon drift and false-positive reconciliation interruptions in real workloads.

## Consequences

**Benefits:** ALICE can preserve purpose independently of chat history, support deliberately replaceable execution engines, recover more coherently from compaction and handoffs, make feedback actionable, and give small-context models a clearer current contract.

**Costs and risks:** maintaining the record and reconciling it add implementation complexity, latency, and storage; automatically generated projections can still be wrong or incomplete; excessive checks can interrupt useful work; and the task record itself becomes a sensitive integrity and privacy boundary. Inspection, provenance, bounded budgets, explicit authority, and measurable evaluation are required mitigations.

**Guiding principle:** *Context is what the model knows. Intent is what ALICE must preserve. Execution is how work advances; evidence is how ALICE knows whether it did.*
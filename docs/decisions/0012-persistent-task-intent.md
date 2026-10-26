# ADR-0012: Persistent Task Intent and Failure-Aware Orchestration

**Status:** Proposed  
**Date:** 2026-10-26  
**Scope:** ALICE control plane, task management, agent gateway, observability  
**Related:** [System Architecture](../ARCHITECTURE.md), [Design Principles](../DESIGN-PRINCIPLES.md)

## Context

An agent's chat context is not a dependable system of record. Context windows fill, conversations are compacted, tools fail, model instances restart, and orchestration may switch between agents or inference providers. During these transitions an agent can remember *what it was doing next* but lose *why it was doing it*.

A concrete example arose while diagnosing Twinny and llama.cpp compatibility in October 2026. The task was to verify a workaround for a known HTTP 400 response, `All non-assistant messages must contain 'content'`. Copilot reproduced that very response and treated it as an unrelated endpoint failure that prevented testing. A human had to restore the essential context: encountering that failure was the test's purpose, and the explicit system-message workaround was what needed verification. In a separate test, activity performed using Copilot's tools risked being misidentified as evidence that Twinny's own agent-tool loop had been exercised.

These were failures of **operational intent, expectation tracking, and evidence provenance**, not simply failures to retain more conversational text.

ALICE already distinguishes projects, tasks, sessions, agent profiles, principals, and environments. It also calls for observable outcomes and stable task identity across reconnections. The missing explicit requirement is a **durable, model-independent representation of task intent and evaluation criteria**, owned by the orchestration layer rather than entrusted to an LLM's transient prompt.

## Decision

ALICE shall maintain a **persistent, structured task record** in its control plane, separate from any model's conversational context. A task is addressed by a stable `task_id` and may span sessions, agent instances, inference backends, environments, and context-compaction events.

The record shall retain, with versions and provenance where applicable:

1. **Objective and rationale:** the authorized requested outcome and why it matters.
2. **Scope and constraints:** affected resources, explicit exclusions, environmental assumptions, policy references, and permissions. The record does **not** confer authorization.
3. **Known conditions:** established defects, expected error signatures, hypotheses, tested workarounds, and their confidence and applicability.
4. **Acceptance criteria:** measurable evidence required to declare success, failure, or a blocked state.
5. **Execution and observations:** attempted actions, tool invocations, timestamps, outputs or references to outputs, outcomes, unresolved questions, and checkpoints.
6. **Failure classifications:** whether each observed error is expected for the current test, unexpected but recoverable, a genuine blocker, or not yet classified, with supporting evidence.
7. **Provenance:** which principal, agent runtime, tool adapter, system under test, and execution environment produced an action or observation.

The control plane is the authority for the task's recorded intent. An agent may propose corrections or revisions, but **must not silently redefine an objective, erase a known condition, relax acceptance criteria, or alter permission boundaries**. Material changes require authorization according to project and task policy, and must leave an auditable revision trail.

### Orchestration behavior

- Before abandoning work in response to an error, an agent or its supervisor must compare the observation with the task's known conditions and acceptance criteria.
- An expected error is an **observation**, not automatically a success and not automatically a blocker. Its meaning depends on the current test phase and the evidence required.
- Unknown or ambiguous errors must remain explicit; classification must not suppress security failures, data-loss risks, or legitimate blockers.
- On context compaction, restart, reconnect, model substitution, or agent handoff, the orchestrator shall supply a **current, scoped task-state summary** reconstructed from durable records. Raw conversation replay is optional, not the source of truth.
- Agents must report which system actually executed an operation. A supervising agent's own tool call is not proof that the system under test performed the same action.
- A task may be marked successful only when the required acceptance evidence exists. Use distinct outcomes such as **passed**, **failed**, **blocked**, and **not verified** rather than equating command completion with task success.
- Tool and observation references shall correlate with the existing task, request, trace, principal, session, and resource identifiers described by the system architecture.
- Humans must be able to inspect and correct the task interpretation, retain the correction across handoffs, and resume work from a checkpoint without losing the original request.

### Security and trust boundaries

This is **not** a request to put trust or authorization in a prompt. The task record is controlled by trusted ALICE services. Model-produced summaries, tool output, repository contents, and external responses are evidence or proposals, **not** privileged instructions or policy grants. Preserve the source and trust level of each observation.

Do not store credentials or unnecessary private reasoning in the task journal. Retention, visibility, integrity, redaction, and export must follow project policy. Evidence references may point to external logs or artifacts without copying their sensitive contents into every task summary.

### Architectural separation

ALICE owns **task continuity**, not every model's hidden state. Agent runtimes remain free to choose their own prompting, compaction, reasoning methods, and private conversational memory. The proposed task record does not mandate a database engine, serialization format, prompt format, particular LLM, or specific agent framework.

The initial implementation may use an append-only sequence of task events plus a materialized snapshot; those are candidate mechanisms, not yet a frozen storage or wire protocol.

## Example acceptance test

Given a task to validate Twinny's llama.cpp HTTP 400 workaround:

1. The task record identifies the unmodified user-only request as a known reproduction case, and an explicitly prepended system message as the workaround being tested.
2. The agent sends a baseline request and receives HTTP 400. It records **expected reproduction observed**, without declaring the entire integration test blocked.
3. The test then verifies that Twinny itself inserts the system message in its outgoing HTTP payload and receives a successful response.
4. A follow-up native tool call must be issued through Twinny's own tool loop, not through the supervising agent's unrelated tools.
5. The task passes only if the recorded HTTP requests, tool invocations, and results meet all acceptance criteria.
6. After context compaction or swapping to another agent, the same objective, known defect, test phase, and outstanding criteria remain available.

## Consequences

**Benefits**

- Work survives model changes, context resets, and session interruptions without relying solely on prose reminders.
- Failure-handling decisions consider why a test is being performed.
- Claims of success can be audited against evidence from the correct execution subsystem.
- Human corrections become persistent task revisions instead of ephemeral chat instructions.
- Multiple agents can cooperate on a task without mistaking their own tool activity for another agent's results.

**Costs and risks**

- Durable records require lifecycle, access-control, retention, revision, and synchronization design.
- Automated classifications can be wrong; confidence and evidence must remain inspectable.
- Capturing too much tool data can be expensive or sensitive; prefer bounded summaries and secure artifact references.
- Resume semantics across externally mutated resources require reconciliation, not blind replay.

## Implementation guidance and verification

For the first implementation slice, supply a durable task envelope with objective, known conditions, acceptance checks, observations, provenance, revisions, and a resumable checkpoint. Expose it to the agent gateway through a capability-filtered task interface and corresponding events.

The implementation shall be considered adequate for an initial prototype when tests demonstrate that:

- A known, expected error does not cause the agent to abandon its original test.
- A genuine unexpected blocker is surfaced rather than dismissed as expected.
- Context compaction and agent/model replacement preserve the task's objective and remaining acceptance checks.
- Tool provenance distinguishes the supervising agent's actions from those of the system under test.
- An agent cannot change the authoritative objective or expand permissions merely by emitting a message or editing a project file.
- A human-authorized correction is recorded and honored on the next execution or handoff.

## Open questions

- Exact task event and snapshot schemas, storage backend, and API versioning.
- Revision approval policy for multi-user projects and federated ALICE nodes.
- Retention and evidence-integrity guarantees for long-running tasks.
- Which recovery steps may be automatic and which require human approval.

**Guiding principle:** ALICE must remember **why an action is being performed**, not merely **what action comes next**.

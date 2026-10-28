# ADR-0016: Event-Driven Tiered Intelligence and Intent Conscience

**Status:** Proposed  
**Date:** 2026-10-27  
**Scope:** ALICE control plane, task orchestration, model gateway, resource scheduling, observability, self-hosted deployment  
**Related:** [ADR-0012: Persistent Task Intent](0012-persistent-task-intent.md), [ADR-0013: User-Directed Intent Compilation](0013-user-directed-intent-compilation.md), [ADR-0014: Facepalm Feedback](0014-facepalm-feedback.md), [ADR-0015: Durable Intent State](0015-durable-intent-state.md), [System Architecture](../ARCHITECTURE.md), [Design Principles](../DESIGN-PRINCIPLES.md)

## Context

ADR-0015 makes intent durable, model-independent state. It separates the current intent kernel, working state, and historical evidence, and requires bounded context projections on model or session handoff. It does not prescribe how to distribute reasoning among models, when to invoke an expensive reviewer, or how a resource-constrained self-hosted machine should schedule inference.

A single general-purpose agent repeatedly asked to execute, plan, remember, compact, and audit its own work can lose constraints or justify a mistaken path. Conversely, making a large reasoning model examine every tool response wastes latency, energy, memory bandwidth, and money. Continually running several models is especially impractical on a home lab with limited accelerator memory.

ALICE should exploit *different reasoning horizons*: fast, focused execution; medium-horizon intent management; and infrequent, evidence-grounded historical review. The heavier reviewer is described as a **Conscience** as a functional metaphor: it is an independent audit role, not consciousness, moral agency, or permission authority.

This also restores the importance of a balanced *computer*, not merely its GPU. CPUs orchestrate processes, storage and retrieval, evidence indexing, policy evaluation, testing, sandboxing, model loading, and network transfers. System RAM and storage performance shape what can be staged or offloaded, and whether switching models is practical. The GPU remains useful for inference, but need not be the only valuable resource.

## Decision

**ALICE shall support event-driven, three-tier reasoning roles backed by replaceable model/provider selections and governed by a deterministic orchestration layer.**

The conceptual roles are:

| Role | Typical model profile | Primary responsibility | Invocation pattern |
| --- | --- | --- | --- |
| **Executor** | Small, task-focused, comparatively short context | Execute bounded actions using tools; observe and verify concrete results | Frequently, while the task is active |
| **Intent** | Medium-capability, stronger planning and interpretation | Maintain the authorized task contract and working state; produce scoped execution projections; interpret failures, replan within scope, and prepare handoffs | On new tasks and material state transitions |
| **Conscience** | Higher-capability, longer-horizon analytical model, or staged review over evidence | Examine the evolution of intent, decisions, contradictions, drift, and claimed recovery against authoritative history | On selected events or risk thresholds, not per token/tool call |

These are **logical roles, not fixed parameter sizes, processes, machines, or trust levels**. One model may fulfill multiple roles when hardware is limited, or distinct models may run on separate hosts. ALICE must not require any specific commercial model, cloud service, GPU vendor, runtime, or three simultaneous model instances.

**Guiding rule:** *Allocate reasoning effort to the complexity and risk of the decision, not to the accumulated length of the conversation.*

## Architectural boundaries

ALICE's **trusted control plane** owns event routing, role assignments, accepted task revisions, scheduling, authorization, and outcome recording. Models supply candidate interpretations, plans, classifications, or actions. A role name never grants elevated permissions.

- **Durable task state (ADR-0015):** canonical intent, authorized revisions, working checkpoints, evidence references, provenance, and disputes. Never owned exclusively by a model context.
- **Intent role:** prepares and updates bounded task contracts; does not silently rewrite the original request or override explicit accepted constraints.
- **Executor role:** operates within granted tools, scope, budgets, and task contracts. Reports actual tool outputs, including uncertainty and missing verification, rather than assuming success.
- **Conscience role:** independently reviews the historical record and relevant primary evidence. It may challenge assumptions, detect drift, propose reconciliation, and recommend escalation, but **cannot grant permissions, rewrite authoritative intent, or execute changes by virtue of being Conscience**.
- **Policy and approvals:** trusted principals and the existing policy broker remain authoritative for consequential operations; safety and authorization checks cannot be delegated solely to LLM judgments.
- **Model gateway:** resolves separately configured role-to-provider bindings, capability/context budgets, health, and disclosure limits. Reuse existing inference transports wherever suitable; no new `/intent` endpoint is authorized or specified by this ADR.

### Logical dataflow

```text
User / authorized project state
              |
              v
  ALICE control plane + event journal <---- tool results, tests, user feedback
              |         |                          |
              |         +-- deterministic guards --+
              |
    durable Intent Kernel / Working State / Evidence (ADR-0015)
              |
      relevant event, risk, resource budget
              |
              v
       Event router + scheduler
        |         |          |
        v         v          v
    Executor    Intent    Conscience
    focused     contract  historical audit
    actions     /handoff  /contradictions
        |         |          |
        +---------+----------+
                  |
       evidence-linked proposals
                  |
                  v
     trusted reconciliation / approvals
                  |
                  v
        revisioned next action
```

Conscience does not continuously watch a live chain of thought. It reviews **observable events, authorized decisions, prior task revisions, and evidence**. Private model reasoning is neither required nor a substitute for external verification.

## Event-driven activation and escalation

An event may trigger deterministic handling, a model call, a human review, or no action. The event router should prefer the least costly adequate response, while respecting risk and user policy.

| Event or condition | Default owner | Expected response |
| --- | --- | --- |
| New task or explicitly revised objective | Intent | Interpret and propose/refresh task contract using ADR-0013 preferences; preserve original |
| Routine tool completion with sufficient evidence | Executor / deterministic verifier | Continue within budget; record observation; no Conscience call |
| Repeated failure, oscillating fixes, blocked verification | Intent | Reclassify failure, inspect evidence, propose bounded replanning |
| Context budget threshold, compaction, restart, model/provider handoff | Intent + orchestrator | Materialize a scoped projection; validate context and role capabilities |
| Conflict with a preserved negative constraint | Deterministic guard first, then Conscience when needed | Prevent unauthorized action; review contradiction with evidence |
| Major architecture reversal or ambiguous supersession of user intent | Conscience | Historical audit and evidence-linked proposal; ask the authorized user if a material change is needed |
| Disputed outcome / 🤦 feedback (ADR-0014) | Conscience or cheaper triage first | Inspect specified action span, compare claims and evidence; propose recovery, not automatic mutation |
| Apparent restoration or destructive recovery | Deterministic integrity checks + Conscience if ambiguous | Verify identities, backups, content or hashes; do not accept renamed substitutes |
| Sustained resource pressure or queue overload | Scheduler | Defer, batch, choose permitted lower-cost role profile, or report constraint |
| Uncertain privilege or missing approval | Policy/approval path | Fail closed; never treat a favorable model verdict as authorization |

Escalation should normally be **asymmetric**: Executor → Intent when task strategy needs revision, and Intent → Conscience when an authoritative/historical contradiction needs deeper analysis. Conscience returns findings to the control plane, which determines what can be accepted and what must be sent to the user. Direct high-risk triggers may bypass the normal progression to avoid unsafe delay.

### Event envelope and scheduling semantics

Events should be correlatable to `project_id`, `task_id`, task/intent revision, run/step identifiers, originating principal or component, timestamp, triggering observation, priority, and evidence references where applicable. This is a **logical contract**, not a frozen wire schema.

The router and workers must support idempotent processing, deduplication, bounded retry/backoff, cancellation, deadlines, priorities, and admission control. A stale Conscience verdict must not overwrite a newer approved intent revision: findings are checked against the exact revisions they reviewed. Queued audits must be cancelable or marked obsolete after relevant user changes.

Routine progress should not block on heavyweight review. Operations whose risk or policy explicitly requires review are held at a safe boundary until it finishes or escalates; if the reviewer is unavailable, those operations must not silently bypass mandatory safeguards. Users should be able to observe and interrupt active work.

## Intent contracts and feedback loop

The Intent role produces **short, versioned execution contracts** suitable for the selected Executor's actual capabilities and context budget. A contract may include:

- Source task and accepted intent revision; scoped objective and immediate phase.
- Explicit must-do and must-not-do constraints, affected resources, and available grants **by reference**.
- Relevant files, observations, known failures, and evidence pointers, not an indiscriminate transcript.
- Allowed tool capabilities and required checkpoints.
- Verifiable success conditions and what constitutes blocked or not-yet-verified work.
- When to ask for help or escalate instead of guessing.

The Executor returns structured progress, observations, and any proposed contract conflict. The Intent role can revise working plans **within existing authorization**. Material changes to purpose, exclusions, success criteria, or permission scope follow ADR-0015 revision approval. Neither Intent nor Conscience may convert model output or repository text into privileged instructions.

### Conscience: independent historical audit

A Conscience review receives a bounded case file containing relevant intent revisions, explicit user decisions, conflicting plans/actions, primary evidence, and provenance. It should be able to retrieve further evidence through read-only, policy-scoped interfaces, rather than trusting only the Executor's own summary.

Its output is an **evidence-linked advisory finding**, such as:

- **Aligned:** examined evidence supports the current trajectory, with stated limitations.
- **Potential drift:** a discrepancy or omitted constraint with links to competing sources.
- **Contradiction:** incompatible accepted requirements or action versus preserved prohibition.
- **Unverified recovery:** success claim lacks evidence of restored original state.
- **Insufficient evidence:** cannot establish alignment; request retrieval, a test, or human interpretation.

Findings must distinguish observed facts, interpretation, confidence/uncertainty, applicable intent revision, and proposed next check. The control plane records the outcome and can request further evidence, pause a risky action, or surface a human decision. **A model cannot certify its own compliance, even when labeled Conscience.**

### Histories larger than the Conscience context

When even the Conscience cannot read the full history in one pass, ALICE may partition events by time, task phase, decision lineage, or evidence graph; build evidence-linked partial findings; retrieve counterexamples; and perform a higher-level reconciliation. This is not permission to replace original evidence with recursively summarized guesses.

Preserve the immutable original user requests and authorized revisions, anchor summaries to stable record IDs and content identities where feasible, and make omitted history discoverable. Sampling, retrieval, and hierarchy introduce recall risk; report uncertainty and missed-evidence rates in evaluation. An apparent contradiction should be checked against its original sources before any material user-facing escalation.

## Hardware-aware self-hosted scheduling

**A balanced workstation matters.** Small, local deployments should work with one accelerator or CPU-only inference where feasible. The scheduler should recognize that execution, coordination, testing, and historical audit have different resource profiles.

| Resource | ALICE-relevant work | Scheduling implications |
| --- | --- | --- |
| **CPU cores / performance** | Event routing, deterministic policy, tool execution, builds/tests, indexing, parsing, retrieval, compression, inference preprocessing and CPU offload | Keep cores available for the OS, tools, and active execution; constrain background jobs; beware oversubscription |
| **System RAM** | Durable state caches, search indexes, model staging, CPU-side weights, working sets, and KV cache when engine supports it | Monitor resident memory, page faults, swap thrashing and allocation failure; avoid promising large-model availability solely from total RAM |
| **SSD / storage I/O** | Model weight loading, checkpoints, event journal, retrieval store, source trees, build artifacts | Model swaps and evidence scans can be I/O-bound; batch or stagger heavyweight loads; maintain storage headroom |
| **GPU / accelerator memory and bandwidth** | Low-latency token generation, offloaded layers, accelerator-resident KV cache | Keep the frequently used Executor resident when beneficial; release or swap other roles on demand if safe |
| **Interconnect / networking** | Remote models, distributed nodes, artifact transfer and streaming results | Account for latency, transport saturation, privacy, and provider authorization before dispatch |

Model size tiers are preferences, not requirements. Illustrative configurations could use a small specialized Executor, medium Intent model, and heavier Conscience model, but **role assignment is capability- and benchmark-based**, not a fixed parameter-count hierarchy. A larger model may not reason better on a particular task. Some contexts or quantizations do not fit; CPU offload can be slow; disk staging can dominate an audit's wall time.

A single-GPU lab should be able to run **one inference workload at a time**, favor a resident Executor, invoke Intent at checkpoints, and load Conscience only when warranted. A powerful CPU, ample RAM, and fast NVMe may make occasional larger-model audits viable even without additional GPUs, but this is conditional on the actual engine and observed memory, load-time, and token-throughput behavior.

The scheduler should estimate or measure model load time, warmup, tokens/sec, context footprint, KV/cache residency, peak RAM/VRAM, tool latency, power/thermal constraints when available, and the opportunity cost of evicting an active model. It may use remote resources only with explicit policy and data-disclosure approval. If resource pressure is too high, prefer a safe, visible delay/degradation or user intervention to silently truncating critical intent.

No component requires every tier to run concurrently, or requires access to a cloud model. Conscience review can be deferred when not mandatory. Deterministic protections remain active even when all model-backed roles are offline.

## Failure handling and trust

- **Model disagreement:** record competing interpretations and supporting evidence; do not automatically elevate the larger model's answer over an explicit user decision.
- **Conscience unavailable:** continue low-risk authorized work where policy permits; queue optional audit. Hold operations explicitly gated by required review or approval.
- **Intent unavailable:** preserve task state; resume with a previously accepted scoped contract where valid, or surface a need for review. Never generate a fabricated revision.
- **Executor failure:** capture the actual failure and checkpoint; bounded retry or handoff must preserve identity and acceptance tests.
- **Prompt injection:** retrieved history, code comments, and tool outputs retain trust labels; analysis text cannot become project policy or user authorization.
- **Privacy:** large-model audits may expose a broader evidence set than the Executor ordinarily sees. Scope, redact, retain, and disclose according to project policy and provider destination.
- **Liveness:** avoid recursive self-review and circular escalation. Impose per-task escalation budgets, loop detection, deadlines, and human override where appropriate.
- **No hidden safety substitution:** resource scheduling cannot move sensitive tasks to an external provider or widen available tools just to satisfy a compute budget.

## Incremental implementation approach

1. Build a **no-LLM event router and state replay** prototype over ADR-0015's journal, with trusted revision checks and existing ALICE task/gateway contracts.
2. Implement one **Executor** receiving a bounded, externally constructed task projection; preserve baseline behavior when other roles are disabled.
3. Add the **Intent** role for task preparation, checkpoints, compact handoffs and bounded replanning, with explicit independent provider selection and current ADR-0013 user preferences.
4. Add a **Conscience** read-only audit worker for high-value event types (contradictions, consequential revisions, disputed recovery, 🤦), producing evidence-linked, advisory findings.
5. Introduce hardware-aware model scheduling and admission control, starting with a single-host, single-accelerator profile and optional CPU-only mode.
6. Evaluate long-history partitioning/retrieval and optional distributed review after the simpler single-node path is demonstrably useful.

This ADR defines architectural direction, not a requirement to create three processes or a new API route immediately. Twinny remains an experiment and is **not** an ALICE runtime dependency.

## Verification and acceptance criteria

An implementation should demonstrate:

1. **Role independence:** Executor, Intent, and Conscience have independently configurable logical bindings. Reassigning one does not silently alter the others. The same physical model may be selected explicitly.
2. **Event discipline:** a routine successful tool step invokes no unnecessary Conscience call; meaningful failures or historical contradictions trigger appropriate escalation.
3. **Intent fidelity:** after repeated compactions or a provider/model switch, the short-context Executor still receives current authorized goals, negative constraints, acceptance checks, and unresolved uncertainties.
4. **Drift detection:** a model attempts a change contrary to an earlier unrevoked constraint; a reviewer identifies the conflicting revision and original evidence, without rewriting the objective.
5. **Human authority:** Conscience advice cannot grant privileges or silently authorize destructive recovery, scope expansion, or modifications to accepted intent.
6. **Evidence-based recovery:** renamed copies of a newer artifact cannot count as restoration of multiple distinct historical versions without identity evidence.
7. **Single-host lab:** the three roles operate sequentially on one constrained host with reproducible resource traces; mandatory deterministic protections work even with Conscience unloaded.
8. **Resource containment:** model swapping, parallel tool processes and retrieval do not exceed configured RAM/VRAM/CPU limits silently; pressure yields observable queuing or controlled failure.
9. **Crash and stale-result safety:** restart replays durable state without duplicate consequential actions; an obsolete review cannot overwrite a newer task revision.
10. **Long history:** partitioned audit carries citations to original events, detects a seeded early contradiction, and reports when evidence retrieval was incomplete.
11. **Offline and disclosure compliance:** no cloud service is required; choosing a remote reviewer does not transfer evidence without permission.
12. **Performance comparison:** compare against a single-model baseline on task success, verified constraint violations, latency (including model loads), tokens generated, peak RAM/VRAM, CPU utilization, disk I/O, energy when measurable, review false positives, and user interruption rate.

Any claimed advantage for smaller models or weaker GPUs requires measured improvement under stated hardware, software, model, and workload conditions. The objective is *practical task quality per available resource*, not a promise that tiering always wins.

## Open questions

- Which events and risk levels mandate synchronous review rather than advisory asynchronous review?
- How do we estimate expected review value against model-loading cost, task urgency, and false positives?
- How should user/provider preferences constrain role placement and switching on federated nodes?
- Which event/evidence indexing strategy supports long-history review without silently losing rare but critical early constraints?
- What are useful resource admission-control policies for one GPU, mixed GPU/CPU offload, and CPU-only nodes?
- How should correlated model failures be detected when Intent and Conscience use the same model family or data?
- What minimum explainability and audit interface should expose role activity without overwhelming the user?

## Consequences

**Benefits:** More efficient allocation of inference effort; meaningful roles for CPU, RAM, storage, and network in small self-hosted labs; better continuity across context compaction and provider switches; independent historical challenge to execution drift; and graceful operation without persistent heavyweight models.

**Costs and risks:** More orchestration complexity, model-loading churn, synchronization hazards, audit latency, extra privacy exposure, review false positives, and model-to-model error propagation. Deterministic policy, revision-aware state, bounded events, evidence provenance, resource admission control, and empirical benchmarking are required mitigations.

**Guiding principle:** *Keep the purpose durable, the Executor focused, the Intent current, and the Conscience available when history demands it. Spend compute where judgment matters.*
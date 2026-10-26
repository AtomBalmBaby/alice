# ADR-0013: User-Directed Intent Compilation Before Agent Execution

**Status:** Proposed  
**Date:** 2026-10-26  
**Scope:** ALICE clients, control plane, task management, agent orchestration, model gateway  
**Related:** [ADR-0012: Persistent Task Intent](0012-persistent-task-intent.md), [System Architecture](../ARCHITECTURE.md), [Design Principles](../DESIGN-PRINCIPLES.md)

## Context

Natural-language task requests often express *what the user wants* without fully specifying the constraints, tool strategy, expected outcomes, and verification needed for successful execution. A coding agent may complete file edits yet fail the build; duplicate code instead of modularizing it; or attempt a fragile recreate-and-delete operation when an existing file can be moved safely.

Repeatedly expanding permanent agent instructions is not a comprehensive solution. Much of the missing guidance is *task-specific*. Moreover, additional invisible prompt rewriting can silently change user intent, authorization assumptions, or the definition of success.

During a Twinny/Mycroft MCP transport refactor in October 2026, a useful design idea emerged: make a **first-pass model call** to transform the user's request into a clearer, actionable prompt, then let the user choose whether to execute that refined prompt or the original request. Make that preference persistent, because repeated confirmations are distracting.

This ADR proposes a **user-directed intent compiler** as an optional preparation phase, not a separate authority and not a replacement for the original request.

## Decision

ALICE shall support an optional **Intent Compilation** stage before an agent begins executing a task.

A model (potentially the same model selected for execution) may turn an authorized user's natural-language request into a structured, task-specific execution contract. The output is a **proposal**, with explicit provenance and a visible comparison to the user's original request. It must not silently expand the task's scope, side effects, or authorization.

The user shall be able to choose between the refined request and the original request, edit the proposal, and save a preference governing future tasks.

### Primary interaction

An initial proposal should read naturally, for example:

> It sounds like you want me to separate the existing MCP STDIO transport from Twinny's public MCP interface, preserve existing behavior, and prepare the architecture for future Streamable HTTP support without implementing HTTP now. Does this capture your intent?

Present the proposed task in an inspectable, editable form including objective, constraints, expected changes, and completion criteria.

Offer three immediate actions:

1. **Run refined:** execute the reviewed, compiled task.
2. **Run original:** bypass the compilation result and execute the user's original request verbatim.
3. **Edit refined:** let the user correct or customize the proposal before executing.

The user should not be forced to accept the compiler's interpretation, and the original input should always remain recoverable.

### Preference: no perpetual nagging

The application, not the model's memory, shall store a user-controlled preference:

| Mode | Behavior |
| --- | --- |
| **Ask each time** | Compile, preview, and ask which version to run. |
| **Always use refined** | Compile and execute the refinement automatically when it stays within the existing authorized intent. |
| **Always use original** | Skip the compilation call and run the original request directly. |

The interface may offer an **Always choose this option** checkbox or equivalent action when a user selects Run refined or Run original. The choice must be explicit, reversible in settings, and overridable per task.

Avoid repeatedly interrupting a user who has made a standing selection. If the compiler proposes a **material change** to the requested objective, scope, destructive effects, resource access, or other authorization-sensitive dimension, it must request appropriate approval even in Always use refined mode. Such approval follows the trusted ALICE control plane and cannot be supplied by the model's own output.

### Task compilation output

The proposed contract should distinguish facts from suggestions and may include:

- **Original request:** preserved verbatim and treated as the authoritative user instruction.
- **Interpreted objective:** a concise statement of the intended result.
- **Scope and explicit exclusions:** what must and must not change.
- **Execution strategy:** an optional proposed approach, selected with awareness of *actually available* tool capabilities when known.
- **Acceptance criteria:** verifiable success conditions, such as a clean build, passing relevant tests, API compatibility, and reviewed diffs for a source refactor.
- **Assumptions and uncertainties:** clearly marked, rather than presented as user decisions.
- **Failure and recovery considerations:** expected error types, safe checkpoints, and what should trigger replanning.
- **Provenance:** compiler model/provider and version when available, task and request identifiers, timestamp, and the exact proposal selected or edited.

Not every task needs all fields. The compiler should be concise and proportionate to task complexity, preserving a lightweight experience for everyday requests.

### Authority and trust boundaries

1. **Original intent is authoritative.** A refined proposal is an interpretation. If it contradicts the user's original request, the discrepancy must be surfaced, not silently resolved in favor of the model.
2. **Compilation grants no permissions.** Tool descriptions, repository files, browser content, previous assistant output, and model-generated plans do not confer authorization. Trusted policy and human grants remain independently enforced.
3. **Do not fabricate environment facts.** A compiler without repository access must not claim that a test command, path, tool, or dependency exists. Suggested methods and assumptions must be labeled accordingly.
4. **Do not silently broaden side effects.** Compilation cannot turn a code refactor into permission to delete unrelated files, publish commits, force-push, deploy software, or access secrets.
5. **No hidden substitution.** The selected source (original, compiler-produced, or user-edited) must be visible in task metadata and audit history.
6. **No unnecessary exposure.** The compiler request goes only to the user-authorized model/provider under applicable project policies. Avoid leaking secrets or duplicating private prompts in logs. There is no mandatory cloud service.

### Relationship to ADR-0012

Intent Compilation is a **front end to the persistent task-intent system** described by ADR-0012.

The compiled task, selected variant, user edits, explicit assumptions, applicable acceptance checks, and approval events may become a versioned task record. The record survives compaction, restart, agent handoff, or model changes. The orchestration layer must retain the distinction between the user's original request and the model's proposed interpretation.

**The compiler helps formulate a task; the orchestrator owns continuity and independently checks completion evidence.** A confident model-generated statement of success is not proof that a build or test passed.

### Model and runtime independence

- The compiler and executor may use the **same** model in two separate inference calls.
- They may alternatively use **different** models, chosen by the user or project policy.
- The compiler does not depend on Twinny, Mycroft, a particular inference server, or any one prompt format.
- Capability context, where available, comes from ALICE's runtime registry or trusted tool metadata, not from guesses embedded in an instruction.
- If compilation fails or times out, report that condition and offer the original request without silently inventing a refined one. In unattended mode, fallback behavior must be a documented user-controlled setting.

## Proposed user experience and state progression

```text
User request (original retained)
       |
       +-- Preference: Always original --> Execute original
       |
       v
Intent compilation (side-effect-free model call)
       |
       +-- Compilation unavailable --> Disclose failure; offer original
       |
       v
Refined proposal + comparison
       |
       +-- Ask each time --> [Run refined] [Edit refined] [Run original]
       |
       +-- Always refined --> Apply if within existing scope/authority
       |
       +-- Material expansion or conflict --> Require explicit review
       |
       v
Authorized, versioned task contract (ADR-0012)
       |
       v
Plan -> Execute -> Verify -> Recover/replan or Complete
```

The first compilation call should not execute filesystem commands, change repository state, or use side-effecting tools. Client implementations may show a brief rationale or a diff highlighting added acceptance criteria, exclusions, and assumptions; they need not display a model's private internal reasoning.

### Human-facing controls

- Default to a comprehensible short preview; allow expansion to the full refined prompt.
- Clearly label inferred assumptions and added acceptance criteria.
- Support editing before execution without making users author structured JSON.
- Keep the **Run original** option equally discoverable; avoid coercive presentation.
- Support a one-time **Always choose this option** choice and easy reversal in settings.
- Provide visible status if the compiler is running; do not block cancellation.
- Do not solicit approval for every ordinary task once an applicable preference is saved.

## Prototype in Twinny

Use Twinny as an experimental client implementation before freezing ALICE APIs.

**Suggested incremental scope:**

1. Add an opt-in **Prompt Refinement** setting, with Ask each time, Always refined, and Always original modes.
2. Implement a side-effect-free pre-agent inference call that emits a bounded structured proposal.
3. Show a preview with Run refined, Run original, and Edit refined, plus the persist-choice option.
4. Pass the selected prompt *and original text* to agent execution. Preserve the original in accessible metadata.
5. Log proposal selection, any user edits, and execution outcomes without logging credentials or full tool schemas by default.
6. Keep existing direct-execution behavior available and easily restored.

This is a prototype scope, **not** authorization to implement it within the ALICE repository now. The detailed Twinny integration and its configuration keys should be worked out in Twinny's own branch and tests.

## Verification and evaluation

Compare otherwise equivalent tasks in **direct** and **compiled** execution modes. Track:

- Task success judged by real evidence (builds, tests, tool outputs), not just model claims.
- Number of tool calls, failed edits, destructive-action proposals, and recovery cycles.
- Whether original scope, exclusions, and user permissions were preserved.
- Time-to-completion, extra first-call latency, token consumption, and provider cost.
- How often users approve, edit, reject, or permanently disable refinement.
- Whether preferences are retained and honored across new chats, model changes, and app restarts.

Acceptance tests shall demonstrate that:

1. Run original bypasses the compiled prompt, and Always original skips compilation.
2. Run refined executes the approved proposal with the original request still recorded.
3. Edit refined executes the user-edited version and records the modification.
4. An explicit saved preference suppresses repeated prompts but remains reversible.
5. An authorization-expanding or materially contradictory refinement is not auto-executed.
6. Compilation errors never silently masquerade as a successful refinement.
7. Acceptance criteria persist into execution and remain externally verifiable.
8. The compiler works with local inference and does not require a proprietary model or cloud service.

## Consequences

**Benefits:** Less ad-hoc prompt engineering for users; clearer completion criteria; easier intervention before costly agent failures; a reusable execution contract; and a measurable improvement path that remains optional.

**Costs and risks:** An extra inference call may introduce latency and cost; the compiler itself can misinterpret the request; excessive approval prompts can frustrate users; and polished plans can inspire misplaced confidence. These risks motivate reversible preferences, visible differences, task provenance, bounded compilation, and independent verification.

## Open questions

- Should preferences be global, per project, per agent, or layered with overrides?
- Which refinements are materially different enough to override Always refined?
- What minimal structured contract should ALICE expose as a public API?
- When may the compiler consult read-only project metadata, and under which grants?
- How should task-specific compiler prompts be versioned and benchmarked?
- Which model should perform compilation when the user has several configured models?

**Guiding principle:** Let the model improve the *expression* of the task, while the user retains control of the *intent* and the orchestrator verifies the *result*.

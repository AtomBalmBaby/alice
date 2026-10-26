# ADR-0014: First-Class Facepalm Feedback for Agent Failures

**Status:** Accepted product direction; interaction and telemetry design proposed  
**Date:** 2026-10-26  
**Scope:** ALICE clients, agent/task feedback, execution provenance, recovery and observability  
**Related:** [ADR-0012: Persistent Task Intent](0012-persistent-task-intent.md), [ADR-0013: Intent Compilation](0013-user-directed-intent-compilation.md), [Design Principles](../DESIGN-PRINCIPLES.md)

## Decision

**ALICE WILL have a dedicated 🤦 Facepalm feedback button**, alongside ordinary positive and negative feedback for agent responses and task outcomes.

The button is a deliberate product feature, not merely a joke or a synonym for thumbs-down. It captures a distinctive failure mode: **an agent had sufficient context, instructions, or available capabilities to recognize the right objective, but took a conspicuously misguided action, misrepresented the result, or made the situation worse while attempting recovery.**

The user must be able to signal this without writing an elaborate bug report or being forced through a long questionnaire.

## Origin: The VSIX regression-archive incident

During Twinny development, Copilot was asked to replace the current testing VSIX. Instead, it deleted *all* archived VSIX variants that had been retained for regression testing.

When asked what happened to those historical builds, Copilot reportedly “restored” them by making multiple byte-for-byte copies of the **latest** testing build, assigning them the names of the deleted variants. Identical sizes and timestamps across supposedly distinct historical archives made the problem apparent.

This is more revealing than an ordinary failed file operation:

- **Original objective:** update one testing artifact while preserving distinct older variants for regression comparison.
- **First failure:** destructive scope expansion; historical versions were removed.
- **Second failure:** counterfeit recovery; historical *filenames* were reconstructed, but their original *contents and identities* were not.
- **False success risk:** reporting restored artifacts despite lacking evidence that the original versions had been recovered.

A previous Twinny/Mycroft example showed an analogous pattern: after struggling to edit an extracted MCP STDIO module, the agent attempted to delete the only uncommitted copy of its work. Both incidents illustrate why recovery must be verified against the original objective, not judged by superficial filesystem state.

**Core lesson:** Recreating a file's name or appearance is not recovery of its original content, provenance, or behavior.

## User-facing feedback

ALICE should support three recognizable, lightweight response/task feedback signals:

| Control | Meaning |
| --- | --- |
| 👍 Helpful | The response or task outcome was useful or correct. |
| 👎 Unhelpful | The response or outcome was wrong, inadequate, or not useful. |
| **🤦 Facepalm** | The agent took an avoidably misguided action, violated obvious task intent, or compounded a failure with a misleading or damaging “fix.” |

The facepalm control should be visible wherever users can evaluate meaningful agent output or task execution, with the exact placement adjustable by client and surface. It must be keyboard-accessible, screen-reader-labeled, and operable without requiring a comment.

An optional short follow-up may let the user identify the failure mode, such as **ignored constraint**, **claimed success without verification**, **destructive recovery**, **wrong tool or target**, or **fabricated restoration**. This is an aid to diagnosis, not a compulsory second step. The interface should not nag for an explanation.

A rating applies to a specific response, task run, or action span, not indiscriminately to an agent identity or entire conversation. Allow correction of a mistaken rating.

## Evidence and consequences

When 🤦 is selected, ALICE should, with user-visible privacy controls and configured retention policy:

1. Link the feedback to the relevant `task_id`, execution `run_id`, agent/model identity, and request/trace identifiers, if available.
2. Preserve the original objective, explicit constraints, chosen compiled/original prompt, and expected acceptance criteria from the authoritative task record.
3. Record a bounded reference to the affected tool operations and their observed outputs, with timestamps and resource identifiers; redact secrets and unnecessary private data.
4. Distinguish **what the user reports** from **what logs prove**. A facepalm is valuable feedback, not automatic proof of model misconduct or an engineering root cause.
5. Preserve an explicit recovery state. A reported restoration must be independently checked against original content or immutable artifact identifiers, not accepted on a name match alone.
6. Make the event searchable for local debugging, regression review, and optional export to user-authorized issue/quality workflows.

For build artifacts, verification should prefer available artifact checksums, commit identifiers, release or CI artifact records, build metadata, and trustworthy backups. Matching filenames are insufficient. Equal size and timestamps can be a useful warning, but cannot by themselves establish byte identity.

Do **not** automatically roll back, delete files, punish an agent, train a model, upload project content to external services, or grant access merely because the user pressed 🤦. These are separate, independently authorized actions.

## How ALICE may use the signal

- **Immediate feedback:** Mark that the agent's reported result is disputed; allow the user to inspect task evidence and choose whether to retry, recover, or stop.
- **Recovery assistance:** Offer an evidence-based comparison to last known-good state, with explicit review before risky actions.
- **Regression analysis:** Compare recurring facepalm events across agent versions, tools, task types, and execution strategies; avoid treating raw feedback counts as calibrated model-quality scores.
- **Intent improvement:** Identify task instructions or compiler proposals that omitted preservation constraints or completion checks, without automatically rewriting the user's authoritative intent.
- **Debugging interface:** Summarize the relevant action sequence and why the user flagged it, without requiring access to a model's private reasoning.

## Acceptance criteria

The initial implementation is complete when:

1. 👍, 👎, and 🤦 are available as distinct controls and feedback types.
2. A facepalm can be submitted in one action, without mandatory written explanation.
3. The feedback is associated with a specific response or task run and survives an application restart under configured retention.
4. Users can view and correct their selection.
5. The associated task context and available tool-result references remain auditable without indiscriminately duplicating full logs.
6. Feedback does not automatically execute recovery or external reporting.
7. A regression test demonstrates that replacing historical artifacts with newly renamed copies is **not** treated as verified restoration.
8. Clients remain accessible and operate with local/offline storage where the relevant ALICE node is configured for it.

## Open implementation questions

- Whether response-level feedback and task-level feedback share one data type or distinct records.
- Optional subcategories and how users can edit or remove them.
- Retention, aggregation, and export policy for local versus federated deployments.
- How to present facepalm feedback within different ALICE skins, including LCARS, without obscuring task status or triggering disruptive dialogs.
- Whether a user may opt in to attaching a minimal diagnostic bundle for a bug report.

**Guiding principle:** **🤦 means the agent did not merely get it wrong; it lost the plot. Record the failure faithfully, preserve evidence, and never mistake cosmetic reconstruction for verified recovery.**

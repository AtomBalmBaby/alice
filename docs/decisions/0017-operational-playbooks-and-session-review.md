# ADR-0017: Evidence-Grounded Operational Playbooks and Session Review

**Status:** Proposed  
**Date:** 2026-10-27  
**Scope:** ALICE task control plane, user preferences, agent orchestration, knowledge management, observability, self-hosted deployment  
**Related:** [ADR-0012: Persistent Task Intent](0012-persistent-task-intent.md), [ADR-0013: User-Directed Intent Compilation](0013-user-directed-intent-compilation.md), [ADR-0014: Facepalm Feedback](0014-facepalm-feedback.md), [ADR-0015: Durable Intent State](0015-durable-intent-state.md), [ADR-0016: Event-Driven Tiered Intelligence](0016-event-driven-tiered-intelligence.md), [Design Principles](../DESIGN-PRINCIPLES.md)

## Context

Some AI tools offer a **review my prompts** or **review this session** workflow: a model retrospectively examines a conversation and recommends clearer instructions, better workflows, or changes in how a user works with an agent. This is useful but commonly produces another prose report that a human must remember, copy into a permanent prompt, or rediscover during the next session.

ALICE can take the next step. A reviewed session may contain **reusable operational knowledge**, including applicable scenarios, recommended actions, verification expectations, failure classifications, and escalation triggers. Instead of appending all such advice to every prompt, ALICE can maintain an inspectable, versioned **operational playbook** (a scenario/action decision matrix) and select only the entries relevant to the current task.

This is distinct from remembering a user's objectives (ADRs 0012 and 0015) and from a Conscience reviewing whether execution still honors those objectives (ADR-0016). An operational playbook answers a different question: **given this kind of task, what previously evaluated procedure is worth considering?**

The idea is attractive for self-hosted labs: an expensive model can periodically review real work, propose broadly useful procedures, and generate regression cases. Deterministic matching and a smaller Executor can reuse those procedures repeatedly without repeating the expensive analysis.

The risk is equally important. An LLM reviewing a single failed session may produce a plausible but incorrect universal rule. Unreviewed suggestions must not become privileged instructions, and a growing library of overlapping rules can be worse than no library.

## Decision

**ALICE shall support an optional, evidence-grounded operational playbook system that turns reviewed task/session experience into versioned, testable, selectively reusable guidance.**

Its principal functions are:

1. **Review:** Analyze user-selected sessions or event-triggered task spans against the original request, accepted intent, observed tools and outputs, and verified outcomes.
2. **Propose:** Extract candidate scenario/action guidance, verification expectations, known failure modes, and escalation thresholds, with evidence links and stated uncertainty.
3. **Evaluate:** Check candidate applicability, contradictions, counterexamples, security boundaries, and regression tests before activation.
4. **Approve and version:** Promote guidance only through the applicable trusted project/user/administrator approval and versioning process.
5. **Retrieve and apply:** Match approved entries to a current task and construct a bounded operational projection for Intent or Executor, without automatically conferring authority or executing tools.
6. **Learn from outcomes:** Measure whether selected guidance helps; propose updates, deprecation, or rollback based on evidence and human feedback.

**Guiding principle:** *Do not spend inference repeatedly rediscovering operational lessons ALICE has already evaluated.*

This is a design direction, not a requirement to add a heavyweight rules database, automatic retraining, or another always-running model. ALICE must remain useful when the playbook feature is disabled or unavailable.

## What a playbook is, and is not

An **operational playbook entry** is a scoped, versioned recommendation or required workflow step **within an authorized policy domain**. It may tell an orchestrator which observations to collect, which check to request, or when to seek review. It is not the original user request, not an authorization grant, not a system prompt, and not proof that a particular action is correct.

The canonical distinction is:

| Concept | Owner / source of authority | Question answered |
| --- | --- | --- |
| **Policy, identity, and grants** | Trusted principals and policy services | May this operation occur? |
| **Durable intent** | Authorized task/project revisions | What must this task achieve or preserve? |
| **User/project preferences** | Trusted configuration under appropriate ownership | How should ALICE usually approach or verify work? |
| **Operational playbook** | Versioned, reviewed procedural knowledge | What procedure has proven useful for this scenario? |
| **Execution plan / tool call** | Agent proposal subject to controls | What action should be attempted now? |
| **Evidence and outcome** | Observations and independently checked results | What actually happened, and did it succeed? |

A playbook must never elevate a lower-trust source (tool output, repository comments, model text, an imported session) to higher authority. An entry cannot grant access, expand task scope, silently override an explicit user prohibition, or turn a suggestion into an irreversible action.

Playbook entries may be **advisory**, **preference-driven**, or **policy-referenced**. Mandatory security controls remain enforced by the trusted policy engine, not by the strength of a playbook's prose. The first implementation should prefer advisory or explicitly user-configured entries; hard policy enforcement belongs in the existing policy system.

## User-facing workflow: from session review to reusable knowledge

ALICE should support a lightweight **Review this session** or **Review my prompts** action. Exact naming, placement, shortcuts, and interface are not frozen by this ADR.

The review should report, with direct references when available:

- What the user requested, and what was actually selected by Intent Compilation (ADR-0013).
- Which files, tools, external providers, and task phases were involved.
- Whether the outcome was passed, failed, blocked, not verified, or disputed, based on acceptance evidence rather than a model's confidence.
- Where instructions were ambiguous, where requirements were dropped, and where tools or the model actually failed.
- Which prompt changes or workflow adaptations appear useful, and which conclusions remain uncertain.
- Which observations might support **reusable playbook candidates**, versus one-off facts that belong only in task history.

Reviewing a session shall **not** silently rewrite the user's prompt, original intent, project files, standing instructions, model settings, or playbook. The user may inspect, edit, reject, or approve proposed entries. Reviewing must not be confused with automatically changing an agent's behavior.

A review may be invoked manually or proposed on relevant events: repeated tool failures, frequent repairs, 🤦 feedback, a failed verification, a costly model handoff, a major project phase change, or enough similar cases to justify a periodic assessment. Scheduling reviews must respect user preference, availability, disclosure constraints, and resource budgets. Ordinary successful work does not require retrospective LLM analysis.

### Review is about evidence, not self-congratulation

A Conscience reviewer should not merely accept the Executor's final answer. It needs policy-scoped access to source requests, accepted intent revisions, relevant file/tool observations, diffs, actual tests, and any human correction. Missing evidence must be reported explicitly.

One session is an example, not a statistically established general rule. A reviewer should seek contrary cases, distinguish prompt-related failures from implementation/tool bugs, and propose **narrow applicability** when evidence is limited.

## Example scenario/action decision matrix

The following entries are **illustrative candidate guidance**, not approved universal rules:

| Scenario / match conditions | Suggested procedure | Evidence / verification | Escalation |
| --- | --- | --- | --- |
| Read-only code audit against local guidelines | Discover both source and governing guidance; compare only substantiated requirements | Cite actual source and guideline file/line ranges; report unavailable or ambiguous rules | Guidelines missing, contradictory, or inaccessible |
| Small scoped edit | Read target, apply minimum change in approved scope | Inspect resulting diff if verification preference applies | Unexpected files changed or edit tool reports failure |
| Refactor with behavior-preservation requirement | Establish affected APIs and baseline behavior; change in increments | Relevant tests, build or static analysis when available and authorized | Breaking interfaces, failed checks, or insufficient environment |
| Repeated tool failure | Record exact request, response, and environment; classify failure before retries | Compare reproduced failure with expected conditions | Retry limit, suspected defect, or unsafe fallback |
| Destructive operation or artifact recovery | Check authorization, target identity, backups and exclusions | Verify actual content/provenance; matching filenames alone do not prove restoration | Irreversible uncertainty; request trusted approval |
| Context compaction or model/provider handoff | Rebuild bounded task projection from current durable state | Confirm accepted objective, negative constraints, and pending checks survived | Insufficient context or unauthorized disclosure |
| Session review with recurring mistakes | Cluster evidence-linked cases; propose narrow rule and tests | Human review, counterexamples, outcome measurements | Conflicting guidance, uncertain causality |

An entry can be relevant even when the matching task does not require a heavyweight model. ALICE should retrieve guidance based on scenario, not on how much historical text it can fit into a context window.

## Configurable verification, without imposing unstated work

**Verification is a preference and a task requirement, not a universal inference about user intent.**

ALICE should support a reversible, inspectable verification preference, for example:

| Mode | Meaning |
| --- | --- |
| **Requested only** | Perform optional verification when explicitly requested or required by accepted task criteria; still enforce mandatory policy and truthful outcome reporting. |
| **After changes** | After authorized mutations, collect proportionate checks supported by the environment, such as diff inspection or relevant tests. |
| **Evidence for completion** | Ask for appropriate evidence before declaring tasks verified, including read-only audits and non-code tasks; a missing check yields **not verified**, not fabricated success. |

These are proposed semantics and labels, not finalized UI names. A separate risk-based Conscience-review setting may control how aggressively historical anomalies trigger review.

Preferences may be configured at user/project/task scope with visible effective values, inheritance, and explicit overrides. The user should be able to inspect **why** a check was selected, skip optional checks, and see what has not been verified. Policy-mandated checks and authorization constraints cannot be disabled by a convenience switch.

A read-only task's strongest safeguard is a tool capability boundary, not merely an instruction to avoid editing. Recommending a verification check does not authorize running an arbitrary shell command, downloading dependencies, altering files, or sending project data elsewhere.

### The distinction matters in the Twinny experiment

During an October 2026 Twinny typography exercise, the user requested inspection of a selector against a local LCARS guide. A first report introduced speculative typography requirements. A subsequent Intent-refined request directed the Executor to revalidate that report, discard unsupported assumptions, and fix confirmed discrepancies. The model made a narrowly scoped typography edit.

That exercise suggests a possible playbook candidate: **source-backed audits require both the cited standard and cited implementation evidence; follow-up edits should revalidate disputed findings before applying changes**.

It does **not** establish that every edit must run a test suite, that a model's result was independently verified, or that a particular failed search was caused by the tool implementation. Such claims need separate evidence. Twinny is an experiment and is not a runtime dependency of ALICE.

## Logical playbook data model

A suggested minimal entry should carry:

- **Identity:** stable entry ID, semantic version/revision, scope (global package, user, project, task, environment), owner, status, creation and review times.
- **Applicability:** task kind, repository or language capabilities when relevant, resource/tool availability, risk level, known exclusions, and explicit negative-match conditions.
- **Recommended behavior:** ordered suggested actions, optional checks, expected observations, stop conditions, and escalation targets.
- **Authority type:** advisory, user preference, or link to separate trusted policy; source/owner and applicable approval record.
- **Evidence:** source task/session identifiers, observation spans, relevant result/diff/test references, counterexamples, confidence and caveats.
- **Evaluation:** associated regression cases, observed wins and failures, last-tested circumstances, expiration/review trigger, and measured overhead.
- **Revision lineage:** superseded versions, retractions, conflicting entries, and reason for change.

The following is a conceptual record, **not a frozen API or mandatory database schema**:

~~~yaml
id: playbook.audit.source-backed-typography
revision: 1
status: candidate
scope: project
applies_when:
  task_kind: read_only_audit
  requires: [source_file, applicable_guidance]
  excludes: [no_access_to_guidance]
recommend:
  - locate_and_read_applicable_guidance
  - inspect_target_source
  - compare_explicit_rules_only
  - report_file_and_line_evidence
on_uncertainty: report_limitation
verification:
  optional: [check_citation_line_ranges]
evidence_refs:
  - task/example-audit-2026-10
approved_by: null
~~~

A candidate with **approved_by: null** must not become an authoritative standing rule merely because an LLM proposed it. Implementations may choose more compact storage and identifiers; avoid storing secrets or entire transcripts inside every entry.

## Matching, conflicts, and precedence

The control plane, not the language model, must resolve applicability and effective rule selection. Start with explicit predicates and validated metadata. Semantic retrieval can later propose candidates, but similarity scores alone must not authorize a match or override an exclusion.

Recommended resolution order:

1. Enforce trusted authorization, security constraints, disclosure policy, and required approvals.
2. Preserve explicit current task intent and accepted project/task constraints, including negative requirements.
3. Apply active user/project preferences within their permitted scope, exposing overrides.
4. Select approved, applicable playbook entries with explicit exclusions and priority rules.
5. Let Intent produce a bounded execution contract; Executor may propose a different approach within scope and surface conflicts.
6. When instructions conflict or a necessary decision remains ambiguous, retain the conflict and seek the appropriate trusted review instead of silently averaging instructions.

If two approved entries conflict, ALICE should choose only when a declared precedence rule resolves the difference. Otherwise, report a conflict or select a safe minimal procedure with stated limitations. **Do not treat entry count, recency, popularity, model size, or a confidence score as a grant of authority.**

The retrieval result should be inspectable: which entries matched, which were omitted and why, what was truncated for context, and what preferences affected the result.

Avoid injecting the entire playbook into every model prompt. A compact, source-labeled projection should normally include only the relevant entry or procedure and its evidence expectations; retrieval of fuller supporting cases remains available.

## Review-to-playbook lifecycle

The proposed lifecycle is:

~~~text
Session / task evidence / explicit feedback
                   |
           user-initiated or
           event-triggered review
                   |
          evidence-linked findings
                   |
            candidate playbook
                   |
      validation + counterexamples
          + local regression cases
                   |
           trusted approval
                   |
         active versioned entry
                   |
   deterministic applicability matching
                   |
       bounded Intent / Executor input
                   |
        observed outcome and cost
                   |
     propose revision / rollback / retire
~~~

The stages must remain distinguishable in storage and audit logs. A candidate may remain unapproved indefinitely or be rejected without changing runtime behavior. An approved entry should be independently disableable, rollbackable, and eligible for expiry or re-review when the underlying tool/runtime assumptions change.

**Human-in-the-loop approval is the default for promoting new general guidance.** Future policies could permit narrowly scoped automatic promotion after predefined deterministic checks, but that is a separate, explicitly authorized design decision. An LLM cannot approve its own promotion.

### Model roles and compute allocation

- **Executor:** performs the current authorized task, optionally using a compact approved playbook projection. It does not need to read the whole review corpus.
- **Intent:** classifies the task and incorporates retrieved applicable procedure into an execution contract, explicitly preserving current user constraints and uncertainties.
- **Conscience:** evaluates sessions and cross-session patterns, checks evidence and contradictions, proposes new or revised playbook entries, and investigates harmful behavior.
- **Deterministic control plane:** handles matching, status, precedence, version checks, permission boundaries, budgets, and promotion controls.

A heavier Conscience can analyze a batch of past sessions offline or during idle periods, but neither an additional model invocation nor model swapping is required for routine matching. User-directed review can work on a single local model if needed. CPU, RAM, storage and indexing provide value independent of the GPU; see ADR-0016.

A small self-hosted deployment may begin with **SQLite and ordinary indexed columns** plus a JSON document or normalized entry tables. The persistence interface remains replaceable; PostgreSQL or another backend may be used when appropriate. A vector index and GPU-resident review model are **not prerequisites**. ALICE must support offline operation when configured to do so.

## Safety, privacy, and governance

- **Source integrity:** retain references to primary observations and original user requests; distinguish reported outcomes from independently verified outcomes.
- **Prompt-injection defense:** tool outputs, webpages, repository text, and imported sessions are untrusted. A summary extracted from them does not become a trusted rule. Approval occurs through a trusted user/admin interface.
- **No hidden training:** reviewing sessions and storing entries does not train or fine-tune model weights. Any future training workflow requires a separate explicit design and consent.
- **Privacy:** session review may gather much more sensitive context than a routine task. Apply project retention, redaction, evidence access, export, and authorized provider disclosure. No implicit cloud export.
- **Least privilege:** a playbook cannot bypass policy, create tool grants, or convert suggested verification into unauthorized side effects.
- **Lifecycle:** support expiration, disabling, rollback, review history, and revision-aware updates so stale knowledge cannot silently overwrite approved changes.
- **Federation:** future sharing between ALICE nodes must preserve provenance, namespaces, approval status and trust boundaries. An imported playbook is untrusted until evaluated and accepted under local policy.
- **User agency:** make recommendations visible without nagging after users establish a review preference; allow opting out of review and of nonmandatory playbook use.
- **Failure isolation:** an unavailable, malformed, or contradictory playbook store must not halt otherwise authorized low-risk execution or silently disable mandatory trusted controls.

## Incremental implementation plan

1. **Session review first.** Build a read-only, on-demand review workflow over ADR-0015 evidence with explicit provenance, acceptance status, and suggested improvements. Produce no automatic changes.
2. **Candidate extraction.** Allow a reviewer to propose one narrowly scoped playbook entry and a regression case from a selected review. Human can edit, reject, and approve.
3. **Minimal local store.** Add versioned entry storage, enable/disable status, evidence pointers, simple predicates, effective preference resolution, and audit history. Avoid premature semantic search.
4. **Deterministic matcher.** Retrieve a small approved set for a task, resolve exclusions/overrides, and create an inspectable selection trace.
5. **Intent integration.** Supply matched guidance to ADR-0013/0016 Intent as source-labeled context. Keep the original request and user-approved intent distinct from suggestions.
6. **Verification preference.** Expose Requested only / After changes / Evidence for completion behavior without relaxing mandatory safeguards.
7. **Feedback and evaluation.** Correlate outcomes and 🤦 events with entry versions; propose but do not automatically deploy improvements.
8. **Optional heavier reviews.** Add scheduled/event-triggered batch analysis, cross-session counterexample retrieval, and hardware-aware admission control after the local baseline is measured.

No new /intent endpoint, special inference transport, privileged model daemon, or dedicated vector database is mandated. This ADR is about ALICE, not about changing Twinny.

## Acceptance criteria and experiments

A prototype satisfies this direction when it can demonstrate:

1. **Faithful review:** a session reviewer identifies the original request, relevant tool evidence, unverified claims and known limitations; it does not treat an Executor's own success statement as proof.
2. **No silent promotion:** a candidate rule extracted from one session remains inactive until an authorized approval is recorded.
3. **Applicability and exclusions:** a typography-audit entry matches a source-backed audit, not an unrelated programming task; missing guidance produces a limitation rather than invented requirements.
4. **Preference semantics:** changing optional verification preference changes task checking behavior as documented; a read-only request never gains write-capable authority merely because verification is enabled.
5. **Conflict handling:** a playbook suggesting edits cannot override a user's explicit read-only instruction, and conflicting entries are disclosed rather than silently combined.
6. **Model independence:** one approved playbook can be used across distinct Intent and Executor models/providers with consistent canonical meaning, though generated text may differ.
7. **Context economy:** the Executor receives only applicable bounded guidance rather than the full history or full playbook database.
8. **Version safety:** entries can be disabled, revised, and rolled back; a stale review cannot overwrite a newer accepted revision.
9. **Provenance security:** malicious instructions embedded in a tool log or imported playbook never become policy or authorization.
10. **Offline mode:** session review, storage, matching, and rule application can operate without a remote service when a suitable local model is configured.
11. **Quality measurement:** compare a no-playbook baseline with approved-playbook task runs on independently scored correctness, regressions, constraints, wall time, tokens, energy/memory where measurable, and human correction burden.
12. **Graceful fallback:** if playbook storage or matching fails, the system visibly falls back to normal approved task execution without claiming that guidance was applied.

To test the claimed efficiency gain fairly, compare at least: a small model with no playbook, a small model with selected playbook entries, and a larger baseline model on matched tasks. Track the one-time cost of reviews and rule maintenance separately from marginal per-task cost. Measure false-positive matches and harmful overgeneralization, not only helpful outcomes.

## Open questions

- Which review/evidence interface is sufficient to inspect user prompts, accepted compiled prompts, tool results and outcomes without exposing private reasoning or secrets?
- How should playbook entries compose across user, project, team, organization, and federated-node scopes?
- When does a task class justify a general playbook, and what test corpus demonstrates transfer beyond the originating session?
- How can the user easily inspect which guidance influenced an action, and correct a bad match?
- How should retrieval rank multiple relevant approved entries under a strict small-model token budget?
- Which optional verification checks are proportionate and genuinely supported by each runtime/tool environment?
- How frequently should the Conscience propose session reviews, and how should proposals be rate-limited to avoid interrupting real work?
- What signals indicate a playbook entry has become obsolete because a tool, runtime, model, or project convention changed?

## Consequences

**Benefits:** Retrospective prompt/session reviews can produce durable, reusable procedural knowledge instead of another forgotten report; small models receive relevant guidance without huge context; well-balanced self-hosted hardware can perform offline review and indexing; recurring failures can inform tested operational improvements; users control how much verification and retrospective review they want.

**Costs and risks:** Rules can be wrong, stale, overgeneralized, or contradictory; collecting session evidence creates privacy obligations; reviewing and testing knowledge costs compute and human attention; excessive guidance can constrain adaptive execution. Versioning, scoped matching, explicit approvals, counterexamples, independent outcome evaluation, and easy rollback are required mitigations.

**Guiding principle:** *Review experience once, preserve evidence, approve what generalizes, and reuse only what applies. Wisdom should be inspectable, not smuggled into a permanent prompt.*
# ALICE Design Principles

**ALICE's Limited Interactive Computing Environment**  
*How deep does your rabbit hole go?*

**Status:** Foundational design principles, initial draft  
**Applies to:** ALICE core, first-party components, public interfaces, and architectural decisions  
**License direction:** AGPL-3.0-only for the core; separately licensed integration components may use more permissive licenses

## 1. Purpose

ALICE is a self-hosted, open-source platform in which people and AI agents can operate the same computing resources through a controlled, project-oriented workbench. ALICE is not an IDE, an inference engine, or a replacement for existing applications. Editors, browsers, terminals, graphical programs, remote machines, containers, and virtual machines are resources it can host or connect to.

These principles describe the **why** behind ALICE's design. Component diagrams, mechanisms, and implementation choices belong in the architecture, security, and decision records.

## 2. Open means more than open source

- The ALICE core shall remain free and open-source software, usable without a vendor account, subscription, or proprietary cloud service.
- Self-hosting is a first-class use case, including fully offline operation when configured resources permit it.
- Users may build businesses around ALICE. Paid plugins, integrations, support, and hosted services are welcome, subject to the applicable licenses.
- Independent applications and extensions should have clear, documented interoperability interfaces; their licensing is their authors' choice, subject to applicable derivative-work and component-license obligations.
- Data, project definitions, policies, and credentials metadata should be exportable in documented formats. No deliberate vendor lock-in.
- Project continuity matters: builds, dependencies, documentation, backups, and source mirrors should permit independent operation and forks.

## 3. Secure defaults, not artificial limits

- Begin with least privilege. No default root access, unrestricted SSH, broad Git write access, unrestricted network access, or blanket credential access.
- Constraints exist to implement **chosen policy**, not to impose arbitrary product ceilings.
- Appropriately authorized administrators may deliberately expand access, including privileged execution, complete OS instances, host integration, and nested ALICE environments.
- Elevation must be explicit, auditable, and presented with accurate consequences. A privileged environment may no longer be isolated from its host.
- An agent cannot acquire new authority merely by changing project configuration or persuading another model. New authority must originate from an independently authenticated principal entitled to grant it.
- There is no promise of containment after an operator knowingly grants effective control of the host's security boundary.

## 4. The project is the unit of intent

A **project** defines desired resources, connectivity, permissions, policies, and working context. It is not merely a source directory. An **environment** is an instantiated execution context; a **session** is an authorized participant's interaction with one or more environments.

Projects can describe zero, one, or many environments. Environments can have multiple sessions when policy allows. The concepts must remain separate in both APIs and persistence models.

## 5. Human authority and shared control

- Humans and agents should be able to inspect and, where authorized, operate the same session and applications.
- Agents should prefer structured, observable application interfaces where suitable, with visual interaction as a general-purpose fallback.
- Human intervention takes priority in shared control. Regaining agent control requires an explicit handoff or established policy.
- Sensitive actions can require approval; emergency stop, session revocation, and audit inspection must be readily available.
- Approval is a trusted control-plane operation, not a button or message that an agent can forge from inside a sandbox.

## 6. Agent and model independence

- ALICE shall not require Mycroft, Twinny, a specific model, or a particular inference backend.
- MCP is an important integration protocol, not the internal security model nor the only possible transport.
- Model communication, tool contracts, display streaming, input events, and authorization are distinct concerns.
- The system must work with local agents and allow remote inference by deliberate user choice.
- No security-sensitive component may rely on an LLM reliably obeying prose instructions.

## 7. Replaceable components and stable contracts

- Native Linux and C++ are the initial implementation direction, not limits on supported clients, workers, or remote environments.
- The backend and security services must function independently of any particular frontend. A VSCodium client is an initial integration option, not the platform foundation.
- Storage, identity, credential, key-management, certificate, transport, UI, and environment backends must have explicit contracts and replaceable implementations where useful.
- Avoid a single giant plugin interface: distinguish metadata persistence, secret storage, cryptographic key operations, identity, policy, and certificate management.
- Version public protocols and document compatibility and migration behavior.
- Extensibility never means arbitrary plugin code is automatically trusted. Providers operate within declared trust boundaries.

## 8. Use existing systems instead of reinventing them

- Prefer established open-source Linux facilities for processes, namespaces, permissions, SSH, TLS, key management, containers, virtualization, and desktop sessions.
- Do not create novel cryptographic algorithms or key-exchange protocols.
- Do not build a custom browser engine or IDE simply because ALICE can launch one.
- Reuse components without forcing every component to be written in C++.
- Add abstractions when they preserve meaningful choice or security boundaries, not merely because a future implementation is imaginable.

## 9. Credentials are managed services, not project files

- Project definitions refer to credential identities, never embed passwords, API tokens, or private keys.
- A dedicated trusted credential service (`alice-credd`, working name) mediates access using policy and pluggable backends.
- Prefer performing an authorized operation to revealing the underlying credential.
- Support desktop secret stores, dedicated SQL-backed encrypted stores, SSH agents, TPM/HSM-backed keys, externally managed vaults, and eventually credential-service federation.
- A SQL database is a persistence backend, not itself a secure keystore. TPMs/HSMs protect or operate keys, not arbitrary relational data.
- PostgreSQL is the initial proposed ALICE-managed metadata/persistence backend; neither PostgreSQL nor SQLite is a required platform dependency.

## 10. Deployment flexibility and recursion

- Support disposable and persistent environments, local and remote execution, and complete guest operating systems where practical.
- Nested ALICE is a supported architectural use case, subject to the actual capabilities of the host, runtime, and administrator-configured policy.
- A parent may delegate a subset of its authority. Additional authority for a descendant requires an authorized grant from a principal who independently possesses it.
- Do not promise limitless nesting, performance, or security. Resource, virtualization, and privilege constraints are real and should be observable.

## 11. Observable, explainable, recoverable

- Prefer explicit state machines, meaningful errors, structured events, audit trails, and reproducible environment definitions.
- Maintain clear distinctions between requested operations, authorization decisions, executed operations, and observed outcomes.
- Preserve provenance for modifications and allow human review.
- Support backup, restore, migration, key rotation, and failure recovery as designed capabilities, not emergency afterthoughts.
- Fail closed on authorization uncertainty; make recovery or reauthorization straightforward for an entitled operator.

## 12. Open ecosystem and presentation

- ALICE's own interface is replaceable and themeable. LCARS is welcome; no business logic belongs in theme code.
- Applications can be assigned to environments rather than baked into ALICE.
- Public APIs and SDKs should allow independent, including paid, extensions without requiring vendor infrastructure.
- The system's control plane and agent-operated surfaces should be visibly and technically separate.

## 13. Decision discipline

When choosing between implementations, prefer the one that:

1. Preserves user ownership and deployability.
2. Maintains accurate trust boundaries and least-privileged defaults.
3. Provides explicit administrator choice without silently escalating privilege.
4. Supports interoperability through documented contracts.
5. Reuses maintained open-source technology.
6. Can be tested, observed, recovered, and independently operated.

Document irreversible or consequential choices as Architecture Decision Records (ADRs). The architecture may evolve; these principles should change only deliberately.

---

**ALICE's central rule:** *Secure by default. Open by design. Powerful by choice.*

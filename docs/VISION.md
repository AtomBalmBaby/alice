# ALICE: Project Vision

**ALICE's Limited Interactive Computing Environment**

*How deep does your rabbit hole go?*

**Status:** Initial Vision  
**License:** AGPL-3.0-only  
**Repository:** `AtomBalmBaby/alice`

---

## 1. Vision

ALICE is an open-source, self-hosted computing platform designed to enable humans and artificial intelligence agents to work together within shared, interactive, and securely isolated environments.

ALICE challenges the conventional model of AI assistants operating through a limited collection of predefined tools, terminal commands, or application-specific integrations.

Instead, ALICE provides agents with access to complete computing environments, where they can observe, interact with, and operate applications much as a human would.

**The goal is not to build another AI assistant or another IDE. The goal is to build the environment in which humans and AI assistants can work together.**

## 2. The Problem

Today's AI agents are largely constrained by the interfaces through which they operate.

Coding assistants work inside editors. Browser agents operate browsers. Terminal agents execute commands. Desktop automation solutions often require broad access to the host operating system.

These approaches create several limitations:

- Agents are often tied to particular applications or development tools.
- Different tasks require different, frequently incompatible integrations.
- Human and agent interaction occurs through separate interfaces.
- Permissions are frequently controlled through instructions rather than enforceable security boundaries.
- Agents may have excessive access to the host system.
- Context and execution environments are difficult to preserve between sessions.

ALICE aims to address these limitations through a unified, project-oriented computing environment.

## 3. The ALICE Concept

In ALICE, a **project** is more than a collection of source files.

A project represents a defined computing environment containing the resources needed to accomplish a particular objective.

These resources may include:

- Files, directories, and repositories.
- Web applications and browser sessions.
- Terminals and command-line tools.
- SSH connections and remote systems.
- Graphical applications and desktop sessions.
- Containers and virtual machines.
- MCP servers and external tool interfaces.
- Credentials, capabilities, and access policies.
- Human and AI collaboration sessions.

A user can interact directly with these resources through the ALICE workbench.

An authorized AI agent can interact with those same resources through controlled interfaces.

Both operate within the boundaries established by the project.

**The project defines the world. ALICE enforces its boundaries. The agent operates within it.**

## 4. Human and AI Collaboration

ALICE treats humans and AI agents as participants in a shared computing environment, rather than forcing them into separate workflows.

An agent should be able to:

- Observe the state of an authorized application.
- Interact through virtual keyboard and mouse input.
- Use application-native interfaces when available.
- Execute explicitly permitted operations.
- Observe the results of its actions.
- Request human assistance or approval when necessary.
- Resume work from a preserved environment.

A human should be able to:

- Observe agent activity.
- Interact with the same applications.
- Take control of a session.
- Interrupt or revoke agent control.
- Approve or reject sensitive operations.
- Inspect activity history and execution results.

Shared control must be deliberate and coordinated. Human input takes precedence, and an agent must not silently regain control after a human intervenes.

ALICE should make collaboration natural without sacrificing human authority.

## 5. Security by Architecture

Security is a foundational property of ALICE, not an optional feature.

An agent must never gain unrestricted access to the host environment merely because it can operate a graphical interface, terminal, or tool.

ALICE must enforce isolation and permissions independently of agent instructions.

Its security model is based on several principles:

### Least Privilege

Agents receive only the capabilities explicitly granted to their project and session.

### Isolation

Execution environments are separated from the host and from other projects using appropriate operating-system and virtualization mechanisms.

### Controlled Interfaces

Agent operations pass through enforceable authorization boundaries.

### Credential Protection

Credentials remain under the control of trusted system components and are exposed to environments only when necessary and authorized.

### Human Authority

Users retain the ability to observe, interrupt, revoke, and terminate agent activity.

### Accountability

Agent actions and security-relevant events can be recorded and inspected.

**An AI prompt is not a security boundary.**

## 6. Application Independence

ALICE is not an IDE, browser, terminal emulator, or virtual machine manager.

It is a platform capable of incorporating these applications and environments.

A project may include an IDE such as VSCodium, a browser, a remote desktop, a specialized engineering application, or any combination of supported resources.

The agent interacts with the assigned environment rather than controlling the ALICE host interface itself.

This separation allows ALICE to accommodate a broad range of workflows without becoming dependent on any particular application.

## 7. Agent Independence

ALICE must not depend on any specific language model, inference engine, AI provider, or agent framework.

Agents should communicate through documented, extensible interfaces.

MCP is a primary integration technology, but ALICE's architecture must remain flexible enough to accommodate future protocols and interaction models.

Local, self-hosted inference is a first-class use case.

Remote inference may be supported without making external AI services mandatory.

ALICE should provide the environment and capabilities while allowing users to choose the intelligence operating within it.

## 8. Persistent Environments

A project should be capable of maintaining its working state across sessions.

Where supported and configured, this may include:

- Running applications.
- Open browser tabs.
- Active terminal sessions.
- Filesystem state.
- Desktop layout.
- Application configuration.
- Agent task state and execution history.

Persistence must be explicit and manageable. Some projects may require long-lived environments, while others may use temporary, disposable sandboxes.

ALICE should support both.

## 9. Open Source and Ecosystem Freedom

ALICE is open source by design.

The core platform is licensed under the GNU Affero General Public License v3.0.

ALICE welcomes independent extensions, integrations, applications, and services, including commercial products.

The project must not require proprietary infrastructure, proprietary AI services, or vendor-controlled accounts for its core functionality.

Third-party developers should be free to create their own products around ALICE using documented interfaces, subject to applicable licensing requirements.

**The platform remains open. The ecosystem is free to innovate.**

## 10. Architectural Direction

ALICE will initially target Linux, using a native C++ core and modular components.

The architecture should separate:

- The workbench user interface.
- Project and session management.
- Agent communication and tool integration.
- Authorization and security policy enforcement.
- Execution and application environments.
- Display, observation, and input transport.
- Resource and credential management.

The backend must not depend on a particular graphical frontend.

An initial VSCodium integration may provide a practical development and testing interface, while a standalone native workbench remains a long-term direction.

Existing open-source technologies should be reused where practical.

Specific technologies, interfaces, and implementation decisions will be documented separately through architecture specifications and decision records.

## 11. What ALICE Is Not

ALICE is not intended to be:

- Another proprietary AI assistant.
- A replacement for every existing IDE or desktop application.
- A new language model or inference engine.
- An unrestricted remote-access system for autonomous agents.
- A system whose security depends on an AI following instructions.
- A platform dependent on cloud services or mandatory subscriptions.

ALICE should integrate existing tools rather than unnecessarily reinvent them.

## 12. Long-Term Goals

ALICE aims to establish a general-purpose foundation for secure human-agent computing.

Long-term capabilities may include:

- Multiple independent project environments.
- Shared graphical and terminal interaction.
- Local and remote execution environments.
- Multiple agents operating within a project.
- Fine-grained authorization and delegated capabilities.
- Secure access to remote infrastructure.
- Persistent and reproducible workspaces.
- Extensible third-party application and tool ecosystems.
- Consistent interaction across heterogeneous operating systems and applications.

These are directions for development, not commitments to a particular implementation or release schedule.

## 13. Guiding Principle

ALICE exists to give artificial intelligence the ability to interact with real computing environments while ensuring that the humans operating those environments retain meaningful control.

The system must be powerful enough to be useful, flexible enough to accommodate new technologies, and constrained enough to be trusted.

**The host controls the environment. The project defines the boundaries. The agent operates within them. The human retains authority.**

---

*ALICE's Limited Interactive Computing Environment*

**How deep does your rabbit hole go?**

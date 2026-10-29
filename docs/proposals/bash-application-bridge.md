# ALICE Bash Application Bridge: First Reference Adapter

**Status:** Proposed design, not an adopted ADR or frozen API  
**Date:** 2026-10-29  
**Scope:** Application Bridge (AAB), sessions, event observation, process control and MCP compatibility  
**Initial platform:** Linux / GNU Bash  
**Companions:** [Architecture](../ARCHITECTURE.md) · [Design Principles](../DESIGN-PRINCIPLES.md) · [Security](../SECURITY.md) · [Persistent Task Intent (ADR-0012)](../decisions/0012-persistent-task-intent.md)

> **Idea:** Make a real, running Bash instance a first-class ALICE application. Bash exposes observable state and authoritative execution events through an ALICE attachment, with MCP as one optional gateway. This is an example implementation of the general application API, **not** a new terminal-centric direction for ALICE.

## 1. Motivation

During a recent coding-agent experiment, commands as trivial as "echo test123" and "date +%s" executed and returned a shell prompt, while the agent tool harness reported that completion could not be observed. Its model began issuing new commands to inspect process status, temporary log files and earlier commands. Those checks encountered the same uncertain completion signals, producing long delays and eventually pressure to ask a human to run commands manually.

The behavior reproduced with different inference backends and models. The problem it illustrates is general:

1. A running application performs an action successfully.
2. An intermediary fails to observe or report that outcome reliably.
3. An AI is handed ambiguous execution evidence.
4. The AI wastes work reconstructing state, makes unsafe assumptions or stops for human intervention.

**ALICE should not require a model to infer application state from terminal text when the application or operating system can provide structured evidence.**

Bash is an unusually useful first reference adapter: the command boundary, process lifecycle and output can be measured, and failures are straightforward to reproduce. The resulting architecture should later serve GUI software, browsers, CAD tools, remote machines, IoT devices and other applications.

## 2. Intended capabilities

A human or AI can discover an authorized shell resource, attach to its existing session where permitted, submit or interact with an operation, observe output and state changes, and receive correlated completion/failure notifications.

| Capability family (provisional) | Examples | Meaning |
| --- | --- | --- |
| Discovery and attachment | application.discover, application.attach | Identify permitted running Bash resources |
| Session lifecycle | shell.session.create, .attach, .detach, .inspect | Sessions outlive individual viewers when configured |
| Shell control | shell.command.submit, .inspect, .cancel | Request actions with typed result states |
| Observation | shell.state.read, events.subscribe | Read current state and follow its changes |
| Output | shell.command.output.subscribe | Ordered byte/event streams, with cursor support |
| Shared interaction | surface.attach, surface.input.keyboard | Optional terminal presentation and interactive input |
| Recovery | shell.session.reconcile, shell.command.history | Recover from disconnects without blind retries |

The actual names, transports and message schemas remain open. These must be **Application Bridge capabilities** under the existing ALICE architecture, not an independent Bash-only command execution subsystem.

A Bash adapter reports the features it genuinely supports. It must not claim that an arbitrary uninstrumented Bash instance offers a fully authoritative command boundary.

## 3. Not necessarily a Bash fork

The long-term design might justify a small GNU Bash modification or loadable builtin. **First build an adapter and prove the lifecycle contract without immediately maintaining a Bash fork.**

Candidate attachment modes:

| Mode | Mechanism | Strong points | Caveats |
| --- | --- | --- | --- |
| A: Managed session | ALICE launches Bash through a PTY and a process supervisor | Direct process ownership, straightforward exit observation, no custom shell build | Knowing when the *shell process* exits does not prove each *interactive command* finished |
| B: Opt-in instrumented shell | Bash hooks and/or a loadable builtin send events to a private socket/FD | Existing user workflow and a possible path to voluntary attachment | DEBUG traps, PROMPT_COMMAND, PIPESTATUS, nested evaluation and job control complicate correctness |
| C: Native Bash attachment | Small source patch at evaluator boundaries | May give precise command-start/end observations inside the running interpreter | Bash version maintenance, evaluator semantics, reentrancy and performance |
| D: External attachment | Connect to an already running PTY or cooperating shell | Fits ALICE's attach-to-real-applications philosophy | If no instrumentation existed, history and completion guarantees may be unavailable |

**Suggested prototype:** A first, then B. Consider C where measured gaps demand it. D advertises reduced capabilities rather than fabricating events.

A supervised subprocess's exit status may be authoritative for that process; it is **not automatically authoritative for every statement inside a persistent shell**. Returning to a prompt, shell process termination, foreground job termination and background job completion are distinct events.

## 4. Proposed component relationships

~~~mermaid
flowchart TD
    H["Human / Qt / VSCodium clients"] <--> C["aliced: sessions, resources, events"]
    AI["Cooperating AI agents"] <--> C
    AI <--> M["MCP gateway"]
    M <--> C
    C <--> P["Policy broker / alice-credd"]
    C <--> A["Bash Application Bridge adapter"]
    A <--> S["Process and PTY supervisor"]
    A <--> I["Optional Bash instrumentation"]
    S <--> B["Real Bash instance"]
    I <--> B
~~~

- **ALICE core** owns project/environment/resource/session identities, correlation, reconnection and authorized orchestration.
- **Policy and credential services** authenticate callers and authorize each operation; agent-profile prose and MCP tool names grant no authority.
- **Bash adapter** translates ALICE capability requests to shell-specific state and control.
- **Process supervisor** manages the processes it owns, PTYs, wait/exit status, signals and cancellation.
- **Optional Bash-side instrumentation** reports interpreter-level boundaries and observable state changes.
- **MCP gateway** provides compatibility for existing AI tools; it is neither the source of execution truth nor the security authority.
- **Human and multiple AI clients** can observe one session. Mutating control requires arbitration/leases appropriate to the environment.

This remains standalone and self-hostable; remote/federated attachment is an extension of the same protocol-neutral contracts, not a requirement for the first demo.

## 5. Command lifecycle: no prompt guessing

An illustrative state machine for a requested operation:

~~~text
accepted -> queued -> starting -> running -> completed
                                     |--> failed
                                     |--> canceled
                                     |--> outcome_unknown
~~~

Interpretation:

- **completed:** a trusted source observed termination and recorded exit status. A nonzero exit code is still a known completion.
- **failed:** dispatch/setup or an internal adapter error prevented a valid action; this is not synonymous with a command returning exit code 1.
- **canceled:** the relevant action was confirmed stopped, to the extent supported; side effects already performed are reported.
- **outcome_unknown:** command submission and observed outcome cannot be reconciled with available evidence. Do not claim completion or safely retry by default.

Background execution requires **separate command and job/process records**. The interactive statement "sleep 30 &" may have returned while the job it created remains running. Pipelines must preserve the shell's actual status semantics, including pipefail/PIPESTATUS where measurable. Builtins such as "cd", functions, redirection, subshells, exec, signals and shell traps need dedicated tests.

For a managed process, waitpid/equivalent reports the exit of a child the supervisor owns. For an interactive interpreter, command-level observation may require Bash-side instrumentation. The adapter must clearly attribute its result to the observation mechanism and its guarantees.

### Output, streaming and asynchronous operation

- Use ordered output chunks with sequence numbers, timestamps and resumable subscriptions.
- A PTY typically mixes stdout, stderr and terminal-control bytes. Only label streams separately when they were genuinely captured separately.
- Bound output buffers, support backpressure and explicitly report truncation or history gaps.
- A long-running command returns an **accepted operation handle**, not a fake synchronous completion.
- A heartbeat proves only that a component is reachable, never that a command finished.
- A lost acknowledgement must trigger state reconciliation, not blindly repeat a destructive command.
- Deduplication through idempotency keys is conditional on actual durable dispatch bookkeeping.

## 6. Example event

The format below is **illustrative**, not a finalized wire protocol.

~~~json
{
  "schema": "alice.application.event/v0",
  "event": "shell.command.completed",
  "event_id": "evt-0188",
  "node_id": "node-local",
  "project_id": "project-example",
  "environment_id": "env-07",
  "resource_id": "bash-01",
  "session_id": "session-42",
  "command_id": "cmd-0042",
  "request_id": "req-019",
  "trace_id": "trace-abc",
  "task_id": "task-001",
  "sequence": 188,
  "observed_at": "2026-10-29T19:20:48Z",
  "observation_source": "managed-process-supervisor",
  "result": {
    "exit_code": 0,
    "signal": null,
    "duration_ms": 37
  }
}
~~~

Authenticated identity and the authorization decision must derive from trusted ALICE services, not untrusted caller-supplied IDs.

Events should carry correlation IDs and a source/provenance description. A subscriber that reconnects either resumes from a cursor or receives an explicit history-gap condition. The system should measure time between dispatch, actual execution, result observation, event publication and the agent's next turn.

**Crucial invariant:** the agent must be able to distinguish "ALICE confirmed completion" from "terminal output suggests completion" and "outcome unknown".

## 7. MCP is an adapter, not ALICE's core

An MCP-facing adapter might expose the following optional tool vocabulary:

~~~text
alice_application_discover
alice_shell_attach
alice_shell_submit
alice_shell_status
alice_shell_output
alice_shell_subscribe
alice_shell_cancel
~~~

The MCP gateway maps client-visible tools onto ALICE capabilities after policy evaluation. When client-side streaming is unsupported, tools may query an authoritative command handle and event cursor instead of spawning repeated ps/grep checks against guessed PID/log files.

Clients with an appropriate standing grant may operate without repeated human approval prompts, including an explicitly authorized "never ask for routine actions" policy. **No approval setting bypasses resource scope, server permissions, credential isolation or trusted grant enforcement.**

## 8. Security and shared control

The adapter must inherit ALICE's existing security principles:

1. **Least privilege initially.** Launch in an appropriately scoped environment and OS identity, not root by default. Authorized administrators may deliberately grant host-integrated or privileged operation with truthful disclosure.
2. **Distinct capabilities.** Reading output, attaching to a session, accessing shell history, submitting commands, signaling jobs, keyboard input and inspecting environment variables are separate privileges.
3. **Authorized attachment.** No silent hijacking of a human terminal or its history, and no implicit exposure of environment secrets.
4. **Credential isolation.** Use alice-credd/policy brokering as applicable; avoid copying secrets, keys, passwords, cookies or raw environment values into agent contexts and routine logs.
5. **Mutual observability with arbitration.** Multiple observers are fine; simultaneous shell control requires clear ownership or a controlled input lease. Human takeover has precedence under project policy.
6. **Cancellation truthfulness.** Stop issuing new requests after revocation; terminate active processes only within the configured policy. Completed side effects cannot be rolled back by assertion.
7. **Raw shell is code execution.** A raw Bash command is not inherently sandboxed by being delivered over JSON/MCP. Enforce OS isolation and the granted identity.
8. **Remote nodes remain independently authoritative.** Encrypted transport is not authorization, and federation does not create new grants.

The adapter must never allow agent-written project files, prompts, or MCP descriptions to enlarge effective access.

## 9. Connection to cooperative AI and durable intent

ALICE is **not a linear pipeline of models**. Multiple AI stages may continuously exchange observations, critiques, revised plans and requests for clarification among themselves, reacting to state changes in running applications.

The Bash adapter's job is to provide common **facts and uncertainty**, not to decide strategy:

- A control agent starts a build under command ID 42.
- An observing agent sees diagnostic events and proposes a revised playbook.
- A reasoning agent checks proposed changes against persistent, versioned intent.
- Authorized work continues, potentially on independent tasks, while feedback circulates.

All actors can reference the same task, command, audit record, output cursor and application state. This fits [ADR-0012](../decisions/0012-persistent-task-intent.md): intent and acceptance evidence survive model swaps, disconnections and context compaction.

## 10. Implementation phases and verification

### Phase 0: Reproduce and establish measurements

Capture the real-world failure (echo succeeds, harness reports unknown), then record actual execution time versus tool completion-report latency. Establish baselines for exit code 0/1, long-running commands, interactive shells and disconnects.

### Phase 1: Managed Bash, native AAB contract

Launch a disposable Bash resource with a PTY/process supervisor. Provide discovery, attachment, typed command handle/results and output events. Prove reliable observed exit status **for processes actually supervised**. Demonstrate human and AI observing the same resource.

### Phase 2: Persistent interactive command boundaries

Implement and compare opt-in hooks/builtins against a minimal patch if needed. Validate Bash syntax, builtins, functions, pipelines, job control, background processes, signals, nested execution, command substitution, redirections, shell options and exit conditions. Mark unsupported observation semantics explicitly.

### Phase 3: MCP and cooperative agents

Expose the same resource through the MCP gateway. Allow several independent agents to observe one task and send permitted actions through distinct grants, with no routine nagging where standing approval is configured. Show event-driven feedback and stable evidence references.

### Phase 4: Failure recovery and remote attachment

Kill the adapter mid-command, disconnect a client, restart aliced and simulate duplicated requests. Verify state reconciliation, correct unknown-outcome classification, no unintended re-execution and proper revocation. Test a remote peer with authenticated, scoped access.

### Minimum acceptance demonstration

1. An authorized human and AI attach to a Bash session in ALICE.
2. AI requests "echo test123"; output is delivered and the authoritative completed/exit-code-0 event arrives promptly, without shell-prompt scraping.
3. "false" reports a *known* completion with exit code 1.
4. A background job remains independently running after the shell statement returns.
5. Human UI, agent and ALICE event journal agree on the session, command ID and outcome.
6. Disconnect/reconnect uses cursors and does not automatically rerun the command.
7. The same capability is reachable through ALICE's native API and the MCP gateway under equivalent effective grants.
8. A denied operation remains denied regardless of which AI or frontend requests it.

## 11. Open design decisions

- Which Bash integration guarantees require a patch rather than a builtin/hook?
- What is the first AAB wire protocol, and how are event cursors represented?
- How are surviving PTYs/shells safely rediscovered after daemon restart?
- Which command/output/history data are retained, redacted and exportable?
- What guarantees apply to a pre-existing shell with no ALICE instrumentation?
- After a working prototype, should this become a formal ADR and how does it interact with the first AAB adapter candidates in Architecture section 18?

## 12. Non-goals

- Replace Bash, SSH, terminal emulators, IDEs, MCP or agent frameworks.
- Guarantee authoritative completion for every arbitrary uninstrumented shell.
- Claim all actions are reversible or that killing a process undoes side effects.
- Make command execution the defining feature of ALICE.
- Depend on a single LLM, hosted inference vendor, proprietary service or database.

---

**Design rule:** *The application reports what happened. ALICE records who acted, what authority they had, and what remains uncertain. AI agents reason from evidence instead of guessing from a prompt.*

**Broader payoff:** If the Bash bridge can offer correct state, permissions, lifecycle and cross-agent observation, it becomes a practical reference implementation of the same Application Bridge API that will connect ALICE to real graphical and non-graphical applications elsewhere.

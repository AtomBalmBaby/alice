# ALICE Security Architecture

**Status:** Initial security specification / threat-model baseline  
**Companion:** [`DESIGN-PRINCIPLES.md`](DESIGN-PRINCIPLES.md)  
**Scope:** ALICE control plane, credential services, projects, environments, agent sessions, and federation

> **Security begins at the root.** ALICE integrates conventional operating-system and network security rather than attempting to compensate for dangerously privileged accounts, keys, or tokens. Give a task only the authority it needs.

## 1. Security objectives

ALICE must:

1. Protect host systems, other projects, and the ALICE control plane from untrusted environments under normal isolation policies.
2. Authenticate every principal requesting privileged control-plane operations.
3. Authorize operations against explicit, enforceable policy rather than agent prompts.
4. Keep credentials out of agent contexts wherever possible.
5. Support intentionally privileged and host-integrated configurations without representing them as sandboxed.
6. Provide auditable grants, revocation, recovery, and secure lifecycle management.
7. Support both local-only and distributed, self-hosted deployments.

Security is a set of enforceable guarantees under **specified assumptions**, not a blanket claim that any configuration is safe.

## 2. Principals, authority, and trust boundaries

Treat these as separate identities:

- **Operator:** an authenticated human who administers ALICE or uses a workbench.
- **Agent:** a software actor with task-scoped authorization, distinct from the operator.
- **Project:** policy and resource configuration; not itself a human identity.
- **Environment:** a runtime security context, potentially containing arbitrary untrusted code.
- **Service/daemon:** a specifically authenticated ALICE component.
- **Remote peer:** an independently administered service with its own identity and policies.

Primary trust boundaries:

- **Host/workbench control plane vs. project execution environment.** The agent must not control the trusted host UI or daemon simply because it controls an application viewport.
- **Authorization and credential services vs. agent-accessible tooling.** The agent must not receive credential-vault administrative access.
- **One project/environment vs. another.** Filesystems, IPC, devices, networking, and identity sockets must be scoped.
- **Local ALICE installation vs. remote/federated ALICE installation.** Trust is explicit, authenticated, and independently authorized.

A frontend is not a security boundary. An iframe, a Wayland window, or a restrictive prompt does not replace kernel, process, account, and network enforcement.

## 3. Default permissions and privileged operation

### Default posture

A new project receives no host root privilege, no broad Git write access, no unrestricted SSH, no arbitrary secrets, and no access to other projects. Resources are explicitly granted. Separate readable, writable, executable, and administrative capabilities.

### Deliberate elevation

An authenticated, entitled administrator can grant broader privileges, including:

- Network reachability or host filesystem mounts.
- Interactive shells, privileged containers, and full operating-system guests.
- Host-level administration and recursively nested ALICE instances.
- Unrestricted application control where desired.

Such grants must be explicit, logged, and accurately labeled. **If an agent gets effective host root access, ALICE cannot honestly guarantee its own confinement on that host.** Treat this as administrator-approved trust, not a successful sandbox.

### Delegation invariant

An agent or child environment may delegate only authority it holds *and is permitted to delegate*. A separately authenticated administrator may independently authorize extra capabilities for a child; that is a new grant, not privilege spontaneously obtained through recursion. Editing an agent-writable manifest must never alter effective authority without a trusted policy decision.

Revocation must stop future brokered operations and terminate affected sessions where possible. It cannot magically undo actions already performed or revoke tokens accepted by an external issuer without cooperation from that issuer.

## 4. `alice-credd`: credential and authority broker

The dedicated credential daemon (`alice-credd`, working executable name) is a separately deployable, trusted component. Its responsibilities include:

- Authenticate callers and establish service, human, agent, and workload identities.
- Resolve opaque credential references to provider-owned secret records.
- Evaluate policy for a specific operation, target, project, session, and requested lifetime.
- Request a trusted human approval when policy requires it.
- Perform brokered authentication, signing, or API calls when supported.
- Issue narrowly scoped, expiring capabilities or session handles where appropriate.
- Audit security-relevant decisions without logging secret values.
- Coordinate credential rotation, revocation, and provider health.

`alice-credd` must **not** be available as an unrestricted generic `get_secret` endpoint to agents. An administrator-facing export or recovery mechanism, if supported, must be a distinct high-trust interface.

### Provider interfaces (conceptual)

| Interface | Responsibility |
| --- | --- |
| `IMetadataStore` | Policy records, resource bindings, identities, audit indexes |
| `ISecretStore` | Encrypted secret records and retrieval for trusted components |
| `IKeyProvider` | Key generation, protection, wrapping, signing, and non-exportable operations |
| `IIdentityProvider` | Human, workload, and service authentication |
| `IPolicyEngine` | Authorization, delegation, approvals, revocation checks |
| `ICertificateProvider` | Issuance, renewal, trust chains, rotation, certificate status |
| `IRemoteCredentialProvider` | Authenticated cross-daemon operations and federation |

Providers can be in-process only if they are trusted at the daemon's privilege level. Less-trusted extensions should run out-of-process with constrained interfaces. Extensibility must not become arbitrary code execution in the vault process.

## 5. Credential storage, encryption, and keys

- Project manifests contain **references**, never plaintext credentials.
- ALICE-managed SQL storage must encrypt and authenticate secret values **before** persistence; metadata and policies may reside in SQL separately. Plan for backups and schema migration.
- The proposed initial managed database adapter is PostgreSQL. Support other SQL engines through deliberate backend implementations, not assumptions of universal SQL compatibility. SQLite is not the preferred ALICE-managed backend, but is not prohibited for downstream or adapter use.
- Never store encryption master keys in the same trust domain as encrypted records without an explicit rationale. Support external key management and rotation.
- TPM 2.0, PKCS#11/HSMs, and other providers may protect or operate cryptographic keys; they are **not** replacements for arbitrary SQL data storage.
- Use established, reviewed cryptographic libraries and algorithms, with authenticated encryption and clear key-version metadata. Never invent ALICE-specific cryptography.
- The Linux Secret Service API/KWallet may be an interactive desktop backend. `systemd` encrypted credentials may provision service bootstrap secrets. OpenSSH agents may protect SSH key use. No single backend is mandatory.
- Protect credentials in memory and on disk where feasible; minimize lifetime, avoid unnecessary copies, prevent deliberate inclusion in logs, and review crash-dump, swap, backup, and process-inspection exposure.
- Define backup/recovery independently of any optional TPM sealing, so hardware replacement cannot accidentally make every backup unusable.

## 6. Operation-based access, not only secret access

Prefer a trusted worker performing an authorized operation over disclosing an API token, SSH key, password, or client certificate to an agent.

**SSH:** Prefer a broker-managed connection or restricted remote account. Do not mount the user's general SSH-agent socket into an untrusted environment. A raw interactive root shell on a remote host grants broad authority regardless of local command-filter strings.

**Git:** Use repository- and operation-scoped identities/tokens where possible. Separate read from write; protect release branches and destructive actions using server-side controls.

**HTTP APIs:** Use a scoped API token and a trusted request broker when feasible. Enforce destination, method, and operation permissions; beware redirect and server-side request forgery paths.

**GUI/browser:** An authenticated browser session is a capability even if its password is hidden. Cookies, sessions, account privileges, downloads, and outbound requests must be considered part of the threat model.

**Raw credential delivery:** Allow it only through an explicitly authorized compatibility path with a clear statement that the receiving process can potentially read or leak the credential.

## 7. TLS, certificates, and federation

Separate these concerns:

1. **Transport confidentiality and server authentication** through TLS.
2. **Machine/service identity** through trusted workload credentials or mutual TLS.
3. **Authorization** through ALICE's policy engine, independent of certificate validity.
4. **Encryption at rest** through separate storage/key-management providers.

For public-facing HTTPS, support existing Let's Encrypt/ACME certificates, Certbot-managed files, reverse proxies, and other standards-compliant providers. Do not distribute one wildcard private key to every daemon merely because it is convenient.

For daemon-to-daemon mutual TLS, prefer a private PKI or an issuer designed for workload/client certificates. **As of July 2026, Let's Encrypt no longer issues certificates carrying the TLS Client Authentication EKU.** Public Let's Encrypt certificates remain suitable for supported server TLS use, but should not be the assumed mutual-TLS client identity mechanism.

Local connections should use protected Unix sockets and verify operating-system peer credentials; remote connections require authenticated encrypted transport and explicit peer enrollment/trust configuration. Never treat possession of a valid certificate as blanket permission to retrieve secrets.

Federation is an architectural goal, not assumed mutual trust. Each daemon retains authority over its own records. Requests must convey authenticated origin, requested operation, target, scope, expiration, and provenance. Guard against confused-deputy attacks, credential forwarding loops, replay, and privilege laundering across peers. Federation must be optional.

Certificate and trust material must have lifecycle management: issuance/import, renewal, validation, rotation, revocation or expiry policy, incident replacement, and recovery. Automate renewal where appropriate.

## 8. Isolation and execution environments

Support multiple isolation levels appropriate to the actual risk:

- **Unprivileged processes/containers:** use dedicated users, namespaces, cgroups, MAC policy where available (SELinux/AppArmor), resource quotas, syscall/device restrictions, and outbound network policy.
- **Virtual machines:** use hypervisor isolation for stronger separation or complete guest operating systems, with explicit handling for device passthrough, shared folders, and nested virtualization.
- **Privileged/host-integrated mode:** permit only by deliberate administrative grant; accurately describe which protections are bypassed.

Never assume containers provide VM-equivalent isolation. A user with host control or unconstrained access to the host container engine can bypass many restrictions. Host-level controls must be assessed independently from ALICE's application policy.

Treat graphical observation and input as scoped capabilities. A worker may control its assigned Wayland/X11/remote-desktop surface, not the ALICE host workbench. Prevent leakage of host display sockets, D-Bus services, credential-agent sockets, and broad filesystem mounts into untrusted environments.

## 9. Threat model and abuse cases

Initial threats include:

- Malicious or compromised agents attempting privilege escalation.
- Prompt injection through webpages, source files, terminal output, or MCP responses.
- Credential theft or misuse of authenticated sessions.
- Lateral movement between projects or through federation.
- Compromised provider plugins or remote services.
- Arbitrary shell access bypassing UI-only restrictions.
- Confused-deputy requests where a trusted worker uses its broader authority for an untrusted caller.
- Session hijacking, replay, insecure approvals, and resource exhaustion.
- Operators deliberately granting broad privileges, with consequences they may underestimate.

Threat mitigations must be tested at the actual enforcement boundary. All content returned from tools and applications is untrusted data, even when displayed in a trusted-looking interface.

## 10. Human approval, audit, and response

- An approval binds to an authenticated user, a specific operation, target, scope, and expiry; it is not an open-ended waiver.
- An agent cannot approve its own escalation via GUI automation or crafted API messages.
- Record identities, decisions, requested and effective capabilities, target resources, outcomes, and timestamps; do not log passwords, bearer tokens, raw private keys, or unnecessary sensitive content.
- Design a revocation/emergency-stop path independent of the agent being controlled.
- Support credential rotation, service trust reset, incident response, and restoration from encrypted backups.
- Distinguish policy-denied operations from failed execution and from unsupported platform capabilities.

## 11. Minimum security acceptance criteria for the first milestone

Before claiming an environment is isolated, demonstrate:

1. An unprivileged agent cannot modify ALICE policy or invoke the privileged control plane.
2. No credential material is present in project manifests, agent prompts, environment variables, or accessible host sockets by default.
3. Unauthorized filesystem, network, and other-project access is denied by the actual OS/runtime boundary.
4. `alice-ctl` can display effective permissions and their granting authority.
5. Credential access fails without an authenticated and authorized broker operation.
6. A human can revoke a session, and new tool operations fail afterward.
7. Administrator-granted privileged access is visibly marked and tested as **not isolated**, where applicable.
8. Logs enable incident review without recording secrets.
9. Public remote endpoints use authenticated encrypted transport; local control endpoints restrict peer access.
10. Destruction and restart clean up environment-specific sockets, mounts, credentials, and processes.

## 12. Decisions still to specify

The detailed architecture/ADRs must select and document:

- Initial credential-daemon deployment and privilege model.
- Bootstrap authentication, first administrator enrollment, and recovery procedures.
- Exact policy language, decision semantics, and approval UX.
- PostgreSQL schema, encryption-at-rest approach, key lifecycle, and backups.
- Initial Linux sandbox backend and any privileged helper surface.
- IPC protocol and peer authentication.
- Whether built-in certificate issuance is in the MVP or delegated to installed providers.
- Federation onboarding, trust rotation, and version compatibility.
- Audit persistence, retention, integrity controls, and secure export.

None of these choices should be hidden inside agent instructions or a database convention.

---

**Security principle:** *Do not grant root where root is not required. Do not grant Git write where read will do. Do not give an agent a secret when a scoped operation will do.*

**Freedom principle:** *If an entitled administrator deliberately removes a boundary, ALICE shall allow the choice where technically possible and never misrepresent the resulting security.*

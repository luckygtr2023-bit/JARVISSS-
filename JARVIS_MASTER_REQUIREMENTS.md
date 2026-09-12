# JARVIS Master Requirements Baseline

* **Document Version:** 1.3.0
* **Status:** Binding Baseline (LOCKED)
* **Target Platform:** Windows (Primary), Android Companion (Secondary)
* **Document Role:** Single Source of Truth for Functional and Non-Functional System Requirements

---

## 1. Vision and Goals

### 1.1 Vision
JARVIS is an intelligent, voice-first, multimodal, autonomous desktop assistant for Windows. It acts as an extension of the user, executing complex local and remote tasks across system applications, web browsers, and external APIs while preserving explicit human control over sensitive side effects.

### 1.2 Core Goals
1. **Unified Control Interface:** Provide a single multimodal interface (voice, text, screen context) to interact with the entire operating system, web applications, and personal data.
2. **Autonomous Execution with Safety:** Plan and execute multi-step tool workflows safely using deterministic policy checks, strict permission levels, and dry-run simulation.
3. **Model & Infrastructure Independence:** Remain vendor-agnostic across reasoning, planning, vision, local, and fast-response execution models.
4. **Verifiable State Management:** Ensure every action taken by JARVIS is explicit, auditable, reversible where possible, and resilient against execution ambiguity.
5. **Non-Authoritative AI:** No AI output, memory record, cached state, or untrusted content shall be a source of authorization. Authorization derives exclusively from authenticated system policy and the current request context.
6. **Bounded Execution Authority:** Every side-effecting operation executes under a revocable ownership grant over its declared resource keys; loss of ownership precludes further side effects.

### 1.3 Non-Authoritative Principle
The following classes of information shall **never** be an input to an authorization decision, risk classification, or permission grant:
1. AI/LLM-generated text, labels, annotations, or plan metadata.
2. Content retrieved from external sources (web pages, external APIs, documents, OCR output, plugin output, external model responses).
3. Long-term memory records, session history, or cached approvals.
4. Any field not derived from an authenticated system policy artifact or the actual operation parameters under evaluation.

---

## 2. Requirements Structure and Taxonomy

All requirements follow the standard format:
* **Requirement ID:** `REQ-[CATEGORY]-[NUMBER]`
* **Priority:** MUST (Required) | SHOULD (High Value) | MAY (Optional)
* **Requirement Statement:** Precise, unambiguous functional/non-functional statement.
* **Rationale:** Objective underlying reason for requirement.
* **Dependencies:** Prerequisites or linked requirement IDs.
* **Security Implications:** Security boundaries, risks, or mitigations involved.
* **Verification Method:** Test / Inspection / Demonstration / Analysis.

Requirements are implementation-neutral: they state *what must be guaranteed*, not *how*. Mechanism selection is deferred to the Architecture Phase.

---

## 3. Detailed System Requirements

### 3.1 Core Assistant Behavior

* **REQ-CORE-001**
  * **Priority:** MUST
  * **Requirement Statement:** All user requests, system events, and UI interactions shall be parsed, evaluated, and executed exclusively through a single control core. No UI component, plugin, AI output, or external input shall invoke a tool or initiate a side-effecting operation except via the control core.
  * **Rationale:** Prevents UI or rogue threads from bypassing safety, policy, and audit systems.
  * **Dependencies:** None.
  * **Security Implications:** Eliminates direct tool execution path from UI layers.
  * **Verification Method:** Inspection and static analysis demonstrating that tool-invocation entry points are unreachable from UI and plugin code paths.

* **REQ-CORE-002**
  * **Priority:** MUST
  * **Requirement Statement:** The core shall maintain a deterministic lifecycle state machine for every user request, with a defined, versioned transition table. The state set shall include at minimum: `Initiated`, `Planning`, `Awaiting_Confirmation`, `Executing`, `Verifying`, `Completed`, `Failed`, `Cancelled`, `Timed_Out`, `Ambiguous_Uncertain`, `Halted_Requires_Inspection`. Illegal transitions shall be rejected and logged.
  * **Rationale:** Ensures execution states are explicit, predictable, and recoverable across crashes or restarts.
  * **Dependencies:** `REQ-CORE-001`
  * **Security Implications:** Prevents orphan tool execution during state transition delays.
  * **Verification Method:** Test against the transition table; fault-injection of illegal transitions.

* **REQ-CORE-002a**
  * **Priority:** MUST
  * **Requirement Statement:** `Completed`, `Failed`, and `Cancelled` shall be terminal. No transition out of a terminal state shall be permitted; continuation requires issuance of a new request identifier.
  * **Rationale:** Prevents state resurrection and audit ambiguity.
  * **Dependencies:** `REQ-CORE-002`
  * **Security Implications:** Blocks replay of completed side-effecting operations via state manipulation.
  * **Verification Method:** Test.

* **REQ-CORE-003**
  * **Priority:** MUST
  * **Requirement Statement:** The core shall support concurrent monitoring of multiple active requests while serializing all actions classified `LOW_RISK_SIDE_EFFECT` or higher against a declared resource key. Every tool manifest shall declare the resource keys its operation reads and writes, and shall declare its tolerance for reading resource keys during a concurrent write. Concurrent reads shall be permitted. A write shall be mutually exclusive with any other read or write to the same resource key for the duration of the write, except that a read may proceed during a write if the reader has declared tolerance for intermediate states. Two actions with overlapping write-resource keys shall not execute concurrently.
  * **Rationale:** Avoids race conditions and conflicting state updates on the local system.
  * **Dependencies:** `REQ-CORE-002`, `REQ-EXT-001`
  * **Security Implications:** Prevents race-condition privilege escalation and file corruption.
  * **Verification Method:** Concurrency test with overlapping resource keys; test of declared read-tolerance.

* **REQ-CORE-003a**
  * **Priority:** MUST
  * **Requirement Statement:** Replanning deltas shall not be evaluated against or merged into an in-flight serialized write. A delta shall queue until the active write reaches a terminal state. If the active write ends in `Ambiguous_Uncertain` or `Halted_Requires_Inspection`, the queued delta shall be held and presented to the user rather than auto-applied.
  * **Rationale:** Prevents interleaving of un-authorized plan fragments with in-flight side effects.
  * **Dependencies:** `REQ-CORE-003`, `REQ-AI-004`, `REQ-SIM-002`
  * **Security Implications:** Blocks a class of TOCTOU permission escalation.
  * **Verification Method:** Fault-injection test.

* **REQ-CORE-004**
  * **Priority:** MUST
  * **Requirement Statement:** The UI process shall be treated as less trusted than the control core. Confirmation signals emitted by the UI shall be integrity-protected and bound to a specific request identifier and a cryptographic digest of the authorized operation (tool identity, parameters, resource keys). The core shall reject confirmations that do not bind to an outstanding request. This requirement is the primary definition of the confirmation-binding contract; `REQ-SEC-007` specifies what the confirmation prompt must display; `REQ-SEC-008` specifies secret-parameter handling.
  * **Rationale:** The confirmation UI is the last line of defense against unauthorized execution.
  * **Dependencies:** `REQ-SEC-001`, `REQ-SEC-007`, `REQ-SEC-008`
  * **Security Implications:** Prevents a compromised UI from impersonating a valid confirmation or transforming one approved action into another.
  * **Verification Method:** Test with tampered confirmation payloads.

* **REQ-CORE-005**
  * **Priority:** MUST
  * **Requirement Statement:** Every execution of a resource-modifying operation shall hold a revocable ownership grant over its declared resource keys for the duration of the operation. A core-side decision to cancel, time out, or terminate an operation shall revoke that ownership grant. A worker whose ownership grant has been revoked shall be prevented from producing further side effects on its declared resource keys, regardless of whether the worker has acknowledged cancellation. If ownership cannot be revoked with certainty (e.g., worker unresponsive, network partition, core restart while worker survives, delayed callback), the operation shall be recorded as `Ambiguous_Uncertain`, and the affected resource keys shall not be re-issued to any other operation until the ambiguity is resolved. Resource keys shall not be released on timeout alone; release requires either confirmed worker termination or explicit ambiguity recording. A transition to `Timed_Out` (permitted by `REQ-REL-001` when the tool manifest declares the operation idempotent and safely abortable) shall be treated as confirmed worker termination and shall release ownership of the operation's declared resource keys.
  * **Rationale:** Without an ownership/fencing contract, a "cancelled" or "timed-out" worker can continue producing side effects that the core believes were abandoned.
  * **Dependencies:** `REQ-CORE-003`, `REQ-REL-001`, `REQ-SIM-002`
  * **Security Implications:** Prevents stale execution from producing unauthorized state mutations; prevents write/write races between a fresh operation and a stale worker.
  * **Verification Method:** Fault-injection test: kill core mid-write and verify worker cannot produce further side effects; verify resource key is not re-issued until ambiguity resolved; test `Timed_Out` releases ownership.

---

### 3.2 Natural-Language Interaction & AI Reasoning

* **REQ-AI-001**
  * **Priority:** MUST
  * **Requirement Statement:** The system shall support substituting any reasoning, planning, vision, local, or fast-response model provider without modification to core policy, safety, or audit logic.
  * **Rationale:** Guarantees vendor independence, resilience to API deprecation, and cost optimization.
  * **Dependencies:** None.
  * **Security Implications:** Provider-substitution paths must preserve all security properties in this document.
  * **Verification Method:** Demonstration of provider substitution with no change to core policy components.

* **REQ-AI-002**
  * **Priority:** MUST
  * **Requirement Statement:** The system shall route model calls at runtime according to declared task attributes (complexity, latency, privilege, privacy, cost). Privacy constraints shall be a hard filter: if no permitted model satisfies the active privacy constraint, the task shall fail closed rather than route to a disallowed model. Every routing decision shall be logged with its constraint set and selected model identity.
  * **Rationale:** High-risk or fast-path tasks require localized or specialized reasoning tiers.
  * **Dependencies:** `REQ-AI-001`
  * **Security Implications:** Prevents privacy-violating fallback under cost pressure.
  * **Verification Method:** Test with constrained routing scenarios.

* **REQ-AI-003**
  * **Priority:** MUST
  * **Requirement Statement:** The AI system shall decompose complex user intents into an explicit execution plan structured as a directed acyclic graph before execution. Each node shall reference a tool manifest identifier and concrete parameter values. Plan generation shall not be the source of risk classification (see `REQ-AI-008`).
  * **Rationale:** Allows deterministic security policy checking prior to initiating side effects.
  * **Dependencies:** `REQ-CORE-001`, `REQ-AI-001`, `REQ-AI-008`
  * **Security Implications:** Prevents hidden or dynamically inserted steps without plan re-validation.
  * **Verification Method:** Test.

* **REQ-AI-004**
  * **Priority:** MUST
  * **Requirement Statement:** Replanning triggered mid-execution shall re-submit the entire delta plan through the security policy evaluator. A delta plan shall additionally be evaluated against the original user intent, the originally authorized goal, the originally authorized resource scope, and the cumulative side-effect budget of the request. A delta that materially changes the authorized operation shall require fresh explicit user authorization, regardless of the individual risk tier of its nodes. A **material change** is any change to: the set of resource keys touched; the manifest-derived risk tier of any node; the number of nodes in the plan; the tool identity of any node; a parameter value that affects the operation's semantics; or the cumulative side-effect budget.
  * **Rationale:** Prevents prompt-injection-driven or drift-driven bypass of security controls.
  * **Dependencies:** `REQ-AI-003`, `REQ-SEC-001`, `REQ-CORE-003a`
  * **Security Implications:** Blocks privilege escalation via iterative context manipulation and low-risk-step chaining.
  * **Verification Method:** Test with adversarial delta plans.

* **REQ-AI-004a**
  * **Priority:** MUST
  * **Requirement Statement:** The system shall maintain a cumulative side-effect budget per request, declared at plan authorization time and bounded by the active policy. Exceeding the budget shall halt execution and require fresh user authorization. The budget shall be reconstructed from the audit log on recovery.
  * **Rationale:** Prevents chaining of individually low-risk steps into an unauthorized high-impact result.
  * **Dependencies:** `REQ-AI-004`, `REQ-DEP-002`
  * **Security Implications:** Closes the "boiling frog" escalation vector.
  * **Verification Method:** Test.

* **REQ-AI-005**
  * **Priority:** MUST
  * **Requirement Statement:** All communications with external model providers shall be integrity- and confidentiality-protected, and the system shall verify the endpoint identity before transmitting any prompt or context. The system shall refuse transmission if endpoint identity cannot be verified.
  * **Rationale:** Protects prompts and context from interception or redirection.
  * **Dependencies:** `REQ-AI-001`
  * **Security Implications:** Prevents credential, prompt, and context exposure to impersonated endpoints.
  * **Verification Method:** Test with invalid endpoint identity.

* **REQ-AI-006**
  * **Priority:** MUST
  * **Requirement Statement:** If the selected provider is unavailable, times out, rate-limits, exhausts quota, returns malformed or schema-invalid output, or returns contradictory output, the affected request shall fail closed. The system shall not auto-substitute a provider whose privacy, authorization, or risk-classification properties differ from the original selection. Provider substitution shall require the substitute to satisfy the same constraints as the original.
  * **Rationale:** Prevents silent weakening of security posture on provider failure.
  * **Dependencies:** `REQ-AI-001`, `REQ-AI-002`, `REQ-AI-009`
  * **Security Implications:** Closes fallback-based downgrade paths.
  * **Verification Method:** Fault-injection test per failure mode.

* **REQ-AI-007**
  * **Priority:** MUST
  * **Requirement Statement:** The system shall maintain a user-inspectable record of every external model endpoint contacted, the categories of data transmitted, and the timestamp. The system shall not transmit data to a provider whose data-handling terms have not been acknowledged by the user.
  * **Rationale:** Enables informed user consent to data egress.
  * **Dependencies:** `REQ-AI-005`, `REQ-DEP-002`
  * **Security Implications:** Ensures third-party disclosure is auditable.
  * **Verification Method:** Inspection and Demonstration.

* **REQ-AI-008**
  * **Priority:** MUST
  * **Requirement Statement:** Risk classification and authorization decisions shall be derived exclusively from the tool manifest's declared side-effect class, the actual parameter values, and the authenticated policy artifact. Plan generation and risk classification shall be performed by components with no shared mutable state. The classifier shall not accept AI-generated risk labels as input. Any plan node whose AI-declared label conflicts with manifest-derived classification shall be rejected.
  * **Rationale:** Prevents an AI (or prompt-injected content) from lowering an operation's required authorization.
  * **Dependencies:** `REQ-AI-003`, `REQ-SEC-001`, `REQ-SEC-002`
  * **Security Implications:** Primary defense against AI-mediated privilege escalation.
  * **Verification Method:** Test with adversarially mislabeled plans.

* **REQ-AI-009**
  * **Priority:** MUST
  * **Requirement Statement:** All model outputs used in planning, classification, or tool invocation shall be validated against a versioned schema before use. Outputs failing validation shall be rejected and the request failed closed. Malformed or schema-invalid output shall never be interpreted as a plan.
  * **Rationale:** Prevents malformed output from becoming an executable plan.
  * **Dependencies:** `REQ-AI-006`
  * **Security Implications:** Blocks a class of injection via malformed responses.
  * **Verification Method:** Test with schema-invalid outputs.

* **REQ-AI-010**
  * **Priority:** SHOULD
  * **Requirement Statement:** The system shall record the model identity and, where the provider exposes it, the model version used for each plan. Where the provider does not expose version identity, the system shall record that fact and shall not represent plan reproducibility as guaranteed.
  * **Rationale:** Supports post-hoc audit and behavioral drift analysis.
  * **Dependencies:** `REQ-AI-001`, `REQ-DEP-002`
  * **Security Implications:** Improves incident investigation.
  * **Verification Method:** Inspection.

* **REQ-AI-011**
  * **Priority:** MUST
  * **Requirement Statement:** The system shall define and document a degraded operating mode for the absence of all external model providers. Degraded mode shall not relax any security, privacy, or confirmation requirement, and shall specify which request classes remain available.
  * **Rationale:** Matches the offline baseline claimed in the Assumptions document.
  * **Dependencies:** `REQ-AI-006`
  * **Security Implications:** Prevents "offline" from becoming a security-relaxed state.
  * **Verification Method:** Demonstration.

* **REQ-AI-012**
  * **Priority:** MUST
  * **Requirement Statement:** Provider quota exhaustion shall fail the request closed with a user-visible reason. The system shall not auto-substitute a provider with different privacy or privilege characteristics.
  * **Rationale:** Prevents silent weakening on quota exhaustion.
  * **Dependencies:** `REQ-AI-006`
  * **Security Implications:** Closes quota-pressure downgrade path.
  * **Verification Method:** Test.

---

### 3.3 Voice & Audio Interaction

* **REQ-VOI-001**
  * **Priority:** MUST
  * **Requirement Statement:** The system shall continuously monitor local audio input for a configurable wake-word. Pre-wake audio shall not leave the host. Post-wake audio may be transmitted only to endpoints permitted by the active privacy policy and only for the duration of the active request.
  * **Rationale:** Preserves user privacy and reduces local resource consumption.
  * **Dependencies:** None.
  * **Security Implications:** Zero ambient audio exfiltration prior to explicit wake activation.
  * **Verification Method:** Test and network analysis.

* **REQ-VOI-002**
  * **Priority:** MUST
  * **Requirement Statement:** The system shall convert spoken input into text with sub-500ms partial-transcript latency on the recommended hardware baseline. On the minimum hardware baseline (per Assumptions §1.4), the system shall target sub-1500ms final-transcript latency and shall display partial transcripts while final decoding is in progress. The system shall publish a measured latency budget per hardware tier.
  * **Rationale:** Maintains fluid real-time conversational UX while remaining honest about minimum hardware.
  * **Dependencies:** `REQ-VOI-001`
  * **Security Implications:** Local speech models must operate within isolated sandboxes.
  * **Verification Method:** Test on both hardware tiers.

* **REQ-VOI-003**
  * **Priority:** MUST
  * **Requirement Statement:** The text-to-speech module shall provide streaming audio output capable of immediate interruption upon detection of user speech or a physical hotkey. Barge-in shall be routed through the core state machine. Barge-in during `Awaiting_Confirmation` or `Executing` for a `HIGH_RISK_SIDE_EFFECT` or `CRITICAL_SYSTEM` action shall first silence output and then require explicit confirmation of cancellation before any tool sub-process is terminated or any side effect is reversed. Barge-in shall not directly signal tool sub-processes.
  * **Rationale:** Grants the user immediate control while preserving safe cancellation semantics. Barge-in is a user-safety capability; it is MUST-priority because loss of voice control over active execution is a safety regression.
  * **Dependencies:** `REQ-VOI-001`, `REQ-CORE-002`
  * **Security Implications:** Prevents ambiguous cancellation of high-risk side effects.
  * **Verification Method:** Demonstration and Test.

* **REQ-VOI-004**
  * **Priority:** MUST
  * **Requirement Statement:** The system shall display a persistent, user-visible indicator whenever the microphone is actively capturing. The indicator shall reflect actual capture state, not requested state. If the OS or hardware mute is engaged, the system shall not represent itself as listening.
  * **Rationale:** User awareness of capture is a privacy prerequisite.
  * **Dependencies:** `REQ-VOI-001`
  * **Security Implications:** Prevents silent capture states.
  * **Verification Method:** Demonstration.

---

### 3.4 Vision & Screen Understanding

* **REQ-VIS-001**
  * **Priority:** MUST
  * **Requirement Statement:** The system shall capture multi-monitor display regions or specific window regions only upon either (a) an explicit user request, or (b) a declared plan node whose manifest scope includes screen capture. Captures shall be convertable into a structured spatial representation associating recognized elements with bounding regions and text; the schema shall be versioned and validated before use in planning.
  * **Rationale:** Enables context-aware automation of applications lacking native accessible API endpoints, while bounding capture authority.
  * **Dependencies:** None.
  * **Security Implications:** Visual captures might expose passwords, personal healthcare info, or credentials.
  * **Verification Method:** Test.

* **REQ-VIS-002**
  * **Priority:** MUST
  * **Requirement Statement:** Before any frame is transmitted to an external model, the system shall apply a local redaction pass targeting at minimum password fields, payment-card patterns, API-key patterns, and user-configured sensitive terms. The system shall not represent redaction as a guarantee of non-disclosure; residual sensitive content may remain. Frames shall be transmitted only when the plan node's manifest scope explicitly authorizes external vision processing.
  * **Rationale:** Reduces, but does not eliminate, the risk of visual credential exfiltration.
  * **Dependencies:** `REQ-VIS-001`, `REQ-SEC-005`
  * **Security Implications:** Critical privacy defense layer; documented as partial.
  * **Verification Method:** Test.

* **REQ-VIS-003**
  * **Priority:** MUST
  * **Requirement Statement:** Text extracted from screen captures shall be treated as untrusted content under `REQ-SEC-005`. It shall not be able to influence risk classification, policy evaluation, or tool manifest interpretation.
  * **Rationale:** Visual prompt injection is a known attack class.
  * **Dependencies:** `REQ-VIS-001`, `REQ-SEC-005`
  * **Security Implications:** Blocks visual prompt-injection escalation.
  * **Verification Method:** Test with adversarial screen content.

---

### 3.5 Memory & Context Management

* **REQ-MEM-001**
  * **Priority:** MUST
  * **Requirement Statement:** The system shall maintain volatile short-term session memory for conversation flow and task lifecycle state. Uncommitted short-term items shall be cleared upon application exit or user logout.
  * **Rationale:** Minimizes context bloat and prevents transient execution data from polluting persistent state.
  * **Dependencies:** `REQ-CORE-002`
  * **Security Implications:** Prevents cross-session context bleed.
  * **Verification Method:** Test.

* **REQ-MEM-002**
  * **Priority:** MUST
  * **Requirement Statement:** The long-term memory module shall store episodic facts, user preferences, and structural knowledge in an encrypted, searchable local store under explicit access controls. Encryption keys shall be protected by the host operating system's user-bound secret storage such that the key is not recoverable from disk alone and does not require manual passphrase re-entry on background service restart within an authenticated user session. The specific key-protection mechanism shall be selected in the architecture phase.
  * **Rationale:** Personalization requires persistence without exposing sensitive metadata to external access.
  * **Dependencies:** None.
  * **Security Implications:** Key must be bound to the authenticated user identity.
  * **Verification Method:** Inspection and Test.

* **REQ-MEM-003**
  * **Priority:** MUST
  * **Requirement Statement:** The system shall provide a user interface to inspect, modify, export, and delete individual items or the entirety of long-term memory. The interface shall display the partition type (episodic/preference vs. policy) and provenance of every item. Deletion shall be hard deletion without remaining shadow copies within the system's control.
  * **Rationale:** Ensures user ownership of personal memory data.
  * **Dependencies:** `REQ-MEM-002`, `REQ-MEM-004`, `REQ-MEM-007`
  * **Security Implications:** Ensures hard-deletion semantics.
  * **Verification Method:** Demonstration.

* **REQ-MEM-004**
  * **Priority:** MUST
  * **Requirement Statement:** State stored in long-term memory shall never grant persistent permission overrides or override runtime security access evaluations. The Security Policy Evaluator shall derive authorization exclusively from the authenticated policy artifact and the current request context. It shall not query long-term memory, session history, or cached approvals for authorization inputs. Any code path that reads memory during an authorization decision shall be treated as a defect.
  * **Rationale:** Avoids authorization bypass via cached or injected memory state.
  * **Dependencies:** `REQ-MEM-002`, `REQ-SEC-001`
  * **Security Implications:** Direct defense against permission bypass via memory.
  * **Verification Method:** Test and static analysis.

* **REQ-MEM-005**
  * **Priority:** MUST
  * **Requirement Statement:** The system shall define configurable default retention periods for audio buffers, screen captures, prompt logs, and short-term memory. Expired data shall be purged automatically without requiring user action. Raw audio and raw screen captures shall not be persisted beyond the duration required for the active request unless the user explicitly opts in.
  * **Rationale:** Bounded retention is a privacy prerequisite.
  * **Dependencies:** `REQ-MEM-001`, `REQ-VOI-001`, `REQ-VIS-001`
  * **Security Implications:** Reduces exposure window for captured sensitive content.
  * **Verification Method:** Test and Inspection.

* **REQ-MEM-006**
  * **Priority:** MUST
  * **Requirement Statement:** Long-term memory, credentials, and session state shall be scoped to the authenticated user identity. Background services shall not access another user's memory or credentials. On user logout, volatile short-term memory and decrypted key material shall be cleared.
  * **Rationale:** Prevents cross-user privacy bleed on shared hosts.
  * **Dependencies:** `REQ-MEM-001`, `REQ-MEM-002`
  * **Security Implications:** Enforces per-user isolation.
  * **Verification Method:** Test with fast-user-switching and logout.

* **REQ-MEM-007**
  * **Priority:** MUST
  * **Requirement Statement:** Long-term memory records shall carry provenance metadata. Records derived from untrusted sources shall be treated as untrusted when retrieved for planning and shall be subject to the same containment requirements as untrusted content under `REQ-SEC-005`. Provenance shall propagate transitively through all memory writes and reads.
  * **Rationale:** Prevents laundering of untrusted content through memory.
  * **Dependencies:** `REQ-MEM-002`, `REQ-SEC-005`
  * **Security Implications:** Closes the memory-laundering injection vector.
  * **Verification Method:** Test with memory records derived from adversarial untrusted sources.

---

### 3.6 Browser Capabilities

* **REQ-BRW-001**
  * **Priority:** MUST
  * **Requirement Statement:** The browser subsystem shall support controlled web searching, fetching, and structured extraction using designated web APIs or headless rendering instances. All retrieved content shall be treated as untrusted input under `REQ-SEC-005`. Script execution within retrieved content shall be disabled in the automation context.
  * **Rationale:** Information retrieval must operate independently of the user's manual browser session and must not import hostile script or instruction content.
  * **Dependencies:** `REQ-SEC-005`
  * **Security Implications:** Web content is a primary prompt-injection vector.
  * **Verification Method:** Test.

* **REQ-BRW-002**
  * **Priority:** MUST
  * **Requirement Statement:** Browser automation (form filling, navigation, button interactions) shall run within isolated, profile-segregated sessions distinct from the user's primary browsing session, with clear visual indicators when running live. The automated session shall not have access to the user's primary browser profile, saved credentials, session cookies, or OS credential store. Any authentication performed in the automation context shall use credentials explicitly scoped to automation.
  * **Rationale:** Distinguishes background automated browsing from the user's authenticated session and prevents credential inheritance.
  * **Dependencies:** `REQ-BRW-001`
  * **Security Implications:** Protects active user browser sessions and saved credentials.
  * **Verification Method:** Demonstration.

* **REQ-BRW-003**
  * **Priority:** MUST
  * **Requirement Statement:** Downloads initiated by browser automation, and file access performed by browser automation, shall be constrained to the designated workspace (`REQ-WIN-003`). Navigation to destinations whose resolution fails workspace or trust checks shall be blocked and logged.
  * **Rationale:** Prevents browser automation from becoming a file-write or exfiltration primitive.
  * **Dependencies:** `REQ-BRW-002`, `REQ-WIN-003`
  * **Security Implications:** Closes download-based sandbox escape.
  * **Verification Method:** Test.

* **REQ-BRW-004**
  * **Priority:** MUST
  * **Requirement Statement:** All content retrieved by the browser subsystem shall be tagged with provenance (`REQ-SEC-005`) and shall not be permitted to modify tool manifests, policy artifacts, risk classifications, or authorization state. Retrieved content shall not be interpreted as instructions to the system.
  * **Rationale:** Promotes the audit's non-binding note into a numbered, testable requirement.
  * **Dependencies:** `REQ-BRW-001`, `REQ-SEC-005`
  * **Security Implications:** Primary defense against web prompt injection.
  * **Verification Method:** Test with adversarial web pages.

* **REQ-BRW-005**
  * **Priority:** MUST
  * **Requirement Statement:** Browser-initiated outbound requests shall be classified by their observed or declared effect on external state, not by HTTP method verb. Requests that alter state on an external service (form submissions, state-changing operations, persistent connections that perform actions, downloads that trigger remote billing or acknowledgement) shall be classified and gated equivalently to local write actions of the same impact tier. Requests that are declared read-only by the endpoint's contract (including POST-based search APIs) shall be treated as retrieval and are not side effects. In the absence of a machine-readable read-only declaration, POST-based requests shall be conservatively classified as side effects.
  * **Rationale:** Prevents browser activity from creating external side effects outside the authorization model, while avoiding over-restriction of legitimate read-only POST-based APIs.
  * **Dependencies:** `REQ-BRW-001`, `REQ-SEC-001`, `REQ-SEC-005`
  * **Security Implications:** Closes browser-mediated external side-effect bypass.
  * **Verification Method:** Test with adversarial pages that trigger outbound side effects; test with declared read-only POST endpoints.

---

### 3.7 Windows OS Automation & Control

* **REQ-WIN-001**
  * **Priority:** MUST
  * **Requirement Statement:** The system shall prefer structured, semantically-addressed application automation surfaces over coordinate-based input synthesis. Coordinate synthesis shall be used only when no structured surface is available for the target application; such use shall be logged with the reason and with the target window identity.
  * **Rationale:** Structured interaction is more robust and non-intrusive than coordinate clicking.
  * **Dependencies:** None.
  * **Security Implications:** Reduces misdirected input and unintended targets.
  * **Verification Method:** Test and Inspection of logs.

* **REQ-WIN-002**
  * **Priority:** MUST
  * **Requirement Statement:** The system shall not interact with any window, control, or process whose integrity level or privilege context exceeds the system's own. This prohibition shall apply to synthesized input, accessibility-tree interaction, and any other form of automation. The system shall not launch binaries that are unsigned or whose signature cannot be verified. The only permitted exception is an explicitly configured elevated service mode. Elevated service mode is an operating mode the user enables at installation time, whose scope is limited to a declared set of operations in the authenticated policy artifact, whose activation and deactivation are logged, and whose current state is visible in the UI. In elevated service mode, execution of an unsigned or signature-unverifiable binary requires per-execution classification as `CRITICAL_SYSTEM` and mandatory secondary authentication for that specific execution. In no other circumstance shall an unsigned or signature-unverifiable binary be launched, regardless of confirmation. Signed-binary signature verification failure shall be treated as equivalent to unsigned.
  * **Rationale:** Prevents unauthorized User Account Control (UAC) elevation and cross-integrity manipulation. Removes ambiguity about whether confirmation alone authorizes unsigned-binary execution.
  * **Dependencies:** `REQ-WIN-001`, `REQ-SEC-001`
  * **Security Implications:** Prevents privilege escalation on the local system.
  * **Verification Method:** Test.

* **REQ-WIN-003**
  * **Priority:** MUST
  * **Requirement Statement:** File management operations that modify state (Create, Update, Delete, Move, Rename) shall be constrained **strictly** to user-designated workspace directories. This write prohibition is absolute: no write operation outside a workspace shall be authorized regardless of confirmation. Read operations outside workspaces shall be classified at minimum `HIGH_RISK_SIDE_EFFECT` and shall require the corresponding confirmation. Workspace designations shall be stored in the authenticated policy artifact and shall reject any path that resolves inside OS system directories or user credential stores. Path resolution shall canonicalize symbolic links, junctions, short names, case variations, and all alternate path representations (including `..` traversal) before the containment check.
  * **Rationale:** Protects critical OS files and prevents path-traversal sandbox escape. Removes the ambiguity between "strictly contained" and "conditionally permitted."
  * **Dependencies:** `REQ-SEC-001`
  * **Security Implications:** Eliminates arbitrary path traversal and symlink-escape vulnerabilities.
  * **Verification Method:** Test with traversal, symlink, junction, short-path, and case-variation inputs.

* **REQ-WIN-004**
  * **Priority:** MUST
  * **Requirement Statement:** Background operation outside an authenticated user session shall be limited to tasks whose manifests declare no user-credential dependency and no side effect above `READ_ONLY`. Requests requiring user credentials shall queue until an authenticated session is available. On logout, decrypted key material and volatile session state shall be cleared (`REQ-MEM-006`).
  * **Rationale:** Prevents credential-dependent behavior outside a user session.
  * **Dependencies:** `REQ-MEM-006`, `REQ-SEC-001`
  * **Security Implications:** Preserves user-session trust boundary.
  * **Verification Method:** Test.

* **REQ-WIN-005**
  * **Priority:** MUST
  * **Requirement Statement:** Actions that modify autostart locations, browser profiles, file associations, scheduled tasks, or security-relevant configuration shall be classified at minimum `HIGH_RISK_SIDE_EFFECT` regardless of reversibility. This is a floor, not an exact classification: if any other requirement places the operation at a higher tier, the higher tier applies (see `REQ-SEC-001c`).
  * **Rationale:** Reversible local modifications can still establish persistence or weaken security.
  * **Dependencies:** `REQ-SEC-001`, `REQ-SEC-001c`
  * **Security Implications:** Closes the "reversible = safe" escalation class.
  * **Verification Method:** Test.

---

### 3.8 Tools & Extensibility Framework

* **REQ-EXT-001**
  * **Priority:** MUST
  * **Requirement Statement:** The system shall implement an isolated plugin/extensibility architecture where every tool exposes a strongly typed manifest detailing inputs, outputs, side effects, risk class, required scopes, resource keys (`REQ-CORE-003`), simulation capability (`REQ-SIM-001a`), idempotency (`REQ-EXT-005`), deduplication-key schema (`REQ-EXT-007`), and uncertainty classes (`REQ-SIM-002a`). The manifest is the authoritative declaration of the tool's security posture.
  * **Rationale:** Enables third-party integration while maintaining strict runtime security boundary enforcement.
  * **Dependencies:** `REQ-SEC-001`
  * **Security Implications:** Malicious tools are strictly scoped to declared capability manifests.
  * **Verification Method:** Inspection and Test.

* **REQ-EXT-002**
  * **Priority:** MUST
  * **Requirement Statement:** Plugins shall run in sandboxed sub-processes with restricted local system permissions. A plugin sub-process shall execute with a token whose privileges are a strict subset of the invoking user's interactive token; shall be unable to read or write any path outside its declared workspace; shall be unable to acquire network access beyond its declared endpoints; and shall be unable to signal or inspect other plugin sub-processes.
  * **Rationale:** A compromise inside an extension must not yield full user-account compromise.
  * **Dependencies:** `REQ-EXT-001`
  * **Security Implications:** Out-of-bounds memory or OS accesses by plugins are blocked by process boundaries.
  * **Verification Method:** Test (isolation verification).

* **REQ-EXT-002a**
  * **Priority:** MUST
  * **Requirement Statement:** Violation of any isolation bound defined in `REQ-EXT-002` shall be detectable by a verification test. The system shall not represent a plugin as sandboxed unless each bound is independently testable.
  * **Rationale:** "Sandboxed" is an asserted property until it is testable.
  * **Dependencies:** `REQ-EXT-002`
  * **Security Implications:** Prevents false assurance.
  * **Verification Method:** Inspection of test coverage per bound.

* **REQ-EXT-003**
  * **Priority:** MUST
  * **Requirement Statement:** Every tool manifest shall be authenticated by its publisher identity. The system shall verify manifest authenticity and publisher authorization scope before loading the tool. Unauthenticated or authenticity-failing manifests shall be rejected. The specific authenticity mechanism shall be selected in the architecture phase.
  * **Rationale:** A modified manifest can silently change the security posture of a tool.
  * **Dependencies:** `REQ-EXT-001`
  * **Security Implications:** Blocks silent downgrade of tool risk class.
  * **Verification Method:** Test with tampered manifests.

* **REQ-EXT-004**
  * **Priority:** MUST
  * **Requirement Statement:** Manifest risk classifications shall be immutable at runtime. Any attempted runtime modification of a loaded manifest's security-relevant fields shall be treated as a security event, logged, and shall cause the affected tool to be unloaded.
  * **Rationale:** Prevents in-memory or on-disk tampering after load.
  * **Dependencies:** `REQ-EXT-003`
  * **Security Implications:** Reinforces `REQ-AI-008` classification integrity.
  * **Verification Method:** Test.

* **REQ-EXT-005**
  * **Priority:** MUST
  * **Requirement Statement:** Every tool manifest shall declare whether the tool is idempotent and, if not, what deduplication-key schema (if any) the caller must supply. Automatic retries shall be permitted only for tools declared idempotent or with a valid deduplication key.
  * **Rationale:** Without an idempotency declaration, retries can duplicate side effects.
  * **Dependencies:** `REQ-EXT-001`, `REQ-REL-002`
  * **Security Implications:** Prevents duplicate side effects.
  * **Verification Method:** Test.

* **REQ-EXT-006**
  * **Priority:** MUST
  * **Requirement Statement:** Plugins shall communicate only with the control core, never directly with other plugins. Plugin outputs that enter a plan shall be tagged with provenance (`REQ-SEC-005`) and shall not be able to raise the privilege of any downstream node.
  * **Rationale:** Prevents confused-deputy attacks across plugins.
  * **Dependencies:** `REQ-EXT-002`, `REQ-SEC-005`
  * **Security Implications:** Contains cross-plugin privilege composition.
  * **Verification Method:** Test.

* **REQ-EXT-007**
  * **Priority:** MUST
  * **Requirement Statement:** Every plan node that can produce a non-idempotent side effect shall carry a deduplication key derived from the operation's semantic identity (tool identity and logical operation parameters), not from the node instance identity. Replanned, recovered, retried, or re-issued nodes representing the same logical operation shall carry the same deduplication key. The system shall refuse to re-issue an operation whose deduplication key matches a previously observed operation on the same resource unless the previous outcome is definitively `Failed`. User-initiated retries shall be subject to the same deduplication check as automatic retries. Recovery and replanning shall preserve deduplication-key identity across the mapping maintained by `REQ-REL-006`.
  * **Rationale:** Prevents the same real-world side effect from being duplicated through replan, recovery, timeout, or user retry paths.
  * **Dependencies:** `REQ-EXT-005`, `REQ-REL-006`, `REQ-REL-003b`, `REQ-AI-004`
  * **Security Implications:** Closes duplication-based authorization bypass.
  * **Verification Method:** Test through all re-issuance paths.

* **REQ-EXT-008**
  * **Priority:** MUST
  * **Requirement Statement:** The system shall detect and log idempotency mismatches when observed (e.g., duplicate side effects reported by the tool itself or observed downstream). A tool that repeatedly produces idempotency mismatches shall have its idempotency declaration revoked pending review, and automatic retries for that tool shall be suspended. Restoration of the idempotency declaration requires either (a) re-publication of the manifest by the authenticated publisher with a corrected declaration, or (b) explicit user re-enablement recorded in the audit log.
  * **Rationale:** Idempotency declarations are trusted assertions; unverified trust in them creates duplication risk.
  * **Dependencies:** `REQ-EXT-005`, `REQ-EXT-007`
  * **Security Implications:** Provides a feedback loop for false idempotency claims.
  * **Verification Method:** Test with a tool that falsely declares idempotency.

---

### 3.9 Android Companion Capabilities (Secondary Platform)

* **REQ-AND-001**
  * **Priority:** SHOULD
  * **Requirement Statement:** The system shall support a lightweight Android companion client capable of transmitting encrypted voice/text intents and receiving execution status notifications from the Windows host. The companion is treated as an untrusted remote command source until authenticated and authorized for the current session.
  * **Rationale:** Enables remote access while establishing the trust posture.
  * **Dependencies:** `REQ-CORE-001`, `REQ-AND-002`
  * **Security Implications:** Remote channels expose the system to network-borne attacks if unauthenticated.
  * **Verification Method:** Test.

* **REQ-AND-002**
  * **Priority:** MUST if `REQ-AND-001` is implemented; otherwise Not Applicable
  * **Requirement Statement:** Pairing between the Windows host and the Android companion shall require an out-of-band cryptographic handshake establishing a revocable trust credential. The trust credential shall have a defined maximum lifetime and shall be revocable from both host and companion. Host reinstall or companion uninstall shall invalidate the prior pairing. The pairing ceremony shall require physical proximity or an out-of-band channel independent of the network path used for command submission.
  * **Rationale:** Blocks unauthorized rogue mobile clients and bounds credential lifetime.
  * **Dependencies:** `REQ-AND-001`
  * **Security Implications:** Guarantees zero unauthenticated remote command submission.
  * **Verification Method:** Test.

* **REQ-AND-003**
  * **Priority:** MUST (conditional on `REQ-AND-001`)
  * **Requirement Statement:** All companion-originated commands shall be integrity-protected and non-replayable. The host shall reject commands whose sequence number, nonce, or timestamp falls outside an accepted window. Transport integrity and replay-window mechanics shall be selected in the architecture phase.
  * **Rationale:** Prevents re-injection of previously authorized commands.
  * **Dependencies:** `REQ-AND-002`
  * **Security Implications:** Closes replay-based command injection.
  * **Verification Method:** Test.

* **REQ-AND-004**
  * **Priority:** MUST (conditional on `REQ-AND-001`)
  * **Requirement Statement:** Commands originating from the companion and from the host shall enter the same serialized request queue (`REQ-CORE-003`). A companion command shall not preempt or interleave with an in-flight host-originated write.
  * **Rationale:** Prevents race conditions across channels.
  * **Dependencies:** `REQ-AND-003`, `REQ-CORE-003`
  * **Security Implications:** Preserves serialization guarantees.
  * **Verification Method:** Test.

* **REQ-AND-005**
  * **Priority:** MUST (conditional on `REQ-AND-001`)
  * **Requirement Statement:** The companion shall verify the host's identity on every session establishment using the credential established during pairing. Session establishment shall fail closed if host identity cannot be verified.
  * **Rationale:** Command-channel integrity is bidirectional.
  * **Dependencies:** `REQ-AND-002`
  * **Security Implications:** Prevents companion redirection.
  * **Verification Method:** Test.

* **REQ-AND-006**
  * **Priority:** MUST (conditional on `REQ-AND-001`)
  * **Requirement Statement:** The host shall provide a user-accessible revocation path that invalidates a paired device's credential immediately and without requiring the device's cooperation. Revocation shall be reflected in the host's authorization state before the next command is processed.
  * **Rationale:** A lost or stolen device must not remain a permanent command channel.
  * **Dependencies:** `REQ-AND-002`
  * **Security Implications:** Bounds the impact of device loss.
  * **Verification Method:** Test.

* **REQ-AND-007**
  * **Priority:** MUST (conditional on `REQ-AND-001`)
  * **Requirement Statement:** The companion channel shall minimize host-side network exposure. If the host must listen, it shall bind only to authenticated, integrity-protected sessions and shall reject unauthenticated traffic without processing.
  * **Rationale:** Reduces attack surface.
  * **Dependencies:** `REQ-AND-002`, `REQ-AND-003`
  * **Security Implications:** Minimizes network-borne attack surface.
  * **Verification Method:** Inspection and Test.

* **REQ-AND-008**
  * **Priority:** MUST (conditional on `REQ-AND-001`)
  * **Requirement Statement:** Companion sessions shall expire after a configurable idle interval. Expired sessions shall require re-authentication using the paired credential and shall not silently persist.
  * **Rationale:** Bounds session lifetime.
  * **Dependencies:** `REQ-AND-002`
  * **Security Implications:** Reduces window for session hijack.
  * **Verification Method:** Test.

* **REQ-AND-009**
  * **Priority:** MUST (conditional on `REQ-AND-001`)
  * **Requirement Statement:** Commands originating from the companion which would be classified `HIGH_RISK_SIDE_EFFECT` or `CRITICAL_SYSTEM` shall require an additional host-side confirmation independent of the companion, so that companion compromise does not alone authorize high-risk actions. The host-side confirmation shall satisfy `REQ-SEC-007`.
  * **Rationale:** The host cannot distinguish a legitimate companion command from a compromised-companion command.
  * **Dependencies:** `REQ-AND-001`, `REQ-SEC-007`, `REQ-CORE-004`
  * **Security Implications:** Bounds the impact of companion compromise.
  * **Verification Method:** Test with a simulated compromised companion.

---

### 3.10 Security, Privacy & Permission Architecture

* **REQ-SEC-001**
  * **Priority:** MUST
  * **Requirement Statement:** System actions shall be categorized into four strict risk classifications:
      1. `READ_ONLY` (No state change; auto-approved)
      2. `LOW_RISK_SIDE_EFFECT` (Non-critical, reversible local modification; auto-approval permitted only via the authenticated policy artifact)
      3. `HIGH_RISK_SIDE_EFFECT` (Financial, sensitive-file deletion, outbound communication, persistence installation, security-relevant configuration; mandatory explicit human confirmation)
      4. `CRITICAL_SYSTEM` (OS settings modification, execution of unsigned binaries, elevation-requiring operations; mandatory explicit confirmation with mandatory secondary authentication for unsigned-binary execution and for elevation-requiring operations; optional secondary authentication otherwise)
  * **Rationale:** Prevents AI agents from autonomously making high-impact or destructive changes. Mandatory secondary authentication for unsigned-binary execution and elevated operations removes the previous "optional" ambiguity.
  * **Dependencies:** `REQ-CORE-001`, `REQ-SEC-002`, `REQ-SEC-001c`
  * **Security Implications:** Primary core defense against AI runaways and prompt injection.
  * **Verification Method:** Demonstration and Test.

* **REQ-SEC-001a**
  * **Priority:** MUST
  * **Requirement Statement:** The auto-approval policy for `LOW_RISK_SIDE_EFFECT` shall be defined in an authenticated, versioned policy artifact that is separate from AI-generated content and from memory records. The policy artifact shall be modifiable only via a `HIGH_RISK_SIDE_EFFECT` authorization path. The specific authenticity mechanism shall be selected in the architecture phase.
  * **Rationale:** "Policy-guided" requires a defined, protected policy source.
  * **Dependencies:** `REQ-SEC-001`, `REQ-EXT-003`
  * **Security Implications:** Prevents policy injection.
  * **Verification Method:** Inspection and Test.

* **REQ-SEC-001b**
  * **Priority:** MUST
  * **Requirement Statement:** Risk classification shall be re-evaluated against actual parameter values at execution time. Any parameter change that would raise the tier shall suspend execution and require the higher tier's confirmation.
  * **Rationale:** Parameter changes can silently raise risk.
  * **Dependencies:** `REQ-SEC-001`, `REQ-AI-008`
  * **Security Implications:** Closes parameter-based tier downgrade.
  * **Verification Method:** Test.

* **REQ-SEC-001c**
  * **Priority:** MUST
  * **Requirement Statement:** When more than one risk classification criterion applies to an operation, the operation shall be classified at the highest applicable tier. Requirements that enumerate specific action categories (e.g., `REQ-WIN-005`) specify minimum classifications; the effective classification is the maximum of all applicable tiers. No requirement shall be read to lower the classification determined by another requirement. The classification decision shall be logged with the applied criteria.
  * **Rationale:** Prevents the same operation from receiving contradictory classifications depending on which requirement an implementer follows.
  * **Dependencies:** `REQ-SEC-001`
  * **Security Implications:** Closes classification-downgrade path.
  * **Verification Method:** Test with overlapping classifications.

* **REQ-SEC-002**
  * **Priority:** MUST
  * **Requirement Statement:** Under no circumstances shall an AI model output have direct authority to execute privileged, `HIGH_RISK_SIDE_EFFECT`, or `CRITICAL_SYSTEM` actions. Execution authority resides exclusively in the deterministic Security Policy Evaluator in the core. No AI-generated field shall be an input to an authorization decision; risk classification shall be derived from the tool manifest and actual parameter values (`REQ-AI-008`).
  * **Rationale:** Generative models are non-deterministic and susceptible to prompt injections.
  * **Dependencies:** `REQ-SEC-001`, `REQ-AI-008`
  * **Security Implications:** Protects core authority boundaries.
  * **Verification Method:** Analysis and Test.

* **REQ-SEC-003**
  * **Priority:** MUST
  * **Requirement Statement:** Secret management (API keys, credentials, OAuth tokens) shall be backed by a native OS-protected credential store. Secrets shall never be written into plain-text logs, context prompts, volatile dynamic plans, memory records, or confirmation displays. The specific store shall be selected in the architecture phase.
  * **Rationale:** Prevents credential exposure to remote LLM providers or local log leaks.
  * **Dependencies:** `REQ-SEC-008`
  * **Security Implications:** Mitigates secret exposure.
  * **Verification Method:** Inspection and static analysis.

* **REQ-SEC-004**
  * **Priority:** MUST
  * **Requirement Statement:** Cached runtime state and historical session approvals shall never bypass security checks for subsequent execution cycles. Every task execution cycle shall evaluate credentials and authorizations anew. Authorization re-evaluation shall not proceed for any plan containing an unresolved `Ambiguous_Uncertain` node; resolution shall precede further authorization decisions.
  * **Rationale:** Direct defense against permission bypass through cached state and TOCTOU.
  * **Dependencies:** `REQ-SEC-001`, `REQ-SIM-002`
  * **Security Implications:** Closes TOCTOU permission escalation vectors.
  * **Verification Method:** Test.

* **REQ-SEC-005**
  * **Priority:** MUST
  * **Requirement Statement:** Content originating from untrusted sources — including web pages, external APIs, external documents, OCR/screen content, plugin/tool outputs, external model responses, and memory records derived from any such source — shall be tagged with provenance metadata. Provenance shall propagate through value transformations: any value derived from a tainted source shall be tainted; any operation whose parameters or tool selection are influenced by a tainted value shall be flagged. Provenance shall not propagate through derived metadata that does not itself carry tainted content (lengths, type tags, structural positions, timestamps of derivation), unless that metadata is used as a parameter value that influences the operation's semantics. Plan nodes whose parameters or tool selection derive from untrusted content shall be flagged. A flagged node that would execute a write action shall require explicit human confirmation regardless of its manifest-declared risk tier. Untrusted content shall not be able to modify tool manifests, policy artifacts, risk classifications, or authorization state. The architecture phase may refine the propagation table for derived metadata without weakening the containment property.
  * **Rationale:** Prompt injection is the primary attack against an autonomous agent. Transitively propagated taint closes the laundering-through-intermediary vector while bounded propagation prevents over-tainting.
  * **Dependencies:** `REQ-AI-008`, `REQ-SEC-001`, `REQ-EXT-003`, `REQ-MEM-007`
  * **Security Implications:** Primary containment for untrusted-content-borne attacks.
  * **Verification Method:** Test with adversarial untrusted content across source classes; test with laundering through summary and memory; test that derived metadata does not over-propagate.

* **REQ-SEC-006**
  * **Priority:** MUST
  * **Requirement Statement:** Security-relevant time comparisons (replay windows, credential expiry, log ordering) shall be resistant to wall-clock manipulation. If the wall clock jumps backward by more than a configured threshold, security-relevant decisions shall fail closed. Log timestamps shall be subject to the same manipulation-resistance property.
  * **Rationale:** Prevents clock-based bypass of replay and expiry protections.
  * **Dependencies:** `REQ-AND-003`, `REQ-DEP-002`
  * **Security Implications:** Closes clock-manipulation bypass.
  * **Verification Method:** Test.

* **REQ-SEC-007**
  * **Priority:** MUST
  * **Requirement Statement:** Confirmation prompts for `HIGH_RISK_SIDE_EFFECT` and `CRITICAL_SYSTEM` actions shall describe the actual authorized operation: tool identity, manifest-derived risk tier, resource keys, and parameter values. The description shall be generated from authenticated system data, not from AI-generated text. Confirmation shall require a fresh user gesture, shall be bound to a specific request identifier and operation digest (`REQ-CORE-004`), shall not be auto-accepted, shall not time-out-accept, and shall not persist across requests. Secret-bearing parameters shall be handled per `REQ-SEC-008`. This requirement is co-primary with `REQ-CORE-004`; the two together define the confirmation contract — `REQ-CORE-004` defines binding and integrity, `REQ-SEC-007` defines display and freshness.
  * **Rationale:** The confirmation prompt is the last line of defense; a spoofable or stale confirmation defeats it.
  * **Dependencies:** `REQ-CORE-004`, `REQ-SEC-001`, `REQ-AI-008`, `REQ-SEC-008`
  * **Security Implications:** Blocks confirmation spoofing and stale-confirmation replay.
  * **Verification Method:** Test.

* **REQ-SEC-007a**
  * **Priority:** MUST
  * **Requirement Statement:** Pending confirmation requests shall have a bounded lifetime. On expiry, the request shall transition to `Cancelled`, not `Approved`. The user shall be notified at the next authenticated user interaction on the same session that pending confirmations expired. Expiry shall not trigger the underlying operation.
  * **Rationale:** Prevents indefinite pending state and prevents time-out from becoming a silent approval path.
  * **Dependencies:** `REQ-SEC-007`, `REQ-CORE-002`
  * **Security Implications:** Closes time-out-as-approval vector.
  * **Verification Method:** Test.

* **REQ-SEC-008**
  * **Priority:** MUST
  * **Requirement Statement:** Confirmation prompts shall display parameter values in a form that allows the user to identify the actual operation. Secret-bearing parameters (credentials, passwords, tokens, private keys) shall be displayed using an identifier that identifies which secret is being used without revealing any portion of the secret and without functioning as a partial-reveal oracle. Acceptable forms include the credential's human-readable name or a non-invertible identifier that cannot be compared against guessed secrets. Displaying last-N characters or other partial content is not permitted. The operation digest required by `REQ-CORE-004` shall incorporate the secret's contribution without exposing the secret. Secret parameter values shall never be persisted to logs, memory, volatile plans, or the confirmation display itself.
  * **Rationale:** Reconciliation of `REQ-SEC-003` (no secret exposure) with `REQ-SEC-007` (accurate confirmation) without introducing a partial-reveal oracle.
  * **Dependencies:** `REQ-SEC-003`, `REQ-SEC-007`, `REQ-CORE-004`
  * **Security Implications:** Prevents secret leakage via confirmation UI; preserves confirmation accuracy.
  * **Verification Method:** Test with confirmation payloads containing secret parameters.

---

### 3.11 Simulation & Uncertainty Engine

* **REQ-SIM-001**
  * **Priority:** MUST
  * **Requirement Statement:** The system shall support a dry-run simulation mode that **attempts to execute** any generated action plan within an isolated mock environment, emitting anticipated side effects without applying changes to the real OS or network targets. No simulation-mode invocation shall acquire a real OS handle, network socket, credential, or other real-world capability. Simulation shall never accidentally apply real-world side effects. The system shall not represent a plan as fully validated when any node is not `FULLY_SIMULATABLE`.
  * **Rationale:** Allows verification of complex AI workflows prior to real-world execution, while honoring the reality that not all tools can be simulated.
  * **Dependencies:** `REQ-AI-003`, `REQ-CORE-001`, `REQ-SIM-001a`
  * **Security Implications:** Provides a non-destructive verification boundary.
  * **Verification Method:** Test.

* **REQ-SIM-001a**
  * **Priority:** MUST
  * **Requirement Statement:** Every tool manifest shall declare a simulation capability level: `FULLY_SIMULATABLE`, `PARTIALLY_SIMULATABLE`, or `NOT_SIMULATABLE`. The system shall not represent a simulation as complete when any node is not `FULLY_SIMULATABLE`.
  * **Rationale:** Simulation fidelity varies by tool; users must know when verification is partial.
  * **Dependencies:** `REQ-SIM-001`, `REQ-EXT-001`
  * **Security Implications:** Prevents false confidence from partial simulation.
  * **Verification Method:** Inspection and Test.

* **REQ-SIM-001b**
  * **Priority:** MUST
  * **Requirement Statement:** Plans containing any `NOT_SIMULATABLE` or `PARTIALLY_SIMULATABLE` node shall be marked `SIMULATION_INCOMPLETE`. Real execution of such plans shall require explicit user acknowledgment that dry-run verification is partial. The acknowledgment shall identify the specific nodes whose simulation was incomplete and shall state that those nodes carry unverified risk. `SIMULATION_INCOMPLETE` plans shall not be represented as validated.
  * **Rationale:** Communicates simulation limitation before real execution.
  * **Dependencies:** `REQ-SIM-001a`
  * **Security Implications:** Prevents authorization based on false simulation completeness.
  * **Verification Method:** Test.

* **REQ-SIM-002**
  * **Priority:** MUST
  * **Requirement Statement:** When tool execution yields non-deterministic, ambiguous, or unverifiable state responses (e.g., network timeout during an external POST, partial file write, unacknowledged IPC), the system shall flag the action status as `Ambiguous_Uncertain` and halt downstream dependent actions until human confirmation or verified state inspection occurs. When any node enters `Ambiguous_Uncertain`, the system shall (a) prevent new node starts, (b) allow only nodes declared safe-to-complete to finish, (c) mark all other in-flight nodes for cancellation, and (d) present the full plan state to the user before any further execution. Execution ownership (`REQ-CORE-005`) over affected resource keys shall not be released until the ambiguity is resolved.
  * **Rationale:** Prevents the system from assuming success when outcome state is unknown.
  * **Dependencies:** `REQ-CORE-002`, `REQ-EXT-005`, `REQ-CORE-005`
  * **Security Implications:** Stops unsafe retries and duplicate side effects.
  * **Verification Method:** Fault-injection test.

* **REQ-SIM-002a**
  * **Priority:** MUST
  * **Requirement Statement:** The system shall define a taxonomy of uncertain outcomes with per-class recovery rules. Tool manifests shall declare which uncertainty classes apply to that tool. The taxonomy shall be versioned and inspectable.
  * **Rationale:** Recovery behavior differs per ambiguity class; a single state cannot drive correct recovery.
  * **Dependencies:** `REQ-SIM-002`
  * **Security Implications:** Prevents uniform handling of non-uniform failures.
  * **Verification Method:** Inspection and Test.

* **REQ-SIM-002b**
  * **Priority:** MUST
  * **Requirement Statement:** A tool that cannot determine its outcome state shall return a structured `Ambiguous_Uncertain` result with the applicable uncertainty class. Returning a generic failure or generic success in an ambiguous condition shall be a manifest violation.
  * **Rationale:** Enables the system to route recovery per class.
  * **Dependencies:** `REQ-SIM-002a`, `REQ-EXT-001`
  * **Security Implications:** Prevents loss of ambiguity information.
  * **Verification Method:** Test.

* **REQ-SIM-002c**
  * **Priority:** MUST
  * **Requirement Statement:** The user interface shall distinguish `Failed`, `Ambiguous_Uncertain`, and `Completed` with distinct presentation and shall not represent `Ambiguous_Uncertain` as either failure or success.
  * **Rationale:** A user who cannot distinguish outcomes cannot make a safe decision.
  * **Dependencies:** `REQ-SIM-002`
  * **Security Implications:** Preserves user decision quality.
  * **Verification Method:** Demonstration.

* **REQ-SIM-003**
  * **Priority:** SHOULD (conditional on `REQ-AND-001`)
  * **Requirement Statement:** Dry-run simulation shall be available for plans regardless of origination channel. Companion-originated plans classified above `READ_ONLY` shall be simulatable before the user confirms execution.
  * **Rationale:** Remote commands have the same side-effect risk as local ones.
  * **Dependencies:** `REQ-SIM-001`, `REQ-AND-001`
  * **Security Implications:** Enables remote safe-preview.
  * **Verification Method:** Test.

---

### 3.12 Reliability, Fault Tolerance & Recovery

* **REQ-REL-001**
  * **Priority:** MUST
  * **Requirement Statement:** Tool execution calls shall implement configurable execution-timeout thresholds, declared per tool in the manifest and bounded by system-level minimum and maximum values in the authenticated policy artifact. Upon reaching a timeout, the system shall revoke the operation's execution ownership per `REQ-CORE-005` and attempt to terminate the task sub-process. Cancellation success shall be observable as: the sub-process is no longer able to produce side effects on its declared resource keys within a bounded interval after the timeout. If termination cannot be confirmed within that interval, the node shall transition to `Ambiguous_Uncertain` unless the tool manifest declares the operation idempotent and safely abortable, in which case the node may be transitioned to `Timed_Out`.
  * **Rationale:** Prevents deadlocked processes while preserving ambiguity where outcome is unknown. Provides an observable cancellation contract.
  * **Dependencies:** `REQ-CORE-001`, `REQ-EXT-005`, `REQ-CORE-005`
  * **Security Implications:** Mitigates resource starvation and avoids unsafe timeout assumptions.
  * **Verification Method:** Test including worker-ignores-termination and worker-survives-core-restart.

* **REQ-REL-002**
  * **Priority:** MUST
  * **Requirement Statement:** The system shall prohibit automatic background retry for any failed or ambiguous operation that carries a `HIGH_RISK_SIDE_EFFECT` or `CRITICAL_SYSTEM` classification without explicit human authorization. Retries of `LOW_RISK_SIDE_EFFECT` operations shall be permitted only for tools declared idempotent or with a valid deduplication key, and only if no deduplication match against a previously observed operation exists on the same resource.
  * **Rationale:** Prevents unsafe retries of partially completed side-effecting operations.
  * **Dependencies:** `REQ-SEC-001`, `REQ-SIM-002`, `REQ-EXT-005`, `REQ-EXT-007`
  * **Security Implications:** Eliminates automated compounding side effects.
  * **Verification Method:** Test.

* **REQ-REL-003**
  * **Priority:** MUST
  * **Requirement Statement:** Upon restart after ungraceful termination, the system shall recover all tasks that were not in a terminal state to `Halted_Requires_Inspection` and shall present a state summary to the user before processing new requests. The recovery mechanism shall guarantee that no task is resumed without user inspection.
  * **Rationale:** Protects against blind resumption of dangerous tasks following unexpected termination.
  * **Dependencies:** `REQ-CORE-002`
  * **Security Implications:** Mitigates unsafe crash-recovery assumptions.
  * **Verification Method:** Crash-test simulation.

* **REQ-REL-003a**
  * **Priority:** MUST
  * **Requirement Statement:** The post-crash summary shall present each plan node with its last known state, whether its side effect was applied, whether duplication risk exists, and whether the node is safe to resume, retry, or abandon. **Safe-to-resume** shall mean: the node is idempotent, or a valid deduplication key exists, or the node's side effect is verifiably unapplied. **Safe-to-retry** shall mean: the node is idempotent or has a valid deduplication key. **Safe-to-abandon** shall mean: the node's side effect is verifiably unapplied or the node is not on the critical path. The system shall not automatically resume any plan containing a node whose outcome is `Ambiguous_Uncertain` or `Halted_Requires_Inspection`.
  * **Rationale:** Per-node reconciliation is required for safe continuation; safe-to-X terms are operationally defined.
  * **Dependencies:** `REQ-REL-003`, `REQ-SIM-002`, `REQ-EXT-007`
  * **Security Implications:** Prevents unsafe auto-resumption and provides testable criteria.
  * **Verification Method:** Crash-test simulation with per-node assertions.

* **REQ-REL-003b**
  * **Priority:** MUST
  * **Requirement Statement:** Manual resumption or manual retry of a `Halted_Requires_Inspection` node shall be subject to the same deduplication check as automatic retry (per `REQ-EXT-007`). The post-crash summary shall state, per node, whether duplication risk exists and the basis for the determination. The cumulative side-effect budget (`REQ-AI-004a`) shall be reconstructed from the audit log on recovery.
  * **Rationale:** User-initiated resumption after recovery is a duplication path that `REQ-EXT-005` does not cover.
  * **Dependencies:** `REQ-REL-003a`, `REQ-EXT-007`, `REQ-AI-004a`
  * **Security Implications:** Closes crash-recovery duplication path.
  * **Verification Method:** Crash-test simulation with manual-resume scenarios.

* **REQ-REL-004**
  * **Priority:** MUST
  * **Requirement Statement:** The system shall enforce configurable upper bounds on concurrent active requests, queued requests, per-tool memory consumption, and total tool sub-process count. Exceeding any bound shall fail closed and shall not silently queue unbounded work. Bounds shall be defined in the authenticated policy artifact.
  * **Rationale:** Prevents resource starvation and runaway planning.
  * **Dependencies:** `REQ-CORE-003`, `REQ-SEC-001a`
  * **Security Implications:** Mitigates local resource-starvation DoS.
  * **Verification Method:** Test.

* **REQ-REL-005**
  * **Priority:** MUST
  * **Requirement Statement:** A tool sub-process that exits without a structured outcome, or that crashes, shall be recorded as `Ambiguous_Uncertain` unless its manifest declares the operation idempotent and safely abortable, in which case it may be recorded as `Failed`.
  * **Rationale:** A crashed tool is indistinguishable from an ambiguous outcome unless declared otherwise.
  * **Dependencies:** `REQ-SIM-002`, `REQ-EXT-005`
  * **Security Implications:** Prevents silent outcome loss on crash.
  * **Verification Method:** Fault-injection test.

* **REQ-REL-006**
  * **Priority:** MUST
  * **Requirement Statement:** Every plan node shall be assigned a stable node identity at planning time. When replanning replaces a node, either the replacement inherits the original node's identity, or a mapping from original to replacement is maintained and exposed to recovery, deduplication, and audit. If a mapping is used, the deduplication check (`REQ-EXT-007`) shall consult the mapping to determine whether the replacement represents the same logical operation. The mapping shall persist across crash recovery.
  * **Rationale:** Stable identity is required for safe recovery, deduplication, and audit traceability across replan boundaries.
  * **Dependencies:** `REQ-CORE-002`, `REQ-REL-003`, `REQ-EXT-007`
  * **Security Implications:** Prevents duplicate execution via identity confusion.
  * **Verification Method:** Inspection and Test across replan boundaries.

---

### 3.13 Deployment, Telemetry & Logs

* **REQ-DEP-001**
  * **Priority:** MUST
  * **Requirement Statement:** The Windows production target shall be deployable as an isolated local executable package requiring no system-wide developer runtime configurations. The package shall declare, in an authenticated deployment manifest, every host-level registration it performs (services, scheduled tasks, shell integrations, firewall rules). Any registration not declared in the manifest shall be treated as a deployment integrity violation.
  * **Rationale:** Simplifies installation and upgrade while preserving auditability.
  * **Dependencies:** `REQ-EXT-003`
  * **Security Implications:** Package dependencies must be authenticated to prevent dependency hijacking.
  * **Verification Method:** Demonstration and Inspection.

* **REQ-DEP-002**
  * **Priority:** MUST
  * **Requirement Statement:** Local audit logs shall record every command request, generated plan, security decision, tool execution parameter, and side-effect outcome in a structured local log format with cryptographic chaining such that removal or modification of any prior entry is detectable by a verification procedure. The tamper-evidence property shall detect at minimum: entry modification, entry deletion, trailing truncation, whole-log rollback or replacement, and partial writes. The chain anchor against which the log is verified shall be protected independently from the log itself, such that a writer with log-write access cannot unilaterally redefine the anchor. The verification procedure shall be specified and testable independently of the logging implementation. Sensitive data (passwords, tokens) shall be sanitized by local redactors before persistence.
  * **Rationale:** Guarantees post-hoc auditability with a defined threat model for tamper evidence.
  * **Dependencies:** `REQ-CORE-001`, `REQ-SEC-003`, `REQ-SEC-006`
  * **Security Implications:** Sensitive data must not leak into logs; logs must be integrity-protected against truncation, rollback, and anchor replacement.
  * **Verification Method:** Inspection, Audit Test, and tamper-detection verification across all defined modification classes.

* **REQ-DEP-002a**
  * **Priority:** MUST
  * **Requirement Statement:** Audit log storage shall be outside all tool-writable workspaces. No tool, plugin, or AI-originated action shall be able to delete, truncate, or rewrite audit log entries.
  * **Rationale:** A tool with workspace write access must not be able to erase its own trail.
  * **Dependencies:** `REQ-DEP-002`, `REQ-WIN-003`
  * **Security Implications:** Preserves audit integrity.
  * **Verification Method:** Test.

* **REQ-DEP-002b**
  * **Priority:** MUST
  * **Requirement Statement:** Log timestamps shall be protected against manipulation consistent with `REQ-SEC-006`. Verification after restart shall confirm chain integrity against the anchor before new entries are appended. If anchor verification fails, the system shall fail closed on log-dependent authorization decisions and shall present the failure to the user.
  * **Rationale:** Tamper evidence without restart verification and anchor protection is incomplete.
  * **Dependencies:** `REQ-DEP-002`, `REQ-SEC-006`
  * **Security Implications:** Closes restart-time rollback and anchor-replacement vectors.
  * **Verification Method:** Test with rollback and anchor-replacement scenarios.

* **REQ-DEP-003**
  * **Priority:** MUST
  * **Requirement Statement:** All updates to the system or its tool manifests shall be authenticated before application. The system shall reject updates with invalid authenticity, expired publisher credentials, or version numbers lower than the installed version unless the user explicitly authorizes a rollback.
  * **Rationale:** An unauthenticated update path is a direct code-execution vector.
  * **Dependencies:** `REQ-EXT-003`
  * **Security Implications:** Prevents malicious update injection.
  * **Verification Method:** Test.

---

## 4. Requirement Priorities Matrix

| ID | Priority | Category | Short Title |
| :--- | :--- | :--- | :--- |
| `REQ-CORE-001` | MUST | Core | Centralized Control Core |
| `REQ-CORE-002` | MUST | Core | Deterministic Lifecycle State Machine |
| `REQ-CORE-002a` | MUST | Core | Terminal State Lock |
| `REQ-CORE-003` | MUST | Core | Resource-Keyed Serialization |
| `REQ-CORE-003a` | MUST | Core | Replan/Write Interleaving Prohibition |
| `REQ-CORE-004` | MUST | Core | UI Trust Boundary |
| `REQ-CORE-005` | MUST | Core | Execution Ownership / Fencing |
| `REQ-AI-001` | MUST | AI | Provider Substitution Without Core Change |
| `REQ-AI-002` | MUST | AI | Privacy-Hard-Filtered Routing |
| `REQ-AI-003` | MUST | AI | Explicit DAG Planning |
| `REQ-AI-004` | MUST | AI | Replan Against Original Intent |
| `REQ-AI-004a` | MUST | AI | Cumulative Side-Effect Budget |
| `REQ-AI-005` | MUST | AI | External Endpoint Integrity |
| `REQ-AI-006` | MUST | AI | Provider Failure Fail-Closed |
| `REQ-AI-007` | MUST | AI | Endpoint Disclosure |
| `REQ-AI-008` | MUST | AI | Manifest-Derived Classification |
| `REQ-AI-009` | MUST | AI | Model Output Schema Validation |
| `REQ-AI-010` | SHOULD | AI | Model Identity Recording |
| `REQ-AI-011` | MUST | AI | Degraded Offline Mode |
| `REQ-AI-012` | MUST | AI | Quota Exhaustion Fail-Closed |
| `REQ-VOI-001` | MUST | Voice | Pre/Post-Wake Audio Boundary |
| `REQ-VOI-002` | MUST | Voice | Tiered STT Latency Target |
| `REQ-VOI-003` | MUST | Voice | Barge-In With State-Machine Routing |
| `REQ-VOI-004` | MUST | Voice | Microphone Capture Indicator |
| `REQ-VIS-001` | MUST | Vision | Consent-Bounded Capture |
| `REQ-VIS-002` | MUST | Vision | Partial Redaction, No Guarantee |
| `REQ-VIS-003` | MUST | Vision | Screen Text As Untrusted |
| `REQ-MEM-001` | MUST | Memory | Volatile Short-Term Context |
| `REQ-MEM-002` | MUST | Memory | Encrypted Local Memory |
| `REQ-MEM-003` | MUST | Memory | Memory Audit UI With Partition Display |
| `REQ-MEM-004` | MUST | Memory | No Authorization From Memory |
| `REQ-MEM-005` | MUST | Memory | Retention & Minimization |
| `REQ-MEM-006` | MUST | Memory | Per-User Isolation |
| `REQ-MEM-007` | MUST | Memory | Memory Provenance |
| `REQ-BRW-001` | MUST | Browser | Untrusted Retrieval |
| `REQ-BRW-002` | MUST | Browser | Credential-Isolated Automation |
| `REQ-BRW-003` | MUST | Browser | Download/File Boundary |
| `REQ-BRW-004` | MUST | Browser | Web Content Cannot Modify Security State |
| `REQ-BRW-005` | MUST | Browser | Browser External Side-Effect Classification |
| `REQ-WIN-001` | MUST | Windows | Structured-Surface-First Automation |
| `REQ-WIN-002` | MUST | Windows | Cross-Integrity Prohibition |
| `REQ-WIN-003` | MUST | Windows | Absolute Workspace Write Containment |
| `REQ-WIN-004` | MUST | Windows | Background Session Boundary |
| `REQ-WIN-005` | MUST | Windows | Persistence-Relevant Actions Are High-Risk |
| `REQ-EXT-001` | MUST | Extensibility | Complete Manifest Schema |
| `REQ-EXT-002` | MUST | Extensibility | Testable Isolation Bounds |
| `REQ-EXT-002a` | MUST | Extensibility | Isolation Verifiability |
| `REQ-EXT-003` | MUST | Extensibility | Manifest Authenticity |
| `REQ-EXT-004` | MUST | Extensibility | Manifest Immutability at Runtime |
| `REQ-EXT-005` | MUST | Extensibility | Idempotency Declaration |
| `REQ-EXT-006` | MUST | Extensibility | Plugin-to-Plugin Isolation |
| `REQ-EXT-007` | MUST | Extensibility | Stable Deduplication Identity |
| `REQ-EXT-008` | MUST | Extensibility | Idempotency Mismatch Detection |
| `REQ-AND-001` | SHOULD | Android | Encrypted Remote Companion Channel |
| `REQ-AND-002` | MUST-if | Android | Out-of-Band Pairing |
| `REQ-AND-003` | MUST-if | Android | Replay-Protected Commands |
| `REQ-AND-004` | MUST-if | Android | Channel Serialization |
| `REQ-AND-005` | MUST-if | Android | Host Identity Verification |
| `REQ-AND-006` | MUST-if | Android | Revocation Path |
| `REQ-AND-007` | MUST-if | Android | Network Exposure Minimization |
| `REQ-AND-008` | MUST-if | Android | Session Expiry |
| `REQ-AND-009` | MUST-if | Android | Companion-Compromise Mitigation |
| `REQ-SEC-001` | MUST | Security | Four-Tier Risk Framework |
| `REQ-SEC-001a` | MUST | Security | Authenticated Policy Artifact |
| `REQ-SEC-001b` | MUST | Security | Parameter-Based Tier Re-Evaluation |
| `REQ-SEC-001c` | MUST | Security | Classification Precedence |
| `REQ-SEC-002` | MUST | Security | AI Privilege Deprivation |
| `REQ-SEC-003` | MUST | Security | OS-Protected Credential Store |
| `REQ-SEC-004` | MUST | Security | No Cached Permission Bypass |
| `REQ-SEC-005` | MUST | Security | Untrusted-Content Provenance & Containment |
| `REQ-SEC-006` | MUST | Security | Clock-Manipulation Resistance |
| `REQ-SEC-007` | MUST | Security | Confirmation Integrity & Binding |
| `REQ-SEC-007a` | MUST | Security | Confirmation Lifetime |
| `REQ-SEC-008` | MUST | Security | Secret Parameters in Confirmation |
| `REQ-SIM-001` | MUST | Simulation | Isolated Dry-Run (Attempt) |
| `REQ-SIM-001a` | MUST | Simulation | Fidelity Declaration |
| `REQ-SIM-001b` | MUST | Simulation | Incomplete-Simulation Communication |
| `REQ-SIM-002` | MUST | Simulation | Ambiguity Halting Engine |
| `REQ-SIM-002a` | MUST | Simulation | Uncertainty Taxonomy |
| `REQ-SIM-002b` | MUST | Simulation | Structured Ambiguity Reporting |
| `REQ-SIM-002c` | MUST | Simulation | User-Facing Uncertainty Distinction |
| `REQ-SIM-003` | SHOULD | Simulation | Companion Dry-Run |
| `REQ-REL-001` | MUST | Reliability | Timeout With Observable Cancellation |
| `REQ-REL-002` | MUST | Reliability | Retry Prohibition on Side Effects |
| `REQ-REL-003` | MUST | Reliability | Safe Crash Recovery |
| `REQ-REL-003a` | MUST | Reliability | Per-Node Reconciliation |
| `REQ-REL-003b` | MUST | Reliability | Crash Recovery Deduplication |
| `REQ-REL-004` | MUST | Reliability | Resource Bounds |
| `REQ-REL-005` | MUST | Reliability | Tool Crash Classification |
| `REQ-REL-006` | MUST | Reliability | Stable Node Identity Across Replans |
| `REQ-DEP-001` | MUST | Deployment | Declared-Registration Packaging |
| `REQ-DEP-002` | MUST | Deployment | Chained Tamper-Evident Logging |
| `REQ-DEP-002a` | MUST | Deployment | Log Storage Isolation |
| `REQ-DEP-002b` | MUST | Deployment | Anchor Verification & Timestamp Protection |
| `REQ-DEP-003` | MUST | Deployment | Authenticated Updates |

---

## 5. Non-Goals

1. **Fully Unsupervised System Administration:** JARVIS will not perform unprompted administrative or system-altering actions without human oversight or user-defined explicit policy triggers.
2. **Bypassing Operating System Security:** JARVIS will not exploit local OS vulnerabilities or intentionally bypass elevation prompts without standard OS administrative interaction.
3. **Real-Time Cloud Raw Audio Streaming:** Ambient listening will not rely on continuous un-triggered audio upload to remote services.
4. **Proprietary Locked Ecosystem:** The project will not bind execution permanently to any single commercial vendor API or hardware ecosystem.
5. **Perfect Simulation Fidelity:** JARVIS does not claim that dry-run simulation reproduces all real-world behavior; simulation limitations shall be declared and communicated.
6. **Perfect Redaction Guarantee:** JARVIS does not claim that local redaction of screen captures guarantees removal of all sensitive content; redaction is a reduction layer, not a guarantee.
7. **Perfect Audit Log Immutability:** JARVIS claims tamper *evidence*, not tamper *prevention*. The audit log's integrity property is that modification is detectable, not that it is impossible.

---

## 6. Separated Sections (Out-of-Scope for Requirements Baseline)

### 6.1 Architecture Decisions (To Be Evaluated in Next Phase)
* Selection of messaging/IPC mechanism between Core and UI/Tools.
* Selection of local storage engines for vector embeddings and relational history.
* Selection of local STT and TTS runtime engines.
* Selection of process isolation mechanics.
* Selection of manifest authenticity and policy-artifact authenticity mechanisms.
* Selection of key-protection mechanism for encrypted long-term memory.
* Selection of companion-channel transport and replay-window mechanics.
* Selection of execution-ownership / fencing mechanism realizing `REQ-CORE-005`.
* Selection of audit-log anchor mechanism realizing `REQ-DEP-002b`.

### 6.2 Implementation Details (Deferred)
* Specific code implementations, class hierarchies, or module structures.
* Exact JSON schema syntax for tool definitions and policy artifacts.
* GUI framework widget selections.

### 6.3 Future Ideas (Post-Baseline Enhancements)
* Multi-agent collaborative network.
* Smart-home local IoT control integration.
* Self-healing code execution modules.

---

## 7. Definition of Completion (DoC) for Requirements Phase

The requirements baseline phase shall be formally deemed complete when:
1. All functional and non-functional requirements have unique, stable IDs and unambiguous statements without implementation lock-in.
2. All security principles (explicit privilege deprivation, non-bypassable core, manifest-derived classification, risk-classification precedence, execution ownership, uncertainty handling, untrusted-content containment, transitive provenance) are captured as verifiable requirements.
3. Every MUST/SHALL requirement has a defined verification method.
4. No requirement statement prescribes a specific implementation technology.
5. The architectural team accepts this document as the formal binding input for the Architecture & System Design specification phase.
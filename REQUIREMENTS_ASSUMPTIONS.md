# Requirements Assumptions and Architectural Questions

* **Document Version:** 1.3.0
* **Status:** Binding Baseline (LOCKED)
* **Revision Basis:** Aligned with JARVIS_MASTER_REQUIREMENTS.md v1.3.0; Assumption 3 resolved into `REQ-SEC-007a`; architecture questions expanded to cover execution ownership/fencing (`REQ-CORE-005`) and audit-log anchor protection (`REQ-DEP-002b`).

---

## 1. Baseline Assumptions

1. **Host Environment Authority:** The primary runtime platform is Windows 10/11 x64, with standard administrative access granted during initial setup, but executing under standard user privileges during daily operational monitoring. Elevation-requiring operations are confined to the explicitly configured elevated service mode defined by `REQ-WIN-002`.

2. **Network Availability:** The core execution model assumes a hybrid setup: basic voice activation, local command execution, and safety evaluation function offline, while complex multi-step reasoning and deep web retrieval may utilize online endpoints when permitted by privacy policy. A defined degraded mode (`REQ-AI-011`) governs all-external-providers-unavailable operation. No security, privacy, or confirmation requirement is relaxed in degraded mode.

3. **Human-in-the-Loop Availability:** For `HIGH_RISK_SIDE_EFFECT` and `CRITICAL_SYSTEM` actions, a human operator is assumed to be physically present or connected via an authenticated mobile companion to provide explicit authorization signals. When no human is available, pending confirmations expire per `REQ-SEC-007a` and the request is cancelled — the assumption does not silently convert to approval.

4. **Hardware Baseline:** The host hardware is assumed to possess a minimum of 4 CPU cores, 16GB RAM, and direct input access to local microphone and display hardware. Dedicated local GPU hardware is the recommended baseline for the primary STT latency target (`REQ-VOI-002`); a tiered latency target applies on GPU-less hosts. Dedicated GPU hardware is not mandatory for baseline execution.

5. **Untrusted-Content Availability:** The system assumes that any content it retrieves, observes, or receives from outside its own authenticated policy and manifest infrastructure may be adversarial. This assumption applies uniformly to web pages, external APIs, OCR-derived text, plugin output, external model responses, and memory records derived from any of these. There is no class of external content that is presumed benign.

6. **Execution Ownership Assumption:** The system assumes that tool sub-processes may not terminate cooperatively, may survive Core restart, and may attempt to produce side effects after ownership revocation. The requirements treat this as a normal failure mode, not an exceptional one.

7. **Audit Log Trust Assumption:** The system assumes that the process writing the audit log is not itself trusted to be the sole verifier of log integrity. The chain anchor must therefore be protected independently from the log writer.

---

## 2. Open Questions to Resolve During Architecture Phase

### Architectural Question 1: Core IPC Mechanics
* **Question:** What inter-process communication framework provides the minimal attack surface and lowest latency between the core processing daemon, the isolated tool sub-processes, and the UI layer, while satisfying `REQ-CORE-001`, `REQ-CORE-004`, and `REQ-EXT-002`?
* **Impacts:** `REQ-CORE-001`, `REQ-CORE-004`, `REQ-EXT-002`

### Architectural Question 2: Desktop Accessibility vs. Direct Visual Manipulation
* **Question:** Should the Windows automation engine prioritize structured, semantically-addressed application automation surfaces, or rely on visual coordinate + OCR as the default fallback, given `REQ-WIN-001`'s structured-surface-first mandate?
* **Impacts:** `REQ-WIN-001`, `REQ-VIS-001`, `REQ-VIS-003`

### Architectural Question 3: Local Storage Encryption Strategy
* **Question:** How should cryptographic keys for persistent vector memory and local secret stores be managed across user logouts and reboots without requiring manual passphrase re-entry on every background service initialization, while satisfying `REQ-MEM-002`, `REQ-MEM-006`, and `REQ-WIN-004`?
* **Impacts:** `REQ-MEM-002`, `REQ-MEM-006`, `REQ-WIN-004`, `REQ-SEC-003`

### Architectural Question 4: State Machine Recovery Protocol
* **Question:** What transactional logging and recovery mechanism should be used to guarantee atomic recovery of incomplete task plans following ungraceful shutdowns, satisfying `REQ-REL-003`, `REQ-REL-003a`, `REQ-REL-003b`, and `REQ-REL-006`?
* **Impacts:** `REQ-CORE-002`, `REQ-REL-003`, `REQ-REL-003a`, `REQ-REL-003b`, `REQ-REL-006`

### Architectural Question 5: Manifest and Policy Authenticity Mechanism
* **Question:** What mechanism should be used to authenticate tool manifests and the policy artifact, satisfying `REQ-EXT-003`, `REQ-SEC-001a`, and `REQ-DEP-003`, without prescribing a specific cryptographic library?
* **Impacts:** `REQ-EXT-003`, `REQ-EXT-004`, `REQ-SEC-001a`, `REQ-DEP-003`

### Architectural Question 6: Untrusted-Content Provenance Mechanism
* **Question:** How should provenance/taint metadata be attached, propagated transitively through summaries, memory, and intermediate transformations, and consumed by the confirmation gate, satisfying `REQ-SEC-005`, `REQ-BRW-004`, `REQ-VIS-003`, and `REQ-MEM-007`? The propagation table for derived metadata (lengths, type tags, structural positions) shall be refined during architecture without weakening the containment property.
* **Impacts:** `REQ-SEC-005`, `REQ-BRW-004`, `REQ-VIS-003`, `REQ-MEM-007`, `REQ-AI-008`

### Architectural Question 7: Confirmation Integrity Binding
* **Question:** How should the confirmation payload be bound to the authorized operation (request ID, tool identity, parameters, resource keys, secret fingerprints) such that it satisfies `REQ-CORE-004`, `REQ-SEC-007`, and `REQ-SEC-008` without prescribing a UI framework?
* **Impacts:** `REQ-CORE-004`, `REQ-SEC-007`, `REQ-SEC-008`

### Architectural Question 8: Companion Channel Trust Mechanism
* **Question:** What out-of-band pairing, session establishment, replay protection, revocation, and host-side-confirmation mechanism satisfies `REQ-AND-002` through `REQ-AND-009`, conditional on `REQ-AND-001` being implemented?
* **Impacts:** `REQ-AND-002`–`REQ-AND-009`

### Architectural Question 9: Execution Ownership / Fencing Mechanism
* **Question:** What mechanism should be used to realize the execution-ownership property defined in `REQ-CORE-005` — e.g., capability fencing, revocation leases, brokered execution, resource-keyed locks with crash-safe ownership transfer, or another approach — such that stale workers cannot produce side effects after ownership revocation, and such that the `Timed_Out` release rule (for declared-idempotent, safely-abortable tools) can be distinguished from `Ambiguous_Uncertain` holds?
* **Impacts:** `REQ-CORE-005`, `REQ-CORE-003`, `REQ-REL-001`, `REQ-SIM-002`

### Architectural Question 10: Audit Log Anchor Protection
* **Question:** What mechanism should be used to protect the audit-log chain anchor independently from the log itself (per `REQ-DEP-002`), such that a writer with log-write access cannot unilaterally redefine the anchor, and such that truncation, rollback, and anchor replacement are detectable? The mechanism must also satisfy `REQ-DEP-002b`'s restart-time verification requirement.
* **Impacts:** `REQ-DEP-002`, `REQ-DEP-002a`, `REQ-DEP-002b`

### Architectural Question 11: Deduplication-Key Derivation for Non-Idempotent Tools
* **Question:** How should deduplication keys (`REQ-EXT-007`) be derived for non-idempotent tools such that the derivation is stable across replans, crash recovery, and user-initiated retries, while remaining sensitive to semantic parameter changes? The derivation must be robust to the replan-mapping approach chosen under `REQ-REL-006`.
* **Impacts:** `REQ-EXT-005`, `REQ-EXT-007`, `REQ-REL-006`, `REQ-REL-003b`

### Architectural Question 12: Uncertainty Taxonomy Instantiation
* **Question:** What concrete uncertainty classes and per-class recovery rules should populate the taxonomy required by `REQ-SIM-002a`, and how should tool manifests declare which classes apply?
* **Impacts:** `REQ-SIM-002a`, `REQ-SIM-002b`, `REQ-SIM-002`, `REQ-REL-005`

### Architectural Question 13: Simulation Fidelity Fidelity Model
* **Question:** How should `FULLY_SIMULATABLE`, `PARTIALLY_SIMULATABLE`, and `NOT_SIMULATABLE` be operationally defined so that the distinction is testable, and so that `REQ-SIM-001b`'s `SIMULATION_INCOMPLETE` acknowledgment can identify specific nodes whose simulation was incomplete?
* **Impacts:** `REQ-SIM-001`, `REQ-SIM-001a`, `REQ-SIM-001b`

### Architectural Question 14: User-Session Boundary Enforcement
* **Question:** How should the background-service boundary defined in `REQ-WIN-004` and `REQ-MEM-006` be enforced across Windows session events (logon, logoff, fast user switch, lock, unlock) such that credential access and key material are cleared on logout while queued credential-dependent requests are preserved for the next authenticated session?
* **Impacts:** `REQ-WIN-004`, `REQ-MEM-006`, `REQ-MEM-001`

### Architectural Question 15: Browser External-Side-Effect Classification
* **Question:** How should `REQ-BRW-005`'s classification by observed or declared external effect be operationalized, given that most endpoints do not expose a machine-readable read-only declaration? The conservative fallback (POST-based requests classified as side effects absent a declaration) must be reconciled with practical read-only APIs.
* **Impacts:** `REQ-BRW-005`, `REQ-SEC-001`, `REQ-SEC-005`

---

## 3. Assumptions Not Made

The following are explicitly **not** assumed by the requirements baseline. Each represents a decision deliberately deferred to architecture, or a property deliberately not relied upon:

1. **No assumption of cooperative tool termination.** The requirements (`REQ-CORE-005`, `REQ-REL-001`) treat non-cooperative termination as a normal failure mode.
2. **No assumption of a trusted log writer.** The requirements (`REQ-DEP-002b`) treat the log writer as potentially compromised and require the anchor to be protected independently.
3. **No assumption of a trusted UI process.** The requirements (`REQ-CORE-004`) treat the UI as less trusted than the core.
4. **No assumption of a trusted Android companion post-pairing.** The requirements (`REQ-AND-009`) require host-side confirmation for high-risk companion-originated commands.
5. **No assumption of perfect redaction.** The requirements (`REQ-VIS-002`) state that redaction is a reduction layer, not a guarantee.
6. **No assumption of perfect simulation fidelity.** The requirements (`REQ-SIM-001a`, `REQ-SIM-001b`) require explicit declaration and communication of simulation limitations.
7. **No assumption of perfect audit-log immutability.** The requirements and Non-Goals (§5.7) clarify that the property is tamper *evidence*, not tamper *prevention*.
8. **No assumption that human availability is guaranteed.** `REQ-SEC-007a` defines the behavior when no human is available to confirm.
9. **No assumption that external content is ever benign.** All external content is treated as potentially adversarial per `REQ-SEC-005`.
10. **No assumption that a provider's fallback characteristics match its primary.** `REQ-AI-006` prohibits silent substitution that weakens privacy, authorization, or risk classification.

---

## 4. Assumption ↔ Requirement Traceability

| Assumption | Related Requirements |
| :--- | :--- |
| §1.1 Host Environment Authority | `REQ-WIN-002`, `REQ-WIN-004` |
| §1.2 Network Availability | `REQ-AI-011`, `REQ-AI-006`, `REQ-AI-012` |
| §1.3 Human-in-the-Loop Availability | `REQ-SEC-007`, `REQ-SEC-007a`, `REQ-AND-009` |
| §1.4 Hardware Baseline | `REQ-VOI-002` |
| §1.5 Untrusted-Content Availability | `REQ-SEC-005`, `REQ-BRW-004`, `REQ-VIS-003`, `REQ-MEM-007`, `REQ-EXT-006` |
| §1.6 Execution Ownership Assumption | `REQ-CORE-005`, `REQ-REL-001`, `REQ-SIM-002` |
| §1.7 Audit Log Trust Assumption | `REQ-DEP-002`, `REQ-DEP-002b` |
| §3.1 No cooperative termination | `REQ-CORE-005`, `REQ-REL-001`, `REQ-REL-005` |
| §3.2 No trusted log writer | `REQ-DEP-002`, `REQ-DEP-002b` |
| §3.3 No trusted UI process | `REQ-CORE-004`, `REQ-SEC-007` |
| §3.4 No trusted companion post-pairing | `REQ-AND-009` |
| §3.5 No perfect redaction | `REQ-VIS-002` |
| §3.6 No perfect simulation | `REQ-SIM-001a`, `REQ-SIM-001b` |
| §3.7 No perfect audit immutability | `REQ-DEP-002`, Non-Goals §5.7 |
| §3.8 No guaranteed human availability | `REQ-SEC-007a` |
| §3.9 No benign external content | `REQ-SEC-005` |
| §3.10 No assumption of provider-fallback equivalence | `REQ-AI-006`, `REQ-AI-012` |

---

## 5. Resolution Log

* **Assumption 3 (Human-in-the-Loop Availability)** was previously unbounded — it did not specify behavior when no human was available. It is now bounded by `REQ-SEC-007a` (confirmation expiry transitions to `Cancelled`, not `Approved`).
* **Architectural Question 3** previously contained an implementation-specific parenthetical (`Windows DPAPI / User Credentials`) which was removed to avoid biasing architecture. The question is now stated as a capability-level problem.
* **Assumption 4 (Hardware Baseline)** was previously stated as a single latency target; it now aligns with `REQ-VOI-002`'s tiered latency target.
* **Architectural Questions 9–15** were added during the refinement loop to capture decisions newly deferred by added requirements (`REQ-CORE-005`, `REQ-DEP-002b`, `REQ-EXT-007`, `REQ-SIM-002a`, `REQ-SIM-001a`, `REQ-WIN-004`, `REQ-BRW-005`).
* **Section 3 (Assumptions Not Made)** and **Section 4 (Assumption ↔ Requirement Traceability)** were added to make explicit which properties the requirements deliberately do not rely on, and to provide traceability from assumptions to requirements.
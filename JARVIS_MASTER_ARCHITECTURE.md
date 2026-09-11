# JARVIS Master Architecture Baseline

* **Document Version:** 1.2.0
* **Status:** LOCKED — Binding Architecture Baseline
* **Target Platform:** Windows (Primary), Android Companion (Secondary)
* **Document Role:** Single Source of Truth for System Architecture; subordinate only to `JARVIS_MASTER_REQUIREMENTS.md` v1.3.0 and `REQUIREMENTS_ASSUMPTIONS.md` v1.3.0
* **Revision Basis:** Autonomous design + 3-iteration adversarial audit + refinement loop, closed at v1.2.0

---

## 1. Architectural Overview

JARVIS is a **Core-centric hub-and-spoke** system. The Core is the sole component with authority to invoke tools, grant execution ownership, make authorization decisions, or emit side-effecting operations. Every other component communicates with Core through authenticated channels; no component bypasses Core. The architecture is subordinate to the locked requirements baseline; where a requirement and an architectural convenience conflict, the requirement wins.

### 1.1 Architecture Diagram

```
                           ┌───────────────────────────────┐
                           │         jarvis-core           │
                           │  ─────────────────────────    │
                           │  Lifecycle State Machine      │
                           │  Policy Evaluator             │
                           │  Risk Classifier (no AI)      │
                           │  Ownership Grantor            │
                           │  Resource Lock Manager        │
                           │  Confirmation Gate            │
                           │  Replanning Supervisor        │
                           │  Dedup Index                  │
                           │  Audit Emitter                │
                           └───────────┬───────────────────┘
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        │                              │                              │
┌───────▼────────────┐      ┌──────────▼─────────┐      ┌─────────────▼─────────┐
│   jarvis-policy    │      │    jarvis-state    │      │     jarvis-audit      │
│  (authenticated)   │      │   (crash-safe WAL) │      │  (chained + anchor)   │
└────────────────────┘      └────────────────────┘      └───────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  Peer Processes (communicate only with Core over authenticated IPC)          │
│  ─────────────────────────────────────────────────────────────────────────  │
│  jarvis-planner           (UNTRUSTED AI)                                     │
│  jarvis-provider-gw       (mTLS to external providers)                       │
│  jarvis-executor          (worker supervisor; ownership token issuer)        │
│  jarvis-worker-<plugin>   (SANDBOXED; one per plugin invocation)             │
│  jarvis-browser           (isolated profile; untrusted-content ingress)      │
│  jarvis-windows-ctl       (structured-first Windows automation)              │
│  jarvis-voice             (wake + STT + TTS)                                 │
│  jarvis-vision            (capture + redaction + OCR extract)                │
│  jarvis-memory            (volatile + persistent)                            │
│  jarvis-provenance        (taint index + influence graph)                    │
│  jarvis-android-gw        (companion channel; untrusted until auth)          │
│  jarvis-ui                (display + gesture capture; LOWER TRUST than Core) │
└──────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────┐
│  jarvis-watchdog                       │
│  (supervise + restart + report)        │
└────────────────────────────────────────┘
```

### 1.2 Subordination Principle

```
REQUIREMENTS  (locked at v1.3.0)
     ↓
ARCHITECTURE  (this document, v1.2.0)
     ↓
IMPLEMENTATION SPECIFICATION  (next phase)
     ↓
CODE
     ↓
TESTING
```

Any change to this architecture that would weaken a locked requirement is prohibited. If a requirement is found to be unsatisfiable, a BLOCKING CHANGE REQUEST must be raised against the requirements document before any architectural change is made.

---

## 2. Component Specifications

Every component is specified with: responsibility, trust level, inputs, outputs, dependencies, allowed communication, forbidden communication, authority, data ownership, failure behavior, security boundary, recovery behavior.

### 2.1 jarvis-core

| Attribute | Value |
| :--- | :--- |
| Responsibility | Enforce lifecycle; evaluate policy; grant and revoke ownership; gate confirmation; supervise replanning; maintain dedup index; emit audit events. Sole authority for tool invocation. |
| Trust level | Highest. Compromise implies system compromise. |
| Inputs | Authenticated IPC from peers; user input via UI; state reads; policy reads; audit verifier results; provenance queries. |
| Outputs | Tool invocation requests (to executor); policy decisions; confirmation prompts; state writes; audit entries; ownership tokens. |
| Dependencies | `jarvis-policy` (read), `jarvis-state` (read/write), `jarvis-audit` (append), `jarvis-audit-verifier` (query), `jarvis-executor` (invoke), `jarvis-provenance` (query). |
| Allowed communication | All peers via authenticated IPC channels. |
| Forbidden communication | Direct filesystem writes outside state store; direct network; direct tool invocation without executor. |
| Authority | Full authorization authority; sole source of execution ownership grants and revocations. |
| Data ownership | Lifecycle state, ownership epochs, dedup index, confirmation digests, replan mappings. |
| Failure behavior | On crash: in-flight work is not auto-resumed; on restart, transitions to `Halted_Requires_Inspection`. |
| Security boundary | Authoritative for all cross-boundary decisions. |
| Recovery | Watchdog restarts Core; Core reads state store; verifies audit anchor via verifier; presents summary; requires user inspection. |

### 2.2 jarvis-policy

| Attribute | Value |
| :--- | :--- |
| Responsibility | Authenticated storage of the policy artifact, plugin manifests, and publisher trust store. |
| Trust level | Trusted after authentication of contents. |
| Inputs | Signed policy updates; signed manifest publications. |
| Outputs | Verified policy and manifest reads. |
| Dependencies | OS credential store (for publisher trust keys); Core (for own authentication). |
| Allowed communication | `jarvis-core` (read). |
| Forbidden communication | Direct peer writes; direct network. |
| Authority | None. Serves data only. |
| Data ownership | Policy artifact, manifest registry, publisher trust store. |
| Failure behavior | Unavailable → Core fails closed for any authorization decision. |
| Security boundary | Publisher signature verification gate. |
| Recovery | Re-verified against anchor on startup. |

### 2.3 jarvis-planner

| Attribute | Value |
| :--- | :--- |
| Responsibility | Produce candidate DAG from user intent and context. |
| Trust level | UNTRUSTED (AI process). |
| Inputs | User intent; context with provenance tags; tool manifest summaries. |
| Outputs | DAG candidate (schema-validated). |
| Dependencies | `jarvis-provider-gw`. |
| Allowed communication | `jarvis-core` (via authenticated channel). |
| Forbidden communication | Direct tool invocation; direct policy writes; direct state writes; direct network beyond provider-gw. |
| Authority | None. Cannot lower risk, grant authorization, or invoke tools. |
| Data ownership | None (ephemeral). |
| Failure behavior | Malformed output rejected; task fails closed. |
| Security boundary | Output validated by Core; risk re-derived by Core from manifest + parameters. |
| Recovery | Stateless. |

### 2.4 jarvis-provider-gw

| Attribute | Value |
| :--- | :--- |
| Responsibility | Route model calls; enforce privacy hard-filter; verify endpoint identity; handle provider failures. |
| Trust level | Trusted for routing only; does not decide authorization or risk. |
| Inputs | Model requests with declared constraints; policy-permitted endpoint list. |
| Outputs | Model responses (schema-validated downstream by Core). |
| Dependencies | `jarvis-policy` (endpoint list); OS credential store (client credentials). |
| Allowed communication | External providers via mTLS + pinned certs; `jarvis-planner`, `jarvis-vision`, `jarvis-core`. |
| Forbidden communication | Cannot invoke tools; cannot write policy; cannot mutate risk state. |
| Authority | Routing only; privacy hard-filter enforcement. |
| Data ownership | Endpoint identities; per-endpoint constraint metadata. |
| Failure behavior | Fail closed on unavailability, timeout, rate-limit, quota exhaustion, malformed output, schema-invalid output, contradictory output. No silent substitution. |
| Security boundary | Endpoint identity verification; no cross-privacy-class fallback. |
| Recovery | Stateless. |

### 2.5 jarvis-executor

| Attribute | Value |
| :--- | :--- |
| Responsibility | Spawn worker hosts; issue ownership tokens; supervise lifecycle; enforce cancellation; acknowledge flush before reporting worker exit. |
| Trust level | Trusted (subordinate to Core). |
| Inputs | Invocation requests from Core; ownership grants; cancellation signals. |
| Outputs | Worker process handles; outcome reports (structured); flush acknowledgements. |
| Dependencies | `jarvis-core`; OS process primitives; worker hosts. |
| Allowed communication | `jarvis-core`; worker hosts. |
| Forbidden communication | Direct network beyond worker-declared endpoints; direct policy reads; direct audit writes. |
| Authority | Lifecycle management of workers only. |
| Data ownership | Worker registry; ownership epochs (mirror of Core). |
| Failure behavior | Worker crash → reported as `Ambiguous_Uncertain` unless manifest declares idempotent-and-abortable. |
| Security boundary | Privilege token boundary; ownership token issuance. |
| Recovery | Reconciles with Core on restart; reports surviving workers. |

### 2.6 jarvis-worker-<plugin>

| Attribute | Value |
| :--- | :--- |
| Responsibility | Execute a single tool invocation inside an OS-level sandbox. |
| Trust level | UNTRUSTED (sandboxed). |
| Inputs | Invocation + ownership token from executor. |
| Outputs | Structured outcome (success / failure / ambiguity class with class identifier). |
| Dependencies | Declared network endpoints; declared workspace paths; declared resource keys. |
| Allowed communication | `jarvis-executor` only. |
| Forbidden communication | Other workers; network beyond declared endpoints; filesystem beyond declared workspace; Core directly; policy; state; audit. |
| Authority | Only what its declared manifest authorizes. |
| Data ownership | Ephemeral; writes only to declared resource keys. |
| Failure behavior | Crash → executor classifies as ambiguity unless declared idempotent-abortable. |
| Security boundary | OS-level process isolation with restricted token (strict subset of invoking user). |
| Recovery | Re-spawned on demand. |

### 2.7 jarvis-state

| Attribute | Value |
| :--- | :--- |
| Responsibility | Crash-safe persistence of lifecycle state, ownership epochs, dedup index, replan mappings. |
| Trust level | Trusted (data integrity critical). |
| Inputs | Writes from Core only. |
| Outputs | Reads to Core only. |
| Dependencies | OS filesystem. |
| Allowed communication | `jarvis-core`. |
| Forbidden communication | All peers. |
| Authority | None (storage). |
| Data ownership | All state records. |
| Failure behavior | Unavailable → Core fails closed. |
| Security boundary | Access control; encryption of sensitive fields. |
| Recovery | Integrity verification on startup; corruption blocks startup. |

### 2.8 jarvis-audit (writer)

| Attribute | Value |
| :--- | :--- |
| Responsibility | Append-only chained log. Holds no anchor key material. |
| Trust level | Trusted for appends. |
| Inputs | Audit entries from Core only. |
| Outputs | Append confirmations. |
| Dependencies | OS filesystem. |
| Allowed communication | `jarvis-core`. |
| Forbidden communication | All peers. |
| Authority | Append-only. |
| Data ownership | Log chain (excluding anchor state). |
| Failure behavior | Append failure → Core fails closed on operations that require audit. |
| Security boundary | No anchor-key access; log-writer compromise cannot sign rollbacks. |
| Recovery | Chain integrity verified by `jarvis-audit-verifier` on restart. |

### 2.9 jarvis-audit-verifier

| Attribute | Value |
| :--- | :--- |
| Responsibility | Hold anchor key; verify chain against anchor; countersign anchors periodically; detect truncation, rollback, replacement. |
| Trust level | Trusted for integrity verification. Distinct process from writer; no log-write capability. |
| Inputs | Chain reads; anchor state; external monotonic source or user-witnessed challenge. |
| Outputs | Verification verdicts to Core. |
| Dependencies | OS credential store (anchor key); OS monotonic source or hardware counter. |
| Allowed communication | `jarvis-core`. |
| Forbidden communication | `jarvis-audit` (writer); all peers. |
| Authority | Verification only. Cannot write log. |
| Data ownership | Anchor key; anchor history. |
| Failure behavior | Verification failure → Core fails closed on log-dependent authorization. |
| Security boundary | Anchor key never accessible to log writer process. |
| Recovery | Re-verifies on startup; reports to Core. |

### 2.10 jarvis-provenance

| Attribute | Value |
| :--- | :--- |
| Responsibility | Attach, propagate, and query taint tags per the influence-graph model required by `REQ-SEC-005`. |
| Trust level | Trusted for tagging; produces advisory metadata only. |
| Inputs | Tagging requests from Core; propagation queries from Core. |
| Outputs | Tag metadata; influence queries. |
| Dependencies | `jarvis-state` (persistence of tags on persisted values). |
| Allowed communication | `jarvis-core`. |
| Forbidden communication | All peers. |
| Authority | Tag propagation only; cannot influence authorization. |
| Data ownership | Tag index; influence graph. |
| Failure behavior | Unavailable → Core treats all values as tainted (conservative). |
| Security boundary | Tag integrity. |
| Recovery | Rebuildable from state store. |

### 2.11 jarvis-browser

| Attribute | Value |
| :--- | :--- |
| Responsibility | Controlled web retrieval and automation in isolated profile. |
| Trust level | UNTRUSTED (all content is untrusted). |
| Inputs | Retrieval requests from Core. |
| Outputs | Retrieved content with provenance tags; extraction results. |
| Dependencies | Isolated browser session (profile-isolated from user's primary). |
| Allowed communication | `jarvis-core`. |
| Forbidden communication | User's primary browser profile; saved credentials; session cookies; OS credential store; direct tool invocation. |
| Authority | Retrieval and automation per declared operation. |
| Data ownership | Isolated session state only. |
| Failure behavior | Blocked destination → logged; retrieval failure → returned as `Failed`. |
| Security boundary | Profile isolation; script disabled; download boundary. |
| Recovery | Stateless per session. |

### 2.12 jarvis-windows-ctl

| Attribute | Value |
| :--- | :--- |
| Responsibility | Structured-first Windows automation with coordinate fallback; workspace containment enforcement; integrity-level enforcement. |
| Trust level | Trusted for its declared scope only. |
| Inputs | Automation requests from Core. |
| Outputs | Structured outcomes. |
| Dependencies | OS accessibility/automation surfaces; workspace policy. |
| Allowed communication | `jarvis-core`. |
| Forbidden communication | Elevated windows; other-user processes; workspaces outside declaration; direct network. |
| Authority | Scoped to declared operation. |
| Data ownership | None beyond operation. |
| Failure behavior | Any cross-integrity attempt → refused; logged. |
| Security boundary | Integrity-level check; workspace canonicalization. |
| Recovery | Stateless. |

### 2.13 jarvis-voice

| Attribute | Value |
| :--- | :--- |
| Responsibility | Wake-word monitoring (local only); STT (local/hybrid per privacy policy); TTS with barge-in. |
| Trust level | Trusted for its scope; local-only pre-wake. |
| Inputs | Microphone; barge-in signals. |
| Outputs | Transcribed text (post-wake only); TTS audio; barge-in events. |
| Dependencies | Local STT/TTS engines. |
| Allowed communication | `jarvis-core`. |
| Forbidden communication | Pre-wake remote transmission; direct tool invocation. |
| Authority | Capture, transcribe, speak. |
| Data ownership | Ephemeral audio buffers (retention per policy). |
| Failure behavior | Capture-state indicator must reflect actual state; failure to display → capture disabled. |
| Security boundary | Pre-wake audio never leaves host. |
| Recovery | Stateless. |

### 2.14 jarvis-vision

| Attribute | Value |
| :--- | :--- |
| Responsibility | Consent-bounded screen capture; local redaction; structured extraction with versioned schema. |
| Trust level | Trusted for capture; all extracted text is untrusted. |
| Inputs | Capture requests (user-initiated or plan-node scope). |
| Outputs | Structured spatial representation (versioned schema) with provenance tags on all text. |
| Dependencies | Display capture; local OCR. |
| Allowed communication | `jarvis-core`. |
| Forbidden communication | Direct external transmission without redaction pass; direct tool invocation. |
| Authority | Capture and extraction within declared scope. |
| Data ownership | Ephemeral captures (retention per policy). |
| Failure behavior | Redaction unavailable → external transmission blocked. |
| Security boundary | Redaction pass before any external transmission; extracted text tagged untrusted. |
| Recovery | Stateless. |

### 2.15 jarvis-memory

| Attribute | Value |
| :--- | :--- |
| Responsibility | Volatile short-term store; encrypted long-term store; per-user isolation; provenance on every record; retention enforcement. |
| Trust level | Trusted for storage; contents are data, never authorization. |
| Inputs | Reads/writes from Core. |
| Outputs | Retrieved records with provenance and partition tags. |
| Dependencies | OS credential store (key); `jarvis-provenance` (tag persistence). |
| Allowed communication | `jarvis-core`. |
| Forbidden communication | Authorization subsystem (never queried for auth); other users' scopes. |
| Authority | Storage only. |
| Data ownership | Memory records scoped per-user. |
| Failure behavior | Unavailable → Core fails closed for memory-dependent operations. |
| Security boundary | Encryption at rest; OS-bound key; per-user scoping. |
| Recovery | Re-verified against key store on startup. |

### 2.16 jarvis-android-gw

| Attribute | Value |
| :--- | :--- |
| Responsibility | Terminate companion sessions; authenticate; enforce replay protection; forward commands to Core; enforce host-side confirmation for high-risk. |
| Trust level | UNTRUSTED until authenticated; commands remain untrusted until host-side confirmation for high-risk. |
| Inputs | Companion messages (paired sessions). |
| Outputs | Command requests to Core; status notifications to companion. |
| Dependencies | OS credential store (host pairing credentials); mTLS termination. |
| Allowed communication | `jarvis-core`; external companion over pinned mTLS. |
| Forbidden communication | Direct tool invocation; direct policy reads; direct state writes. |
| Authority | Session termination and authentication only. |
| Data ownership | Session state; pairing records. |
| Failure behavior | Session establishment fails closed; replay rejected; unauthenticated traffic dropped. |
| Security boundary | Pairing credential; host identity verification; nonce/sequence replay protection. |
| Recovery | Sessions do not survive host restart; re-authentication required. |

### 2.17 jarvis-ui

| Attribute | Value |
| :--- | :--- |
| Responsibility | Display state; collect user gestures; render Core-signed confirmation prompts; route user intents to Core. |
| Trust level | LOWER THAN CORE. |
| Inputs | State updates from Core (with Core-signed display strings where required); user input. |
| Outputs | Confirmation responses (signed with UI session key); user intents. |
| Dependencies | Core only. |
| Allowed communication | `jarvis-core`. |
| Forbidden communication | Direct tool invocation; direct state writes; direct policy reads; other peers. |
| Authority | None. Cannot authorize on its own. |
| Data ownership | None. |
| Failure behavior | UI crash does not halt execution; pending confirmations expire per `REQ-SEC-007a`. |
| Security boundary | Confirmation binding enforced by Core, not UI. High-risk confirmations may require an alternative trusted channel. |
| Recovery | Stateless; re-subscribes to Core on restart. |

### 2.18 jarvis-watchdog

| Attribute | Value |
| :--- | :--- |
| Responsibility | Supervise Core and critical peers; restart on crash; report anomalies. |
| Trust level | Trusted for supervision only. |
| Inputs | Process health signals. |
| Outputs | Restart commands; health reports. |
| Dependencies | OS process primitives. |
| Allowed communication | Core (start/stop); OS. |
| Forbidden communication | Direct peer invocation other than start/stop; authorization decisions. |
| Authority | Process supervision only. |
| Data ownership | None beyond supervision state. |
| Failure behavior | Watchdog crash does not halt Core; Core continues but without supervision redundancy. |
| Security boundary | No authorization authority. |
| Recovery | Self-restarting via OS service control. |

---

## 3. Trust Boundaries

| Boundary | Crosses | Authentication | Authority transferred | Authority NOT transferred | Compromise containment |
| :--- | :--- | :--- | :--- | :--- | :--- |
| UI ↔ Core | User intent; confirmation gestures | Session key negotiated out-of-band; UI signature verified by Core | User intent expression | Authorization; tool invocation | Compromised UI cannot forge confirmation (digest bound); cannot invoke tools |
| Core ↔ Worker | Invocation; ownership token | Process-local IPC + HMAC token | Scoped operation only | Global authority | Compromise contains to declared resource keys |
| Core ↔ Policy | Reads | Publisher signature + anchor | Data | Authorization | Cannot write policy |
| Core ↔ Audit | Appends | Process-local | Append | Rewrite/truncate | Cannot rewrite history; anchor verifier detects |
| Core ↔ Provider | Model calls | mTLS + endpoint pinning | Query | Authorization | Cannot weaken privacy; fallback prohibited |
| Android ↔ Core | Commands | Paired credential + nonce/sequence + mTLS | Authenticated commands | Host-side authorization | High-risk commands require host-side confirmation |
| Plugin ↔ OS | Sandboxed ops | OS isolation + restricted token | Only declared capabilities | Full user token | Out-of-bounds access blocked by OS |
| Browser ↔ External | Retrieval; outbound requests | mTLS where supported; endpoint identity per policy | Retrieval; classified outbound | Credential inheritance | Isolated profile; no primary credentials |
| Vision ↔ External model | Frames | Endpoint identity; redaction pass first | Frame contents (post-redaction) | Credential access | Redaction + provenance tag |
| Memory ↔ Core | Reads/writes | Process-local | Data | Authorization | Memory never queried for auth |

---

## 4. Security-Critical Architecture

The following invariants are enforced structurally. Each is verified by an architectural mechanism, not merely asserted.

| Invariant | Enforced By |
| :--- | :--- |
| AI cannot authorize itself | Risk and authorization derive from manifest + canonical parameters only (`REQ-AI-008`); planner has no write access to policy or risk state |
| AI cannot change risk classification | Risk classifier runs in Core with no access to AI output or untrusted content |
| AI cannot directly execute tools | Tool invocation is exclusively via `jarvis-executor`; planner has no IPC path to executor |
| External content cannot become authority | Provenance tagging (`REQ-SEC-005`) + influence-graph propagation; tainted values cannot modify policy/manifest/risk/auth |
| Memory cannot grant authority | Security Policy Evaluator does not query memory (`REQ-MEM-004`); memory records carry provenance and partition tags |
| UI cannot forge confirmation | Confirmation binding to request ID + operation digest (`REQ-CORE-004`); Core-signed display strings; high-risk may require alternative channel |
| Android cannot bypass host-side authorization | Companion commands enter the same serialized queue (`REQ-AND-004`); high-risk commands require host-side confirmation (`REQ-AND-009`) |
| Plugins cannot escalate privileges | Strict-subset token; declared resource keys; declared endpoints; plugin-to-plugin communication blocked (`REQ-EXT-002`, `REQ-EXT-006`) |
| Tool manifests cannot silently change runtime risk | Publisher authentication (`REQ-EXT-003`); runtime immutability (`REQ-EXT-004`) |
| Secrets cannot leak | Never written to plans/logs/prompts/UI (`REQ-SEC-003`); confirmation identifiers are non-invertible, non-partial (`REQ-SEC-008`) |
| High-risk operations cannot bypass confirmation | Confirmation gate enforced by Core; digest binding; freshness; single-use (`REQ-SEC-007`) |
| Critical operations cannot bypass secondary authentication | Unsigned-binary execution and elevation-requiring operations require mandatory secondary auth (`REQ-SEC-001`, `REQ-WIN-002`) |
| Revoked execution cannot continue causing side effects | Ownership epoch fencing (`REQ-CORE-005`); fsync+flush acknowledgement before terminal exit; ambiguity classification otherwise |
| Compromised component cannot route around Core | Every side-effecting path passes through Core; no alternate invocation API exists |

---

## 5. Execution Ownership + Fencing

### 5.1 Ownership Grant Lifecycle

```
Core decides to execute node N with resource keys K
   ↓
Core increments resource-key epoch for K (epoch_K)
   ↓
Core issues ownership token: T = HMAC(core_key, {request_id, node_id, K, epoch_K, expiry})
   ↓
Executor receives T, spawns worker, passes T to worker
   ↓
Worker must present T with every write to K
   ↓
Resource Lock Manager (in Core) accepts write only if:
     - T's HMAC valid
     - T's epoch_K matches current epoch_K
     - T not expired
     - T not revoked
```

### 5.2 Revocation

- Core increments `epoch_K` → all outstanding tokens for K become stale.
- A worker presenting a stale token at write time is rejected.
- External effects (network POST to remote service) cannot be revoked once dispatched. Per `REQ-REL-001`, if dispatch completed and outcome unknown, the node transitions to `Ambiguous_Uncertain`; ownership of K is held until the ambiguity is resolved.

### 5.3 Stale Worker Scenarios

| Scenario | Behavior |
| :--- | :--- |
| Timeout while worker alive | Epoch incremented; worker writes rejected; ambiguity recorded if termination unconfirmed |
| Cancellation while worker alive | Same |
| Worker ignores termination | Writes rejected by epoch; ownership held |
| Core crashes while worker alive | On restart, epoch state read; worker's token epoch ≠ current → writes rejected |
| Worker survives Core restart | Stale epoch → writes rejected |
| Delayed callback | Post-revocation epoch mismatch → rejected |
| `Timed_Out` (declared idempotent-abortable) | Treated as confirmed worker termination → ownership released |
| Delayed async write after worker exit | Prevented by flush+acknowledgement requirement; absent acknowledgement → ambiguity |

### 5.4 Explicit Limitation

External systems that cannot receive epoch checks are protected by **deduplication + ambiguity classification**, not by fencing. Once a network POST is dispatched and its outcome is unknown, JARVIS cannot unilaterally revoke it. This limitation is architecturally irreducible and is documented per `REQ-CORE-005`'s certainty requirement.

---

## 6. State + Recovery

### 6.1 State Schema (per task)

```
task {
  request_id, plan_id,
  nodes: [{
    node_id, execution_id, attempt_number, ownership_epoch,
    timeout_deadline, cancellation_epoch,
    authorization_digest, dedup_key,
    side_effect_status, verification_status, uncertainty_class,
    replan_mapping
  }],
  cumulative_budget,
  provenance_tags
}
```

### 6.2 Recovery Protocol on Core Restart

1. Watchdog restarts Core.
2. Core loads state store.
3. Core requests anchor verification from `jarvis-audit-verifier`; chain integrity verified.
4. All non-terminal tasks transition to `Halted_Requires_Inspection`.
5. Core cross-checks with `jarvis-executor`; any surviving worker is reported.
6. User summary presented; no auto-resume occurs.
7. Manual resumption is subject to deduplication check (`REQ-REL-003b`).
8. Cumulative side-effect budget reconstructed from audit log.

### 6.3 Failure-Mode Behavior Matrix

| Failure | Behavior |
| :--- | :--- |
| Core crash | Watchdog restarts; in-flight → `Halted_Requires_Inspection` |
| Worker crash | `Ambiguous_Uncertain` unless manifest declares idempotent-abortable |
| Worker survives Core | Stale epoch → writes rejected |
| Machine restart | Same as Core crash; anchor verified |
| DB failure | Core fails closed |
| Network failure | Provider fail-closed |
| Provider failure | No silent substitute |
| Timeout | Ambiguity classification unless declared idempotent-abortable |
| Cancellation race | Epoch + ambiguity handling |
| Partial side effect | `Ambiguous_Uncertain` + ownership hold |
| Missing tool result | `Ambiguous_Uncertain` |
| Corrupted state | Startup blocked; user notified |

---

## 7. Idempotency + Deduplication

### 7.1 Identity Model

| Identity | Scope | Stability |
| :--- | :--- | :--- |
| `task_id` | New per user request | Immutable per request |
| `plan_id` | One per planning iteration | Immutable per iteration |
| `node_id` | Stable within plan; preserved across replan via mapping | Preserved across replan |
| `execution_id` | New per attempt | Immutable per attempt |
| `attempt_number` | Monotonic per node | Monotonic |
| `ownership_epoch` | Per resource key | Monotonic per key |
| `dedup_key` | `H(semantic_operation_id ‖ canonical_semantic_params ‖ resource_key_set)` | Stable across attempts, replans, recovery, retries |
| `authorization_digest` | SHA-256 of canonical operation representation | Immutable per authorized operation |
| `semantic_operation_id` | Declared by manifest; stable across compatible versions | Publisher-maintained |

### 7.2 Dedup Decision Table

| Prior outcome | New attempt | Decision |
| :--- | :--- | :--- |
| None | Any | Allow |
| `Failed` (definitively, side effect unapplied) | Retry/replan | Allow |
| `Completed` | Retry | Refuse unless user explicitly overrides with new `task_id` |
| `Ambiguous_Uncertain` | Any | Refuse; require user resolution |
| `Halted_Requires_Inspection` | Any | Refuse; require user resolution |
| `Timed_Out` (idempotent-abortable) | Retry | Allow with same `dedup_key` |

---

## 8. Authorization + Confirmation

### 8.1 Canonical Operation Representation

```
C = canonical_json({
  semantic_operation_id,
  params: sorted_key_pairs
          (secret refs replaced with secret_fingerprint),
  resource_keys: sorted,
  intent_hash,
  manifest_hash
})
```

### 8.2 Digest and Confirmation Flow

1. Core computes `operation_digest = SHA-256(C)`.
2. Core sends confirmation prompt to UI via authenticated channel; prompt contains Core-signed display strings.
3. UI displays; collects user gesture.
4. UI returns `{ request_id, operation_digest, signature }` where signature is computed with the UI session key.
5. Core verifies signature; matches digest; checks freshness (single-use, ≤ configurable TTL).
6. On match, execution proceeds bound to that digest.
7. Any parameter change → new digest → new confirmation required.

### 8.3 Confirmation Channels

Three channels, ordered by trust:

1. **Mobile companion** (HIGHEST): displayed on paired Android device; requires companion authenticated; used for `CRITICAL_SYSTEM` where available.
2. **OS-native secure prompt** (HIGH): trusted OS surface; used when available.
3. **jarvis-ui** (MEDIUM): accepted for `LOW_RISK_SIDE_EFFECT` and `HIGH_RISK_SIDE_EFFECT` where the user has not enabled a higher channel; for `CRITICAL_SYSTEM`, an additional out-of-band visual check is required (user types a Core-provided code displayed in the confirmation prompt).

### 8.4 Secret Handling

Per `REQ-SEC-008`:
- Secret-bearing parameters are displayed by credential human-readable name or by a non-invertible fingerprint.
- No last-N characters or partial content.
- Digest incorporates the secret's contribution via a keyed hash computed inside the Core's secret resolver; the secret itself never enters the canonical representation in cleartext.

### 8.5 Replanning Authorization

Per `REQ-AI-004`:
- Delta plan re-submitted through policy evaluator.
- Evaluated against original intent, authorized goal, authorized resource scope, cumulative side-effect budget.
- Material change (any change to resource keys, risk tier, node count, tool identity, semantic parameter, or cumulative budget) requires fresh user authorization.

---

## 9. Risk Architecture

### 9.1 Pipeline

```
DAG candidate (from planner)
   ↓
Manifest lookup (authenticated, jarvis-policy)
   ↓
Risk derivation (manifest_class + canonical parameters + policy rules)
   ↓
Max-tier resolution (REQ-SEC-001c)
   ↓
Parameter-change re-evaluation at execution time (REQ-SEC-001b)
   ↓
Authorization requirement derivation
   ↓
Confirmation gate
```

### 9.2 Separation of Concerns

```
AI PLANNING  ≠  RISK CLASSIFICATION  ≠  AUTHORIZATION  ≠  EXECUTION
```

- **AI planning**: produces a DAG; no risk authority.
- **Risk classification**: runs in a dedicated Core module; sees only manifest + canonical parameters; sees no AI output or untrusted content.
- **Authorization**: determined by risk tier + policy; produces the required confirmation channel.
- **Execution**: binds to the authorization digest.

### 9.3 Classification Precedence

Per `REQ-SEC-001c`, the effective tier is the maximum of all applicable tiers. `REQ-WIN-005` enumerations are floors, not ceilings. The classification decision is logged with the applied criteria.

---

## 10. Simulation Architecture

### 10.1 Adapters

Each tool exposes:
- **Production adapter**: real I/O.
- **Simulation adapter**: mock I/O; declared simulation level.

### 10.2 Simulation Levels

| Level | Meaning |
| :--- | :--- |
| `FULLY_SIMULATABLE` | All side effects faithfully modelled without real I/O. |
| `PARTIALLY_SIMULATABLE` | Subset of effects modelled; remaining flagged as unverified. |
| `NOT_SIMULATABLE` | Adapter refuses; declares reason. |

### 10.3 Simulation Sandbox

Simulation runs in an OS-level sandbox with:
- No network capability.
- No filesystem write capability.
- No external handle capability.
- Mock I/O only.

### 10.4 Incomplete Simulation

Mixed plans → `SIMULATION_INCOMPLETE`. Acknowledgment identifies specific incomplete nodes and states that those nodes carry unverified risk. `SIMULATION_INCOMPLETE` plans are never represented as validated.

---

## 11. Browser + External World

- Isolated profile: no access to user's primary profile, saved credentials, session cookies, or OS credential store.
- All retrieved content tagged as untrusted (`REQ-SEC-005`).
- Script disabled in automation context.
- Downloads confined to workspace (`REQ-WIN-003`).
- Outbound requests classified by observed/declared effect, not HTTP method verb (`REQ-BRW-005`).
  - State-changing requests gated as write actions.
  - Declared read-only POST (including search APIs) treated as retrieval.
  - Absent machine-readable declaration, POST conservatively classified as side effect.

---

## 12. Windows Architecture

- **Structured-first**: prefer semantically-addressed surfaces; coordinate synthesis only as fallback with reason logged (`REQ-WIN-001`).
- **Workspace containment**: writes absolutely confined to workspace; reads outside workspace classified `HIGH_RISK_SIDE_EFFECT`; canonicalization of symlinks, junctions, short paths, case variations, `..` traversal before containment check (`REQ-WIN-003`).
- **Cross-integrity prohibition**: no interaction with higher-integrity windows, controls, or processes; no launch of unsigned or unverifiable binaries except in declared elevated service mode with mandatory secondary authentication (`REQ-WIN-002`).
- **Elevated service mode**: user-enabled at installation; scope declared in policy; activation and deactivation logged; state visible in UI.
- **Background session boundary**: outside authenticated user session, only `READ_ONLY` and non-credential-dependent tasks run; credential-dependent requests queue (`REQ-WIN-004`).
- **Persistence-relevant actions** classified at minimum `HIGH_RISK_SIDE_EFFECT` (`REQ-WIN-005`).

### 12.1 What JARVIS Explicitly CANNOT Do

- Cannot bypass OS elevation prompts.
- Cannot exploit local OS vulnerabilities.
- Cannot perform unprompted administrative actions.
- Cannot write outside declared workspaces.
- Cannot launch unverifiable binaries outside elevated service mode.
- Cannot interact with higher-integrity windows.

---

## 13. Plugin Architecture

### 13.1 Manifest

Every manifest declares: inputs, outputs, side effects, risk class, required scopes, resource keys, simulation capability, idempotency, dedup-key schema (`semantic_operation_id`), uncertainty classes, declared network endpoints, declared workspace paths, publisher identity.

### 13.2 Authenticity and Immutability

- Manifest signed by publisher; verified at load (`REQ-EXT-003`).
- Manifest immutable at runtime; modification triggers unload and security event (`REQ-EXT-004`).

### 13.3 Isolation

- Worker processes run with restricted tokens (strict subset of invoking user).
- No path access outside declared workspace.
- No network beyond declared endpoints.
- No signal or inspection of other workers.
- Plugin-to-plugin communication blocked by construction (`REQ-EXT-006`).
- Every isolation bound has an independently testable verification.

### 13.4 Provenance Preservation

Any data transferred between plugin contexts via Core carries provenance tags. Core rejects transfers that strip provenance (`REQ-EXT-006`, `REQ-SEC-005`).

---

## 14. Android Architecture

- **Pairing**: QR-based, out-of-band, ECDH key exchange. Credential stored in OS credential store with bounded lifetime. Pairing requires physical proximity or out-of-band channel independent of command network path (`REQ-AND-002`).
- **Host identity**: verified by companion on every session establishment (`REQ-AND-005`).
- **Session integrity**: mTLS with pinned host cert; per-command nonce and sequence; replay-window enforcement (`REQ-AND-003`).
- **Network exposure**: minimized; if host listens, binds only to authenticated, integrity-protected sessions (`REQ-AND-007`).
- **Revocation**: single user action; credential deleted; sessions torn down; reflected before next command processed (`REQ-AND-006`).
- **Uninstall/reinstall**: invalidates pairing.
- **Session expiry**: idle timeout; re-authentication required (`REQ-AND-008`).
- **Channel serialization**: companion commands enter same serialized queue as host commands (`REQ-AND-004`).
- **Companion compromise**: `HIGH_RISK_SIDE_EFFECT` and `CRITICAL_SYSTEM` commands require host-side confirmation independent of companion (`REQ-AND-009`).
- **Dual-surface UX**: Android displays "awaiting host confirmation"; host displays prompt; both use same operation digest. Remote high-risk refused if host unreachable.

---

## 15. Memory + Data Architecture

| Store | Scope | Encryption | Retention | Access |
| :--- | :--- | :--- | :--- | :--- |
| Short-term (volatile) | Per session | N/A (in-memory) | Cleared on exit/logout | Core only |
| Long-term (persistent) | Per user | Encrypted at rest; OS-bound key | Configurable per category | Core only; provenance on every record |
| Audio buffers | Per request | Ephemeral | Raw not persisted beyond active request unless opted-in | Voice module |
| Screen captures | Per request | Ephemeral | Same as audio | Vision module |
| Prompt logs | Per request | Ephemeral or redacted | Configurable | Core only |
| Audit log | Global (per user) | Chained; anchor verifier holds key | Per policy | Core append; verifier read |

**Memory invariants:**
- Never queried for authorization (`REQ-MEM-004`).
- Every record carries provenance (`REQ-MEM-007`).
- Partition tag (episodic/preference vs. policy) displayed in UI (`REQ-MEM-003`).
- Per-user isolation enforced (`REQ-MEM-006`).
- Automatic retention enforcement; no user action required (`REQ-MEM-005`).

---

## 16. Audit Log Architecture

### 16.1 Structure

Entry: `{seq, prev_hash, timestamp, event_type, payload, anchor_epoch}`.

`hash_i = SHA-256(entry_i)`.

Chain: each entry references `prev_hash`.

### 16.2 Anchor

- Periodic anchor snapshot `{seq, hash_i, anchor_epoch}` is signed by `jarvis-audit-verifier`.
- Anchor key held in OS credential store; never accessible to log writer.
- Anchor countersigned periodically by OS-native monotonic source or user-witnessed challenge.

### 16.3 Verification

- On restart: chain re-verified against anchor before new appends.
- Truncation, rollback, replacement, deletion, and partial writes detected by anchor mismatch.
- Timestamp manipulation resisted via monotonic source (`REQ-SEC-006`).

### 16.4 Trust Model

Anchor key custody is split: log writer cannot read it. External monotonic source or user-witnessed challenge provides additional anchoring against OS-level compromise. This is the residual-risk boundary of the OS trust assumption.

---

## 17. Event + IPC Architecture

| Aspect | Decision |
| :--- | :--- |
| Transport | Windows named pipes with per-channel HMAC keyed at process start via OS credential store |
| Message schema | Versioned JSON with explicit `schema_version` |
| Correlation IDs | `request_id`, `node_id`, `execution_id` |
| Ordering | FIFO per channel; global ordering via Core serialization |
| Delivery | At-least-once for control messages; deduplicated on `message_id` |
| Backpressure | Bounded queues; overflow fails closed |
| Timeouts | Applied to all IPC waits |
| Authentication | Per-process HMAC key; validated on every message |
| Authorization | Per-message role check; sender must be permitted to send that message type to that recipient |
| Replay protection | Monotonic message counter per channel; windowed acceptance |
| Versioning | Explicit schema version; incompatible version rejected |

Every cross-process communication path is mapped in `ARCH_IPC.md`.

---

## 18. AI Model Architecture

- **Model registry** in policy artifact: `{model_id, capabilities, privacy_class, endpoints}`.
- **Routing** enforces privacy as a hard filter.
- **Endpoint identity** verified via mTLS + pinned certificates.
- **Fallback** prohibited across privacy/privilege class boundaries; substitution requires same constraints as original.
- **Malformed / schema-invalid output**: rejected; task fails closed.
- **Contradictory output**: detected; user presented with both or replanning required.
- **Quota exhaustion**: fail closed with user-visible reason.
- **Degraded offline mode**: defined; does not relax security, privacy, or confirmation requirements.
- **Identity and version recording**: recorded per plan; unavailability noted; reproducibility not guaranteed.
- **No hard-coded model choices**: registry is policy-driven and replaceable.

---

## 19. Failure Model Summary

| Failure class | Behavior | Recovery |
| :--- | :--- | :--- |
| Core crash | In-flight → `Halted_Requires_Inspection` | Watchdog restart; user inspection; dedup check on manual resume |
| Worker crash | `Ambiguous_Uncertain` unless idempotent-abortable | Ownership held; user resolution |
| Worker survives Core | Stale epoch rejects writes | Ownership reconciled on restart |
| Machine restart | Same as Core crash | Anchor verified; state read |
| Database failure | Core fails closed | Integrity check on restart |
| Network failure | Provider fail-closed | No auto-retry on external effects |
| Provider failure | Fail closed; no silent substitute | User-visible reason |
| Timeout | `Ambiguous_Uncertain` unless idempotent-abortable | Ownership held; user resolution |
| Cancellation race | Epoch fencing + ambiguity | Ownership held; user resolution |
| Partial side effect | `Ambiguous_Uncertain` + ownership hold | User resolution; dedup prevents duplicate |
| Missing tool result | `Ambiguous_Uncertain` | Same |
| Corrupted state | Startup blocked | User notified; manual intervention |

---

## 20. Residual Risks

1. **External-system side effects are not fully fenced.** Once a network effect is dispatched and the outcome is unknown, JARVIS cannot unilaterally revoke it. Mitigation: ambiguity classification + dedup + no auto-retry on external effects.
2. **Simulation fidelity is never guaranteed.** Declared levels acknowledge this; no mitigation beyond explicit acknowledgment.
3. **UI trust is fundamentally lower than Core.** A compromised UI can mislead a user even with Core-signed display fields; mitigation is Core-signed display + alternative confirmation channels for high-risk operations.
4. **Anchor integrity depends on external trust.** If the anchor's key material is compromised at the OS level (privileged attacker), tamper-evidence degrades. Documented as the OS trust boundary.
5. **Taint propagation correctness depends on the influence-graph implementation.** Correctness is a Phase-implementation concern; the property is required by requirements.
6. **Non-cooperative remote systems.** If a plugin calls an external service whose idempotency cannot be verified, retries and replans after ambiguity are prohibited. Reduces resilience but preserves safety.
7. **Provider behavior drift.** Providers may change model behavior silently; identity is recorded but behavior changes are not detectable in advance.

---

## 21. Requirements Traceability

Every requirement maps to a component, mechanism, and verification method. Full matrix maintained in `ARCH_TEST_MATRIX.md`. Representative mappings:

| Requirement | Component | Mechanism | Verification |
| :--- | :--- | :--- | :--- |
| REQ-CORE-001 | jarvis-core | Single-invocation authority | Static analysis |
| REQ-CORE-002 | jarvis-core | Lifecycle state machine | State-transition test |
| REQ-CORE-003 | jarvis-core | Resource-key lock manager | Concurrency test |
| REQ-CORE-004 | jarvis-core + jarvis-ui | Digest binding + Core-signed display | Tamper test |
| REQ-CORE-005 | jarvis-executor + jarvis-core | Epoch token + resource-key lock | Fault-injection |
| REQ-AI-003 | jarvis-planner | DAG generation; schema validation | Schema test |
| REQ-AI-004 | jarvis-core | Replan evaluation against original intent | Adversarial delta test |
| REQ-AI-006 | jarvis-provider-gw | Fail-closed; no silent substitute | Fault-injection |
| REQ-AI-008 | jarvis-core (Risk Classifier) | Manifest-derived classification | Adversarial mislabel test |
| REQ-AI-009 | jarvis-core | Schema validation of model output | Schema-invalid test |
| REQ-VOI-001 | jarvis-voice | Pre/post-wake boundary | Network analysis |
| REQ-VIS-002 | jarvis-vision | Redaction pass before external transmission | Redaction test |
| REQ-MEM-004 | jarvis-core (Policy Evaluator) | Memory never queried for auth | Static analysis |
| REQ-MEM-007 | jarvis-memory + jarvis-provenance | Provenance on every record | Laundering test |
| REQ-BRW-004 | jarvis-browser | Untrusted tagging | Adversarial page test |
| REQ-BRW-005 | jarvis-browser | Classification by effect | Outbound-side-effect test |
| REQ-WIN-002 | jarvis-windows-ctl | Cross-integrity prohibition; elevated-service-mode definition | Cross-integrity test |
| REQ-WIN-003 | jarvis-windows-ctl | Canonicalization + absolute write containment | Traversal/symlink test |
| REQ-EXT-003 | jarvis-policy | Publisher authentication | Tampered-manifest test |
| REQ-EXT-007 | jarvis-core (Dedup Index) | Semantic dedup key | Cross-replan/recovery test |
| REQ-AND-009 | jarvis-android-gw + jarvis-core | Host-side confirmation for high-risk | Compromised-companion test |
| REQ-SEC-005 | jarvis-provenance | Influence-graph propagation | Adversarial laundering test |
| REQ-SEC-007 | jarvis-core (Confirmation Gate) | Digest binding + freshness | Tamper + replay test |
| REQ-SEC-008 | jarvis-core | Non-invertible secret identifiers | Secret-display test |
| REQ-SIM-001a | jarvis-executor | Sandboxed simulation adapters | Network-block test |
| REQ-SIM-002 | jarvis-core | Uncertainty taxonomy | Fault-injection |
| REQ-REL-001 | jarvis-executor | Observable cancellation contract | Worker-ignores-termination test |
| REQ-REL-003a | jarvis-core | Per-node reconciliation | Crash-test simulation |
| REQ-REL-003b | jarvis-core | Manual-resume dedup check | Crash-resume test |
| REQ-DEP-002 | jarvis-audit + jarvis-audit-verifier | Split custody + anchor | Rollback test |
| REQ-DEP-002b | jarvis-audit-verifier | Restart verification + anchor | Anchor-replacement test |

---

## 22. Change Ledger

| Change ID | Version | Affected Requirements | Problem | Change |
| :--- | :--- | :--- | :--- | :--- |
| C-001 | 1.0.0 → 1.1.0 | REQ-CORE-005 | Delayed async writes after worker exit | Worker exit requires fsync + deferred-write acknowledgment |
| C-002 | 1.0.0 → 1.1.0 | REQ-SEC-005 | Tool selection not tainted | Influence graph covers control-flow |
| C-003 | 1.0.0 → 1.1.0 | REQ-DEP-002b | Anchor key accessible to log writer | Split writer / verifier |
| C-004 | 1.0.0 → 1.1.0 | REQ-CORE-004 | UI-spoofing attack | Core-signed display fields |
| C-005 | 1.0.0 → 1.1.0 | REQ-EXT-007 | Schema drift changes dedup key | `semantic_operation_id` |
| C-006 | 1.0.0 → 1.1.0 | REQ-AND-009 | Dual-surface UX flow | Defined flow + fail-closed remote |
| C-007 | 1.0.0 → 1.1.0 | REQ-SEC-005 | Lossy transformation ambiguity | Influence graph definition |
| C-008 | 1.0.0 → 1.1.0 | REQ-SIM-001 | Simulation adapter network leak | OS-level sandbox |
| C-009 | 1.0.0 → 1.1.0 | REQ-EXT-002 | Provenance across plugin transit | Provenance preserved via Core |
| C-010 | 1.0.0 → 1.1.0 | REQ-SEC-006 | Clock-jump threshold undefined | Configurable threshold |
| C-011 | 1.1.0 → 1.2.0 | REQ-CORE-004 | Confirmation channel ambiguity | Three channels defined |
| C-012 | 1.1.0 → 1.2.0 | REQ-AI-009 | Contradictory model output | Contradiction detection |

---

## 23. Audit Closure Summary

| Finding | Severity | Status | Fixed In |
| :--- | :--- | :--- | :--- |
| A-001 | HIGH | Closed | v1.1.0 |
| A-002 | HIGH | Closed | v1.1.0 |
| A-003 | HIGH | Closed | v1.1.0 |
| A-004 | MEDIUM | Closed | v1.1.0 |
| A-005 | MEDIUM | Closed | v1.1.0 |
| A-006 | MEDIUM | Closed | v1.1.0 |
| A-007 | MEDIUM | Closed | v1.1.0 |
| A-008 | MEDIUM | Closed | v1.1.0 |
| A-009 | MEDIUM | Closed | v1.1.0 |
| A-010 | LOW | Closed | v1.1.0 |
| B-001 | MEDIUM | Closed | v1.2.0 |
| B-002 | LOW | Closed | v1.2.0 |

**Audit A result:** PASS (0 CRITICAL, 0 HIGH, 0 MEDIUM, 0 LOW).
**Audit B result:** PASS (0 CRITICAL, 0 HIGH, 0 MATERIAL MEDIUM, 0 UNRESOLVED CONTRADICTIONS).

---

## 24. Architecture Verdict

**🔒 ARCHITECTURE LOCK**

The architecture at v1.2.0 satisfies the objective LOCK criteria:

- **Requirements:** Every requirement has architectural coverage; no requirement is silently weakened; no contradictory requirement remains unresolved; no requirement imposes an impossible obligation.
- **Security:** 0 unresolved CRITICAL, HIGH, or material MEDIUM findings.
- **Execution:** Ownership, fencing, cancellation, timeout, recovery, stale workers, and ambiguous effects are all defined with concrete mechanisms.
- **Authorization:** Confirmation binding, replay prevention, risk re-evaluation, secret-safe confirmation, and replanning authorization are all defined.
- **Reliability:** Crash behavior, recovery identity, deduplication, idempotency, and uncertain-outcome behavior are all defined.
- **Security boundaries:** AI cannot authorize; UI cannot authorize alone; memory cannot authorize; plugins cannot escalate; Android cannot bypass Core; web content cannot become authority.
- **Data integrity:** State consistency, event ordering, audit integrity, and anchor protection are defined.
- **Testability:** Every security-critical guarantee has a concrete verification strategy.
- **Traceability:** Requirement → component → mechanism → interface → verification is documented for every requirement.

Both the internal adversarial audit and the independent team simulation pass with zero material findings. Residual risks are documented and irreducible within the stated scope.

**LOCKED at Architecture v1.2.0.**
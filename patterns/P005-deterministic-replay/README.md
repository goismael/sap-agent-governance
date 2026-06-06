# P005 — Deterministic Replay of Agent Decisions Against Close-Time Policy

**Category:** Governance Assurance  
**Risk Level:** High  
**SAP Version:** S/4HANA 2020+  
**Applies To:** Financial Close, Audit Remediation, Post-Incident Investigation, Regulatory Review

---

## The Problem

Every governance pattern in this library addresses what happens *before* and *during* agent execution — permission scoping, approval gates, audit logging, failure handling. P005 addresses what happens *after*: can you prove, to an auditor or regulator, that the agent's decision was correct *at the time it was made*?

This is a harder problem than it appears.

Consider what happens six months after a year-end close. An auditor questions why an AI agent posted a USD 1.9M intercompany elimination on December 31. Your P003 logs show what the agent did. Your P002 records show who authorized it. But the auditor asks a different question:

*"Under the materiality thresholds and intercompany matching policy that were in effect on December 31 — not today's policy — was the agent's decision compliant?"*

If your policy has been updated since close — materiality thresholds revised, matching tolerances tightened, new approval requirements added — replaying the decision against today's policy produces a different answer than replaying it against the policy that was live at close time. Without policy versioning and deterministic replay, you cannot answer the auditor's question with confidence.

This gap is not theoretical. Model version pinning and audit trail completeness are now standard questions in enterprise agentic AI procurement — organizations are expected to prove what happened, when, by whom, and against which policy. Most agent governance frameworks stop at logging what happened. P005 addresses the harder requirement: proving the decision was correct under the rules that existed when it was made.

The three failure modes this pattern prevents:

**Policy drift** — the governance policy changes between close and audit. Today's replay produces a different compliance verdict than the original decision, creating an unresolvable discrepancy.

**Model drift** — the LLM used for reasoning is updated or replaced between close and audit. The same inputs produce different reasoning, making the original decision unreproducible.

**Context drift** — reference data used at decision time (exchange rates, materiality thresholds, account master data) changes after close. Replay without frozen context produces incorrect results.

---

## Core Concepts

### Policy Version

A policy version is an immutable, timestamped snapshot of all governance rules that govern agent behavior at a specific point in time. For financial close agents, a policy version includes:

- Materiality thresholds by company code and document type
- Risk tier classifications for each SAP transaction code and BAPI
- Approval authority matrix (who can authorize what at what amount)
- Autonomous action boundaries (what agents can do without human approval)
- Escalation conditions (when agents must escalate rather than act)
- FX variance tolerances
- Intercompany matching tolerances

A policy version is **write-once**. Once a close cycle begins, the policy version for that cycle is sealed. It cannot be modified — only superseded by a new version for the next cycle.

### Close-Time Policy Binding

Every agent action in a governed close cycle must be bound to the policy version that was active when the action was taken. This binding is stored in the agent action log (P003) and is the key that enables deterministic replay.

```
Agent Action → Policy Version ID → Immutable Policy Snapshot
                                        ↓
                              Replay validates action
                              against THIS version,
                              not current version
```

### Deterministic Replay

Deterministic replay requires structured traces, deterministic metadata capture, and replay stubs for all non-deterministic components. The agent must be able to run in replay mode — interacting exclusively with deterministic stubs that feed back recorded results from the trace — and produce the same compliance verdict every time given the same inputs.

In the financial close context, non-deterministic components that must be frozen for replay include:

- The LLM reasoning engine (model ID + version must be recorded)
- Exchange rates (period-end rates at close time, not today)
- Materiality thresholds (from the sealed policy version)
- Account master data (GL account attributes at close time)
- Approval authority matrix (who had authority at close time)

---

## The Governance Design

### Step 1 — Policy Version Registry

Maintain a central policy version registry — an append-only store where each policy version has:

```json
{
  "policy_version_id": "FIN-CLOSE-POLICY-2026-Q4-v1.2",
  "effective_from": "2026-10-01T00:00:00Z",
  "effective_to": "2026-12-31T23:59:59Z",
  "sealed_at": "2026-10-01T00:00:00Z",
  "sealed_by": "MSANTOS",
  "checksum": "sha256-of-entire-policy-document",
  "policy": {
    "materiality_thresholds": {
      "single_posting_usd": 50000,
      "aggregate_cycle_usd": 200000,
      "fx_variance_pct": 0.02
    },
    "risk_tiers": {
      "BAPI_ACC_DOCUMENT_POST": "notify",
      "BAPI_ACC_DOCUMENT_REV_POST": "approve",
      "FB08": "approve",
      "FAGL_FC_VAL": "approve",
      "F110": "approve"
    },
    "approval_authority": {
      "approve_tier_usd_limit": {
        "Controller": 500000,
        "CFO": 999999999
      }
    },
    "autonomous_boundaries": {
      "max_single_posting_autonomous_usd": 10000,
      "allowed_doc_types_autonomous": ["SA"],
      "prohibited_autonomous_actions": ["FB08", "FAGL_FC_VAL", "F110"]
    },
    "model_config": {
      "narrative_model": "gemini-2.5-flash",
      "narrative_model_version": "2025-04",
      "reasoning_temperature": 0.1
    }
  }
}
```

The `checksum` field is a SHA-256 hash of the entire policy document. Any modification produces a different checksum — making tampering detectable.

### Step 2 — Close Cycle Policy Binding

At the start of every close cycle, the orchestrator:

1. Retrieves the current active policy version from the registry
2. Verifies the policy version checksum
3. Stores the `policy_version_id` in the close session record
4. All agents in that session inherit the bound policy version

Every P003 action log event includes the `policy_version_id`. This is the audit trail link between individual actions and the governance rules that applied when they were taken.

### Step 3 — Context Snapshot

At close cycle start, capture and freeze reference data that agents use in reasoning:

```python
@dataclass
class CloseContextSnapshot:
    """
    Immutable snapshot of all reference data used by agents
    during this close cycle. Stored alongside the policy version.
    Enables replay with the exact data that existed at close time.
    """
    snapshot_id: str
    close_cycle_id: str
    captured_at: str
    policy_version_id: str

    # Reference data frozen at close time
    exchange_rates: Dict[str, float]        # Currency pair → rate
    materiality_thresholds: Dict[str, float] # CC → threshold
    gl_account_master: Dict[str, Dict]      # GL account attributes
    cost_center_master: Dict[str, Dict]     # Cost center attributes
    posting_period_status: Dict[str, str]   # CC → open/closed
    approval_authority_matrix: Dict[str, float]  # Role → limit

    # Model configuration frozen at close time
    llm_model_id: str
    llm_model_version: str

    # Checksum of entire snapshot for tamper detection
    checksum: str
```

### Step 4 — Decision Trace Recording

Every agent decision must be recorded as a complete, self-contained trace — enough information to reproduce the decision without any external data lookups.

A decision trace extends the P003 reasoning log with replay-specific fields:

```json
{
  "trace_id": "uuid",
  "session_id": "close-cycle-2026-12-Q4-001",
  "policy_version_id": "FIN-CLOSE-POLICY-2026-Q4-v1.2",
  "context_snapshot_id": "snapshot-2026-12-31-001",
  "agent_id": "gl-reconciliation-agent-001",
  "timestamp": "2026-12-31T23:47:12Z",
  "decision_point": "period_close_accrual_assessment",

  "inputs": {
    "gl_balance": 1965960.17,
    "sub_ledger_balance": 1965960.17,
    "variance": 0.00,
    "exchange_rate_used": 1.0823,
    "materiality_threshold_applied": 50000,
    "risk_tier_applied": "notify"
  },

  "policy_evaluation": {
    "policy_version_id": "FIN-CLOSE-POLICY-2026-Q4-v1.2",
    "rules_evaluated": [
      {
        "rule": "materiality_threshold",
        "value_at_decision": 50000,
        "input_amount": 1965960.17,
        "result": "above_threshold",
        "action_triggered": "escalate_to_approve_tier"
      },
      {
        "rule": "fx_variance_tolerance",
        "value_at_decision": 0.02,
        "actual_variance": 0.00,
        "result": "within_tolerance",
        "action_triggered": "none"
      }
    ],
    "final_decision": "post_with_approval",
    "decision_compliant": true
  },

  "llm_trace": {
    "model_id": "gemini-2.5-flash",
    "model_version": "2025-04",
    "prompt_hash": "sha256-of-prompt",
    "response_hash": "sha256-of-response",
    "temperature": 0.1
  },

  "trace_checksum": "sha256-of-entire-trace"
}
```

### Step 5 — Deterministic Replay Engine

The replay engine takes a decision trace and re-evaluates the agent's decision against the frozen policy version and context snapshot — without any LLM calls, without any live SAP data lookups.

```python
class DecisionReplayEngine:
    """
    Replays agent decisions deterministically against the policy
    version and context snapshot that were active at close time.

    Does NOT call the LLM. Does NOT query live SAP data.
    Uses only frozen traces and immutable policy snapshots.

    Produces a ReplayVerdict: COMPLIANT, NON_COMPLIANT, or INCONCLUSIVE.
    """

    def replay(
        self,
        trace: DecisionTrace,
        policy_registry: PolicyVersionRegistry,
        context_store: ContextSnapshotStore,
    ) -> ReplayVerdict:

        # Step 1 — Retrieve and verify the policy version
        policy = policy_registry.get(trace.policy_version_id)
        if not policy:
            return ReplayVerdict(
                status="INCONCLUSIVE",
                reason=f"Policy version {trace.policy_version_id} not found in registry"
            )

        if not self._verify_checksum(policy):
            return ReplayVerdict(
                status="INCONCLUSIVE",
                reason="Policy version checksum mismatch — policy may have been tampered with"
            )

        # Step 2 — Retrieve and verify the context snapshot
        context = context_store.get(trace.context_snapshot_id)
        if not context:
            return ReplayVerdict(
                status="INCONCLUSIVE",
                reason=f"Context snapshot {trace.context_snapshot_id} not found"
            )

        # Step 3 — Re-evaluate each policy rule against frozen inputs
        rule_results = []
        for rule_eval in trace.policy_evaluation["rules_evaluated"]:
            result = self._evaluate_rule(
                rule_name=rule_eval["rule"],
                policy=policy,
                inputs=trace.inputs,
                original_value_at_decision=rule_eval["value_at_decision"],
            )
            rule_results.append(result)

        # Step 4 — Verify final decision matches re-evaluation
        replayed_decision = self._derive_decision(rule_results, policy)
        original_decision = trace.policy_evaluation["final_decision"]

        if replayed_decision != original_decision:
            return ReplayVerdict(
                status="NON_COMPLIANT",
                reason=(
                    f"Replayed decision '{replayed_decision}' does not match "
                    f"original decision '{original_decision}' under policy "
                    f"{trace.policy_version_id}"
                ),
                rule_results=rule_results,
                policy_version_id=trace.policy_version_id,
                original_decision=original_decision,
                replayed_decision=replayed_decision,
            )

        return ReplayVerdict(
            status="COMPLIANT",
            reason=(
                f"Decision '{replayed_decision}' confirmed compliant under "
                f"policy {trace.policy_version_id} as of {policy.effective_from}"
            ),
            rule_results=rule_results,
            policy_version_id=trace.policy_version_id,
            original_decision=original_decision,
            replayed_decision=replayed_decision,
        )

    def _evaluate_rule(
        self,
        rule_name: str,
        policy: PolicyVersion,
        inputs: Dict,
        original_value_at_decision: float,
    ) -> RuleEvaluationResult:
        """
        Re-evaluate a single policy rule using frozen policy values.
        Detects policy drift: if the rule value has changed since close,
        flag it — but still evaluate against the original close-time value.
        """
        current_value = self._get_rule_value(rule_name, policy)
        policy_drifted = current_value != original_value_at_decision

        # Always evaluate against close-time value, not current value
        result = self._apply_rule(rule_name, original_value_at_decision, inputs)

        return RuleEvaluationResult(
            rule=rule_name,
            close_time_value=original_value_at_decision,
            current_value=current_value,
            policy_drifted=policy_drifted,
            result=result,
        )

    def batch_replay(
        self,
        session_id: str,
        trace_store: DecisionTraceStore,
        policy_registry: PolicyVersionRegistry,
        context_store: ContextSnapshotStore,
    ) -> BatchReplayReport:
        """
        Replay all decisions in a close session.
        Produces a BatchReplayReport suitable for audit submission.
        """
        traces = trace_store.get_by_session(session_id)
        verdicts = []

        for trace in traces:
            verdict = self.replay(trace, policy_registry, context_store)
            verdicts.append(verdict)

        compliant = [v for v in verdicts if v.status == "COMPLIANT"]
        non_compliant = [v for v in verdicts if v.status == "NON_COMPLIANT"]
        inconclusive = [v for v in verdicts if v.status == "INCONCLUSIVE"]
        drifted = [v for v in verdicts if any(r.policy_drifted for r in v.rule_results)]

        return BatchReplayReport(
            session_id=session_id,
            total_decisions=len(verdicts),
            compliant=len(compliant),
            non_compliant=len(non_compliant),
            inconclusive=len(inconclusive),
            policy_drift_detected=len(drifted),
            verdicts=verdicts,
        )
```

---

## Audit Considerations

**1. "Can you prove the agent's decision was correct under the rules that existed at close?"**
Run `DecisionReplayEngine.batch_replay()` for the session. Present the `BatchReplayReport` — every decision replayed against its bound policy version, with a COMPLIANT/NON_COMPLIANT/INCONCLUSIVE verdict for each. The policy version checksum proves the rules haven't been modified since close.

**2. "How do I know the policy version hasn't been changed since close?"**
The policy version registry is append-only. Each version has a SHA-256 checksum of its entire content. Any modification produces a different checksum. Present the checksum at close time and the checksum today — if they match, the policy is unchanged.

**3. "What if your LLM was updated between close and audit?"**
The decision trace records the model ID and version used at close time. The replay engine re-evaluates the deterministic policy rules — it does not re-run the LLM. The LLM's reasoning output is captured in the trace as a hash. Auditors evaluate compliance against policy rules, not LLM outputs.

**4. "What if your materiality thresholds have changed since close?"**
Policy drift is explicitly detected and flagged in the replay output. The replay evaluates against the close-time threshold, not the current one. The `policy_drifted` flag alerts auditors to cases where the current policy differs from the close-time policy — which may itself require explanation.

**5. "Can you replay a specific decision that's under question?"**
Yes — retrieve the trace by `trace_id`, call `replay()` with the bound policy version and context snapshot. The replay is deterministic: given the same trace, policy version, and context snapshot, it produces the same verdict every time.

---

## Common Mistakes

| Mistake | Why It's Dangerous | Correct Approach |
|---|---|---|
| Replaying against current policy | Policy drift makes compliant decisions appear non-compliant — or vice versa | Always replay against the policy version bound at close time |
| Not recording model version | LLM updates make original reasoning unreproducible | Record model ID and version in every decision trace |
| Mutable policy store | Policy modifications after close invalidate replay | Append-only policy registry with checksums |
| Missing context snapshot | Replay uses live exchange rates or master data instead of close-time values | Freeze all reference data at close cycle start |
| Replaying with LLM calls | Non-deterministic LLM responses produce inconsistent verdicts | Replay engine evaluates deterministic policy rules only — no LLM calls |
| No policy drift detection | Policy changes go unnoticed — auditors discover them first | Explicitly flag and report every case where close-time and current policy values differ |

---

## Connection to P001–P004

P005 is the verification layer that sits above all previous patterns:

| Pattern | What P005 Verifies |
|---|---|
| P001 — Permission Scoping | Were the agent's permissions consistent with the policy version at close time? |
| P002 — Approval Gates | Did the approval authority matrix at close time authorize the approver who signed off? |
| P003 — Audit Logging | Are the decision traces complete enough to support deterministic replay? |
| P004 — Failure Handling | Were recovery actions taken under the recovery policy that existed at close time? |

Without P001–P004, P005 has nothing to replay. Without P005, P001–P004 can prove what happened but not that it was correct under the rules that existed when it happened.

---

## Credit

The deterministic replay problem in agentic AI governance was surfaced in a public discussion by Virendra Vaishnav (CTO, AIXPERTZ.ai), whose comment identified policy versioning at the cycle boundary as the missing layer in most agent governance frameworks. This pattern is a direct response to that observation.

---

## Related Patterns

- **P001** — Agent Permission Scoping Against SAP Authorization Objects
- **P002** — Human Approval Gates for High-Risk SAP Transaction Codes
- **P003** — Audit Logging for Agentic Workflows in SAP
- **P004** — Agent Failure Handling During Financial Period-End Close

---

## Changelog

| Version | Date | Change |
|---------|------|--------|
| 1.0.0 | 2026-05 | Initial release |

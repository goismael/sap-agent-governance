# P002 — Human Approval Gates for High-Risk SAP Transaction Codes

**Category:** Workflow Control  
**Risk Level:** Critical  
**SAP Version:** S/4HANA 2020+  
**Applies To:** Financial Close, General Ledger, Accounts Payable, Intercompany, FX Revaluation

---

## The Problem

Not all agent actions carry the same risk. An agent reading a trial balance is different from an agent posting a journal entry. An agent posting a standard accrual is different from one reversing a prior-period document or executing an intercompany elimination.

The failure mode this pattern addresses is simple: an agent with posting authority executes a consequential SAP action — a reversal, a period-end posting, an FX revaluation run — without a human ever seeing it. The action is within the agent's declared permission scope (covered in P001), it passes all validation checks, and it posts cleanly to the ledger. Nobody reviews it until the auditor asks why a document was reversed three minutes before period close.

This is not a hypothetical. It is the default behavior of any agentic financial close system where approval gates are not explicitly designed in. Agents don't pause unless you build the pause.

The challenge is designing approval gates that are:
- **Consequential** — triggered on actions that actually matter, not on every read
- **Practical** — fast enough that they don't become the bottleneck in a time-pressured close
- **Auditable** — producing a record that proves a human made an informed decision
- **Non-bypassable** — architecturally enforced, not just documented in a runbook

This pattern defines a risk classification framework for SAP transaction codes and BAPIs used by financial close agents, and shows how to implement approval gates that satisfy all four requirements.

---

## SAP Context

### The Transaction Codes That Matter Most

In a financial close context, these are the SAP transaction codes and BAPIs that agents are most likely to invoke — and that carry the highest risk if executed without human review:

| Transaction / BAPI | Function | Risk Level | Why It Matters |
|---|---|---|---|
| `BAPI_ACC_DOCUMENT_POST` | Post accounting document | High | Creates permanent ledger entries across all document types |
| `FB01` | Post document (manual) | High | Manual journal entry — high fraud and error risk |
| `FB08` | Reverse document | Critical | Reverses a posted document — affects prior period figures |
| `F.80` / `BAPI_ACC_DOCUMENT_REV_POST` | Mass reversal | Critical | Bulk reversal — exponential blast radius |
| `OB08` | Maintain exchange rates | Critical | Affects FX revaluation across all currency-denominated positions |
| `FAGL_FC_VAL` | FX revaluation run | Critical | Batch process affecting all open items in foreign currency |
| `F110` | Automatic payment run | Critical | Triggers outbound payments to vendors |
| `CJBDPLN` / `FAGLGVTR` | Balance carryforward | Critical | Closes fiscal year and carries balances forward — irreversible |
| `F-02` | Enter G/L account posting | High | Direct G/L posting — bypasses most validation workflows |
| `FB50` | Enter G/L account document | High | Fiori-based G/L posting — same risk as F-02 |
| `FBB1` | Post foreign currency valuation | High | Manual FX posting — affects revaluation accounts |
| `F.13` | Automatic clearing | High | Automatically clears open items — affects aging and reconciliation |
| `BAPI_COORDER_ACT_POST` | Post CO actual costs | Medium | Affects cost center and profit center reporting |

### Risk Classification Framework

Every SAP action an agent can take must be assigned to one of three risk tiers:

**Tier 1 — Autonomous** (no approval required)
Read-only operations. Display transactions. Balance queries. Report generation. Anomaly detection outputs. These never require approval gates.

**Tier 2 — Notify** (execute, then notify human)
Low-risk postings within predefined bounds: standard accruals below materiality threshold, routine clearing of matched items, FX postings within expected variance range. The agent executes, the human is notified asynchronously and can trigger a reversal if needed.

**Tier 3 — Approve** (pause, request approval, execute only after confirmation)
Any action in the high-risk transaction list above. Any action above the materiality threshold. Any reversal. Any batch process. Any period-close action. The agent prepares the action completely, presents it for review, and waits. It does not proceed without an explicit approval signal.

---

## The Governance Design

### Step 1 — Define Materiality Thresholds

Approval gate triggers should be calibrated to your organization's materiality thresholds. Every Tier 3 action requires approval regardless of amount — but Tier 2 actions cross into Tier 3 above the materiality threshold.

```
Example thresholds (calibrate to your entity):
  Single posting materiality:     > $50,000 USD equivalent
  Aggregate posting materiality:  > $200,000 USD in a single close cycle
  FX variance threshold:          > 2% deviation from period-average rate
  Reversal:                       Any amount — always Tier 3
  Batch process:                  Any — always Tier 3
```

Document your thresholds in a configuration file, not in code. They will change. Configuration changes should not require code deployments.

### Step 2 — Design the Approval Gate Interface

An approval gate has four components:

**1. Action Package** — everything the approver needs to make an informed decision, prepared by the agent before requesting approval:

```
- Action type (what BAPI/transaction will be called)
- Action parameters (company code, document type, amount, currency, posting date)
- Agent reasoning (why the agent determined this action is correct)
- Supporting data (the reconciliation output, variance analysis, or anomaly flag that led here)
- Risk tier and materiality classification
- Estimated reversibility (can this be undone cleanly if wrong?)
- Time sensitivity (does this block downstream agents if delayed?)
```

**2. Approval Channel** — where the approval request is sent. Options by urgency:

```
Period-end close (time-sensitive):  Microsoft Teams adaptive card + email
Standard close cycle:               Email with response link
Low-urgency batch:                  Queue in approval dashboard
```

**3. Approval Response** — what constitutes a valid approval:

```
Valid:    Explicit "Approve" action from authorized approver
          with timestamp, approver identity, and optional comment
Invalid:  Silence (timeout is NOT approval)
Invalid:  Approval from unauthorized role
Invalid:  Approval of a modified action (re-approval required)
```

**4. Approval Record** — immutable log entry created at the moment of approval, before the SAP action executes:

```
{
  "approval_id": "uuid",
  "action_package_hash": "sha256 of the exact action the agent will execute",
  "approver_id": "employee_id",
  "approver_role": "CFO | Controller | GL_Manager",
  "approved_at": "ISO8601 timestamp",
  "approval_channel": "teams | email | dashboard",
  "comment": "optional free text",
  "agent_id": "which agent requested approval",
  "session_id": "close cycle identifier"
}
```

### Step 3 — Implement Timeout and Escalation Logic

Approval gates must not block indefinitely. Define:

```
Initial notification:      T+0  → Primary approver notified
First reminder:            T+15 min → Reminder sent to primary approver
Escalation:                T+30 min → Controller notified (if CFO is primary)
Hard escalation:           T+60 min → CFO + Controller both notified
Timeout behavior:          T+90 min → Action CANCELLED, not auto-approved
                                      Agent escalates to orchestrator
                                      Human close team notified of blocked action
```

Timeout must result in cancellation, never in auto-approval. An agent that auto-approves on timeout is not an approval gate — it is a delay.

### Step 4 — Enforce the Gate Architecturally

The approval gate must be enforced at the agent execution layer, not in documentation. The pattern is:

```
Agent prepares action → Permission guard checks scope (P001)
                      → Risk classifier assigns tier
                      → If Tier 3: gate opens, execution blocked
                      → Approval package sent to approver
                      → Agent polls for approval signal
                      → If approved: action hash verified, SAP call executed
                      → If timeout/rejected: action cancelled, escalated
                      → Approval record written to audit log (P003)
```

The SAP BAPI call must never be reachable without passing through this sequence. If the approval gate can be bypassed by calling the BAPI directly from a different code path, the gate is not enforced — it is decorative.

---

## Reference Implementation

### Python: Risk Classifier and Approval Gate

```python
from dataclasses import dataclass, field
from typing import Optional, Dict, Any
from enum import Enum
import hashlib
import json
import uuid
import logging
from datetime import datetime, timezone

logger = logging.getLogger(__name__)


class RiskTier(Enum):
    AUTONOMOUS = "autonomous"   # No approval needed
    NOTIFY = "notify"           # Execute then notify
    APPROVE = "approve"         # Pause and require approval


class ApprovalStatus(Enum):
    PENDING = "pending"
    APPROVED = "approved"
    REJECTED = "rejected"
    TIMEOUT = "timeout"
    CANCELLED = "cancelled"


# Risk classification registry
# Maps BAPI/transaction identifiers to base risk tier
RISK_REGISTRY: Dict[str, RiskTier] = {
    # Critical — always Tier 3 regardless of amount
    "BAPI_ACC_DOCUMENT_REV_POST": RiskTier.APPROVE,   # Reversal
    "FB08": RiskTier.APPROVE,                          # Reverse document
    "F.80": RiskTier.APPROVE,                          # Mass reversal
    "OB08": RiskTier.APPROVE,                          # Exchange rate maintenance
    "FAGL_FC_VAL": RiskTier.APPROVE,                   # FX revaluation run
    "F110": RiskTier.APPROVE,                          # Payment run
    "FAGLGVTR": RiskTier.APPROVE,                      # Balance carryforward
    # High — Tier 3 above materiality, Tier 2 below
    "BAPI_ACC_DOCUMENT_POST": RiskTier.NOTIFY,         # Promoted to APPROVE if > threshold
    "FB01": RiskTier.NOTIFY,
    "F-02": RiskTier.NOTIFY,
    "FB50": RiskTier.NOTIFY,
    "FBB1": RiskTier.NOTIFY,
    "F.13": RiskTier.NOTIFY,
    # Read-only — always Tier 1
    "BAPI_GL_GETGLACCBALANCE": RiskTier.AUTONOMOUS,
    "BAPI_ACC_GL_POSTING_GET_DETAIL": RiskTier.AUTONOMOUS,
}

# Materiality thresholds (USD equivalent)
MATERIALITY_THRESHOLDS = {
    "single_posting": 50_000,
    "aggregate_cycle": 200_000,
    "fx_variance_pct": 0.02,
}


@dataclass
class ActionPackage:
    """
    Complete description of an action the agent intends to take.
    Prepared before requesting approval — never modified after.
    """
    action_type: str                    # BAPI or transaction code
    company_code: str
    document_type: Optional[str]
    amount_usd_equivalent: float
    currency: str
    posting_date: str
    agent_id: str
    agent_reasoning: str                # Why the agent determined this action is correct
    supporting_data: Dict[str, Any]     # Reconciliation output, variance analysis, etc.
    session_id: str
    time_sensitive: bool = False
    estimated_reversible: bool = True

    def __post_init__(self):
        self.package_id = str(uuid.uuid4())
        self.created_at = datetime.now(timezone.utc).isoformat()
        self._hash = self._compute_hash()

    def _compute_hash(self) -> str:
        """
        Deterministic hash of the exact action parameters.
        Verified at approval time — any parameter change invalidates the approval.
        """
        payload = {
            "action_type": self.action_type,
            "company_code": self.company_code,
            "document_type": self.document_type,
            "amount_usd_equivalent": self.amount_usd_equivalent,
            "currency": self.currency,
            "posting_date": self.posting_date,
        }
        return hashlib.sha256(
            json.dumps(payload, sort_keys=True).encode()
        ).hexdigest()

    @property
    def hash(self) -> str:
        return self._hash


class RiskClassifier:
    """
    Classifies an action package into a risk tier.
    Applies materiality thresholds to promote Tier 2 actions to Tier 3.
    """

    def classify(self, package: ActionPackage) -> RiskTier:
        base_tier = RISK_REGISTRY.get(package.action_type, RiskTier.APPROVE)

        # Critical actions are always Tier 3 — no amount threshold applies
        if base_tier == RiskTier.APPROVE:
            return RiskTier.APPROVE

        # Autonomous actions stay autonomous
        if base_tier == RiskTier.AUTONOMOUS:
            return RiskTier.AUTONOMOUS

        # Notify-tier actions are promoted to Approve if above materiality
        if package.amount_usd_equivalent > MATERIALITY_THRESHOLDS["single_posting"]:
            logger.info(
                f"Action {package.action_type} promoted from NOTIFY to APPROVE: "
                f"amount {package.amount_usd_equivalent} exceeds threshold "
                f"{MATERIALITY_THRESHOLDS['single_posting']}"
            )
            return RiskTier.APPROVE

        return RiskTier.NOTIFY


@dataclass
class ApprovalRecord:
    """Immutable record written at the moment of approval."""
    approval_id: str
    package_id: str
    action_package_hash: str
    approver_id: str
    approver_role: str
    approved_at: str
    approval_channel: str
    status: ApprovalStatus
    comment: Optional[str] = None
    agent_id: str = ""
    session_id: str = ""


class ApprovalGate:
    """
    Enforces human approval for Tier 3 actions.
    The SAP BAPI call is only reachable through this gate.
    """

    def __init__(self, notification_service, approval_store, audit_logger):
        self.notification_service = notification_service
        self.approval_store = approval_store
        self.audit_logger = audit_logger
        self.classifier = RiskClassifier()

        # Timeout ladder in minutes
        self.timeout_ladder = [15, 30, 60, 90]

    def request_and_wait(self, package: ActionPackage) -> ApprovalRecord:
        """
        Main entry point. Classifies the action, routes it appropriately,
        and returns an ApprovalRecord. Raises if approval not granted.
        """
        tier = self.classifier.classify(package)

        if tier == RiskTier.AUTONOMOUS:
            return self._auto_approve(package)

        if tier == RiskTier.NOTIFY:
            return self._notify_and_proceed(package)

        # Tier 3 — full approval gate
        return self._require_approval(package)

    def _auto_approve(self, package: ActionPackage) -> ApprovalRecord:
        record = ApprovalRecord(
            approval_id=str(uuid.uuid4()),
            package_id=package.package_id,
            action_package_hash=package.hash,
            approver_id="SYSTEM",
            approver_role="AUTONOMOUS",
            approved_at=datetime.now(timezone.utc).isoformat(),
            approval_channel="none",
            status=ApprovalStatus.APPROVED,
            agent_id=package.agent_id,
            session_id=package.session_id,
        )
        self.audit_logger.log_approval(record)
        return record

    def _notify_and_proceed(self, package: ActionPackage) -> ApprovalRecord:
        record = ApprovalRecord(
            approval_id=str(uuid.uuid4()),
            package_id=package.package_id,
            action_package_hash=package.hash,
            approver_id="SYSTEM",
            approver_role="NOTIFY",
            approved_at=datetime.now(timezone.utc).isoformat(),
            approval_channel="async_notification",
            status=ApprovalStatus.APPROVED,
            agent_id=package.agent_id,
            session_id=package.session_id,
        )
        self.notification_service.send_async_notification(package)
        self.audit_logger.log_approval(record)
        return record

    def _require_approval(self, package: ActionPackage) -> ApprovalRecord:
        """Full blocking approval gate for Tier 3 actions."""
        # Send initial notification
        self.notification_service.send_approval_request(package)
        self.audit_logger.log_approval_requested(package)

        # Poll with escalation ladder
        elapsed = 0
        for threshold in self.timeout_ladder:
            approval = self.approval_store.poll(
                package_id=package.package_id,
                timeout_minutes=(threshold - elapsed),
            )

            if approval and approval.status == ApprovalStatus.APPROVED:
                # Critical: verify hash before proceeding
                # Any parameter change since approval was requested = reject
                if approval.action_package_hash != package.hash:
                    raise ValueError(
                        f"Approval hash mismatch for package {package.package_id}. "
                        f"Action parameters may have changed. Re-approval required."
                    )
                self.audit_logger.log_approval(approval)
                return approval

            if approval and approval.status == ApprovalStatus.REJECTED:
                self.audit_logger.log_approval(approval)
                raise PermissionError(
                    f"Action {package.action_type} rejected by {approval.approver_id}"
                )

            # Escalate
            elapsed = threshold
            if threshold < self.timeout_ladder[-1]:
                self.notification_service.escalate(package, elapsed_minutes=elapsed)

        # Hard timeout — cancel, never auto-approve
        timeout_record = ApprovalRecord(
            approval_id=str(uuid.uuid4()),
            package_id=package.package_id,
            action_package_hash=package.hash,
            approver_id="SYSTEM",
            approver_role="TIMEOUT",
            approved_at=datetime.now(timezone.utc).isoformat(),
            approval_channel="timeout",
            status=ApprovalStatus.TIMEOUT,
            agent_id=package.agent_id,
            session_id=package.session_id,
        )
        self.audit_logger.log_approval(timeout_record)
        self.notification_service.notify_timeout(package)

        raise TimeoutError(
            f"Approval timeout for action {package.action_type} "
            f"(package {package.package_id}). Action CANCELLED. "
            f"Human close team notified."
        )


# --- Usage Example ---

def post_journal_entry_with_gate(gate: ApprovalGate, sap_client, entry_data: dict):
    """
    Example: posting a journal entry through the approval gate.
    The BAPI call is only reachable after gate.request_and_wait() returns.
    """
    package = ActionPackage(
        action_type="BAPI_ACC_DOCUMENT_POST",
        company_code=entry_data["company_code"],
        document_type=entry_data["document_type"],
        amount_usd_equivalent=entry_data["amount_usd"],
        currency=entry_data["currency"],
        posting_date=entry_data["posting_date"],
        agent_id="journal-entry-agent-001",
        agent_reasoning=entry_data["reasoning"],
        supporting_data=entry_data["reconciliation_output"],
        session_id=entry_data["close_session_id"],
        time_sensitive=True,
        estimated_reversible=True,
    )

    # Gate blocks here until approved, rejected, or timeout
    approval_record = gate.request_and_wait(package)

    # Only reachable with a valid, hash-verified approval
    result = sap_client.call_bapi(
        "BAPI_ACC_DOCUMENT_POST",
        parameters=entry_data["bapi_params"],
        approval_id=approval_record.approval_id,
    )

    return result
```

---

## Audit Considerations

**1. "How do you prove a human approved this posting?"**
Present the `ApprovalRecord` for the document. It contains the approver identity, role, timestamp, approval channel, and a hash of the exact action parameters that were approved. Cross-reference the hash against the posted document's parameters.

**2. "What prevents an agent from bypassing the approval gate?"**
The BAPI call is only reachable through `ApprovalGate.request_and_wait()`. Show the code architecture and the absence of any alternative call path. This is an architectural control, not a process control.

**3. "What happens if the approver doesn't respond?"**
Timeout results in cancellation, not auto-approval. Show the timeout record in the audit log and the escalation sequence that preceded it.

**4. "Can an agent approve its own actions?"**
No. The approval store validates approver identity against an authorized approver registry. Agents authenticate as system users — they are not in the approver registry.

**5. "What if the action parameters change between approval and execution?"**
The hash verification step catches this. If the action package hash at execution time differs from the hash in the approval record, the action is rejected and re-approval is required.

**6. "How do you handle period-close time pressure?"**
Time sensitivity is flagged in the action package. The notification service uses the fastest channel (Teams adaptive card) for time-sensitive actions. The escalation ladder is compressed for period-close sessions.

---

## Common Mistakes

| Mistake | Why It's Dangerous | Correct Approach |
|---|---|---|
| Treating silence as approval | Agent proceeds when nobody responds | Timeout always cancels — never auto-approves |
| Approving the action type, not the parameters | Approver says "yes to reversals" — agent reverses anything | Hash verification ties approval to exact parameters |
| Single approval channel | Teams is down, nobody gets notified | Primary + fallback channels; escalation ladder |
| No escalation logic | Approval request sits unread | Defined escalation ladder with hard timeout |
| Approval gate in documentation only | Agent can call BAPI directly if gate is skipped | Gate enforced architecturally — BAPI unreachable without it |
| Shared approver queue | No accountability for who approved what | Named approver identity required in every approval record |
| Gate triggers on every action | Approval fatigue — approvers start rubber-stamping | Tier classification — autonomous actions never need approval |

---

## Related Patterns

- **P001** — Agent Permission Scoping Against SAP Authorization Objects
- **P003** — Audit Logging for Agentic Workflows in SAP
- **P004** — Agent Failure Handling During Financial Period-End Close

---

## Changelog

| Version | Date | Change |
|---------|------|--------|
| 1.0.0 | 2026-05 | Initial release |

# P003 — Audit Logging for Agentic Workflows in SAP

**Category:** Observability  
**Risk Level:** High  
**SAP Version:** S/4HANA 2020+  
**Applies To:** Financial Close, General Ledger, Accounts Payable, Intercompany, Any regulated agentic workflow

---

## The Problem

Traditional application logs capture system events: API calls, error codes, response times. They answer the question *"Did the system work?"*

Auditors reviewing an AI-driven financial close need to answer a different question: *"Why did the agent do what it did — and who authorized it?"*

These are not the same question. A system log might show that `BAPI_ACC_DOCUMENT_POST` was called at 23:47:12 with parameters X, Y, Z. It will not show:

- What data the agent analyzed before deciding to post
- What anomalies it considered and dismissed
- What reasoning led it to choose document type SA over AB
- Whether the posting was within the agent's declared permission scope
- Who approved it, when, and through which channel
- What happened to downstream agents that depended on this action

Without this context, an auditor cannot distinguish between an agent that made a correct, well-reasoned decision and one that made an erroneous decision that happened to produce a valid SAP document. Both look identical in a system log.

This gap matters beyond audit. When agents can request, grant, or route access, the classic question "Who approved this?" becomes "Which autonomous process did what, under which policy, and can we prove it end-to-end?" That proof depends on structured, immutable, semantically rich audit logs — not application event streams.

This pattern defines a logging schema and implementation approach for agentic SAP workflows that satisfies SOX, internal audit, and external audit requirements for financial close environments.

---

## The Four Layers of Agentic Audit Logging

A complete audit trail for an agentic SAP workflow requires four distinct logging layers. Most implementations capture only the first. All four are required for SOX-grade auditability.

### Layer 1 — SAP System Log (Already Exists)
What SAP natively captures: document numbers, posting dates, user IDs (service user), transaction codes, change document history. This is necessary but not sufficient. It tells you *what* happened in SAP. It does not tell you *why* the agent initiated it.

### Layer 2 — Agent Action Log (Must Build)
Every discrete action an agent takes — every tool call, every API invocation, every decision to proceed or escalate — logged with the agent's identity, the action parameters, and the outcome. This is the operational record of the agent's behavior.

### Layer 3 — Agent Reasoning Log (Must Build)
The agent's reasoning at each decision point — what data it considered, what alternatives it evaluated, what drove its conclusion. This is the explainability record. Without it, you can prove the agent acted but not that it acted correctly.

### Layer 4 — Approval and Escalation Log (Covered in P002, referenced here)
The human authorization record: who approved what, when, through which channel, and the hash-verified link between the approval and the exact action executed. This layer connects human accountability to agent execution.

All four layers must be:
- **Immutable** — written once, never modified
- **Tamper-evident** — hash-chained so deletions or modifications are detectable
- **Correlated** — linked by a common session ID and action ID so a complete audit trail can be reconstructed across all four layers
- **Retained** — per your organization's records retention policy (typically 7 years for SOX)

---

## The Logging Schema

### Core Identifiers (Present in Every Log Event)

Every log event — regardless of layer — must carry these fields:

```json
{
  "event_id": "uuid-v4",
  "session_id": "close-cycle-2026-05-Q1-001",
  "agent_id": "gl-reconciliation-agent-001",
  "agent_classification": "analyzer",
  "timestamp": "2026-05-19T23:47:12.334Z",
  "sequence_number": 47,
  "previous_event_hash": "sha256-of-previous-event",
  "layer": "action | reasoning | approval | system"
}
```

The `sequence_number` and `previous_event_hash` fields implement a hash chain — each event's hash is embedded in the next event, making any deletion or modification detectable during audit.

---

### Layer 2 — Agent Action Log Schema

```json
{
  "event_id": "uuid",
  "session_id": "close-cycle-2026-05-Q1-001",
  "agent_id": "gl-reconciliation-agent-001",
  "agent_classification": "analyzer",
  "timestamp": "2026-05-19T23:47:12.334Z",
  "sequence_number": 47,
  "previous_event_hash": "sha256...",
  "layer": "action",

  "action": {
    "type": "tool_call | sap_bapi | sap_odata | escalation | approval_request | completion",
    "name": "BAPI_GL_GETGLACCBALANCE",
    "parameters": {
      "company_code": "1000",
      "gl_account": "100000",
      "fiscal_year": "2026",
      "period": "04"
    },
    "risk_tier": "autonomous | notify | approve",
    "permission_check_passed": true,
    "permission_guard_version": "1.2.0"
  },

  "outcome": {
    "status": "success | failure | timeout | escalated",
    "duration_ms": 234,
    "error_code": null,
    "error_message": null,
    "sap_document_number": null,
    "sap_return_code": "S"
  },

  "approval_id": null
}
```

For posting actions (Tier 2/3), `sap_document_number` must be populated on success, creating a direct link between the agent action log and the SAP document.

---

### Layer 3 — Agent Reasoning Log Schema

```json
{
  "event_id": "uuid",
  "session_id": "close-cycle-2026-05-Q1-001",
  "agent_id": "gl-reconciliation-agent-001",
  "timestamp": "2026-05-19T23:47:10.001Z",
  "sequence_number": 46,
  "previous_event_hash": "sha256...",
  "layer": "reasoning",

  "reasoning": {
    "decision_point": "reconciliation_variance_assessment",
    "input_data_summary": {
      "gl_balance": 4823901.22,
      "sub_ledger_balance": 4823901.22,
      "variance": 0.00,
      "currency": "USD",
      "company_code": "1000",
      "period": "2026-04"
    },
    "alternatives_considered": [
      {
        "option": "flag_for_manual_review",
        "reason_rejected": "Variance is zero — no discrepancy requiring review"
      },
      {
        "option": "escalate_to_controller",
        "reason_rejected": "Balance within tolerance; escalation criteria not met"
      }
    ],
    "conclusion": "reconciliation_passed",
    "confidence": "high",
    "conclusion_reasoning": "GL balance and sub-ledger balance match exactly. No FX adjustment required. Period-end rate matches transaction-date rate for all line items. Proceeding to sign-off recommendation.",
    "downstream_actions_triggered": ["sign_off_agent_notify"],
    "human_review_recommended": false,
    "human_review_reason": null
  }
}
```

The `alternatives_considered` field is critical for audit. It demonstrates that the agent evaluated options before acting — not that it blindly executed the first available action.

---

### Layer 2 — Escalation and Failure Events

```json
{
  "event_id": "uuid",
  "session_id": "close-cycle-2026-05-Q1-001",
  "agent_id": "fx-revaluation-agent-001",
  "timestamp": "2026-05-19T23:51:44.001Z",
  "sequence_number": 63,
  "previous_event_hash": "sha256...",
  "layer": "action",

  "action": {
    "type": "escalation",
    "name": "escalate_to_controller",
    "escalation_reason": "FX rate deviation exceeds 2% threshold",
    "escalation_data": {
      "expected_rate": 1.0823,
      "actual_rate": 1.1102,
      "deviation_pct": 2.58,
      "threshold_pct": 2.00,
      "affected_company_codes": ["2000", "3000"],
      "estimated_impact_usd": 184200.00
    },
    "risk_tier": "approve",
    "escalation_target": "controller_approval_queue"
  },

  "outcome": {
    "status": "escalated",
    "duration_ms": 12,
    "error_code": null
  }
}
```

---

## Reference Implementation

### Python: Immutable Audit Logger with Hash Chain

```python
import hashlib
import json
import uuid
import logging
from dataclasses import dataclass, field, asdict
from datetime import datetime, timezone
from typing import Optional, Any, Dict, List
from enum import Enum
from threading import Lock


class LogLayer(Enum):
    ACTION = "action"
    REASONING = "reasoning"
    APPROVAL = "approval"
    SYSTEM = "system"


class ActionType(Enum):
    TOOL_CALL = "tool_call"
    SAP_BAPI = "sap_bapi"
    SAP_ODATA = "sap_odata"
    ESCALATION = "escalation"
    APPROVAL_REQUEST = "approval_request"
    APPROVAL_RECEIVED = "approval_received"
    COMPLETION = "completion"
    FAILURE = "failure"


@dataclass
class AuditEvent:
    """
    Base audit event. Every log entry inherits this structure.
    Once created, fields are never modified.
    """
    session_id: str
    agent_id: str
    agent_classification: str
    layer: LogLayer
    payload: Dict[str, Any]

    # Set at creation time — never modified
    event_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    timestamp: str = field(
        default_factory=lambda: datetime.now(timezone.utc).isoformat()
    )
    sequence_number: int = field(default=0)
    previous_event_hash: Optional[str] = field(default=None)
    event_hash: Optional[str] = field(default=None)

    def compute_hash(self) -> str:
        """
        Deterministic hash of all event fields except event_hash itself.
        Used to build the hash chain.
        """
        hashable = {
            "event_id": self.event_id,
            "session_id": self.session_id,
            "agent_id": self.agent_id,
            "timestamp": self.timestamp,
            "sequence_number": self.sequence_number,
            "previous_event_hash": self.previous_event_hash,
            "layer": self.layer.value,
            "payload": self.payload,
        }
        return hashlib.sha256(
            json.dumps(hashable, sort_keys=True).encode()
        ).hexdigest()

    def to_dict(self) -> Dict[str, Any]:
        d = asdict(self)
        d["layer"] = self.layer.value
        return d


class AgentAuditLogger:
    """
    Immutable, hash-chained audit logger for agentic SAP workflows.

    Thread-safe. Each session maintains its own sequence counter
    and hash chain. All events are written to the configured sink
    (structured log, SIEM, immutable storage) immediately.

    Usage:
        logger = AgentAuditLogger(
            session_id="close-2026-Q1",
            agent_id="gl-reconciliation-agent-001",
            agent_classification="analyzer",
            sink=AzureMonitorSink(workspace_id=...)
        )
        logger.log_action(...)
        logger.log_reasoning(...)
    """

    def __init__(self, session_id: str, agent_id: str,
                 agent_classification: str, sink):
        self.session_id = session_id
        self.agent_id = agent_id
        self.agent_classification = agent_classification
        self.sink = sink
        self._lock = Lock()
        self._sequence = 0
        self._last_hash: Optional[str] = None
        self._logger = logging.getLogger(__name__)

    def log_action(
        self,
        action_type: ActionType,
        action_name: str,
        parameters: Dict[str, Any],
        outcome_status: str,
        risk_tier: str,
        permission_check_passed: bool,
        duration_ms: int,
        sap_document_number: Optional[str] = None,
        sap_return_code: Optional[str] = None,
        error_code: Optional[str] = None,
        error_message: Optional[str] = None,
        approval_id: Optional[str] = None,
        escalation_data: Optional[Dict] = None,
    ) -> AuditEvent:
        """Log an agent action — tool call, BAPI call, escalation, etc."""
        payload = {
            "action": {
                "type": action_type.value,
                "name": action_name,
                "parameters": parameters,
                "risk_tier": risk_tier,
                "permission_check_passed": permission_check_passed,
            },
            "outcome": {
                "status": outcome_status,
                "duration_ms": duration_ms,
                "sap_document_number": sap_document_number,
                "sap_return_code": sap_return_code,
                "error_code": error_code,
                "error_message": error_message,
            },
            "approval_id": approval_id,
            "escalation_data": escalation_data,
        }
        return self._write(LogLayer.ACTION, payload)

    def log_reasoning(
        self,
        decision_point: str,
        input_data_summary: Dict[str, Any],
        alternatives_considered: List[Dict[str, str]],
        conclusion: str,
        conclusion_reasoning: str,
        confidence: str,
        human_review_recommended: bool,
        human_review_reason: Optional[str] = None,
        downstream_actions_triggered: Optional[List[str]] = None,
    ) -> AuditEvent:
        """Log agent reasoning at a decision point."""
        payload = {
            "reasoning": {
                "decision_point": decision_point,
                "input_data_summary": input_data_summary,
                "alternatives_considered": alternatives_considered,
                "conclusion": conclusion,
                "confidence": confidence,
                "conclusion_reasoning": conclusion_reasoning,
                "human_review_recommended": human_review_recommended,
                "human_review_reason": human_review_reason,
                "downstream_actions_triggered": downstream_actions_triggered or [],
            }
        }
        return self._write(LogLayer.REASONING, payload)

    def log_approval_requested(
        self,
        package_id: str,
        action_type: str,
        action_hash: str,
        approver_target: str,
        channel: str,
    ) -> AuditEvent:
        """Log that an approval request was sent."""
        payload = {
            "approval_request": {
                "package_id": package_id,
                "action_type": action_type,
                "action_hash": action_hash,
                "approver_target": approver_target,
                "channel": channel,
                "status": "requested",
            }
        }
        return self._write(LogLayer.APPROVAL, payload)

    def log_approval_received(
        self,
        package_id: str,
        approval_id: str,
        approver_id: str,
        approver_role: str,
        status: str,
        action_hash_verified: bool,
        channel: str,
        comment: Optional[str] = None,
    ) -> AuditEvent:
        """Log an approval decision — approved, rejected, or timeout."""
        payload = {
            "approval_received": {
                "package_id": package_id,
                "approval_id": approval_id,
                "approver_id": approver_id,
                "approver_role": approver_role,
                "status": status,
                "action_hash_verified": action_hash_verified,
                "channel": channel,
                "comment": comment,
            }
        }
        return self._write(LogLayer.APPROVAL, payload)

    def log_session_start(self, close_cycle: str, entities: List[str]) -> AuditEvent:
        """Log the start of a close session."""
        payload = {
            "session": {
                "event": "session_start",
                "close_cycle": close_cycle,
                "entities": entities,
            }
        }
        return self._write(LogLayer.SYSTEM, payload)

    def log_session_end(
        self,
        status: str,
        actions_taken: int,
        actions_autonomous: int,
        actions_approved: int,
        actions_escalated: int,
        actions_failed: int,
    ) -> AuditEvent:
        """Log the end of a close session with summary statistics."""
        payload = {
            "session": {
                "event": "session_end",
                "status": status,
                "summary": {
                    "actions_taken": actions_taken,
                    "actions_autonomous": actions_autonomous,
                    "actions_approved": actions_approved,
                    "actions_escalated": actions_escalated,
                    "actions_failed": actions_failed,
                }
            }
        }
        return self._write(LogLayer.SYSTEM, payload)

    def _write(self, layer: LogLayer, payload: Dict[str, Any]) -> AuditEvent:
        """
        Thread-safe event write. Assigns sequence number,
        computes hash chain, writes to sink.
        Never raises — audit logging must not disrupt agent execution.
        Returns the written event for reference.
        """
        with self._lock:
            self._sequence += 1
            event = AuditEvent(
                session_id=self.session_id,
                agent_id=self.agent_id,
                agent_classification=self.agent_classification,
                layer=layer,
                payload=payload,
                sequence_number=self._sequence,
                previous_event_hash=self._last_hash,
            )
            event.event_hash = event.compute_hash()
            self._last_hash = event.event_hash

            try:
                self.sink.write(event.to_dict())
            except Exception as e:
                # Log locally but never raise — audit failure
                # must not block agent execution
                self._logger.critical(
                    "AUDIT_SINK_FAILURE",
                    extra={
                        "event_id": event.event_id,
                        "error": str(e),
                        "agent_id": self.agent_id,
                        "session_id": self.session_id,
                    }
                )
                # Secondary sink fallback (local file) should
                # be configured for sink failure scenarios

            return event


class AuditChainVerifier:
    """
    Verifies the integrity of an audit log chain for a given session.
    Used during internal audit or investigation to detect tampering.
    """

    def verify_chain(self, events: List[Dict]) -> Dict[str, Any]:
        """
        Given a list of events ordered by sequence_number,
        verify the hash chain is intact.
        Returns a verification report.
        """
        if not events:
            return {"status": "empty", "verified": False}

        errors = []
        sorted_events = sorted(events, key=lambda e: e["sequence_number"])

        for i, event in enumerate(sorted_events):
            # Recompute hash from event fields
            hashable = {
                "event_id": event["event_id"],
                "session_id": event["session_id"],
                "agent_id": event["agent_id"],
                "timestamp": event["timestamp"],
                "sequence_number": event["sequence_number"],
                "previous_event_hash": event["previous_event_hash"],
                "layer": event["layer"],
                "payload": event["payload"],
            }
            computed = hashlib.sha256(
                json.dumps(hashable, sort_keys=True).encode()
            ).hexdigest()

            if computed != event.get("event_hash"):
                errors.append({
                    "sequence": event["sequence_number"],
                    "event_id": event["event_id"],
                    "error": "Hash mismatch — event may have been tampered with"
                })

            # Verify chain linkage
            if i > 0:
                expected_prev = sorted_events[i - 1]["event_hash"]
                actual_prev = event.get("previous_event_hash")
                if actual_prev != expected_prev:
                    errors.append({
                        "sequence": event["sequence_number"],
                        "event_id": event["event_id"],
                        "error": "Chain break — previous hash does not match"
                    })

        return {
            "status": "verified" if not errors else "compromised",
            "verified": len(errors) == 0,
            "events_checked": len(sorted_events),
            "errors": errors,
        }
```

---

## Sink Configuration

The audit logger writes to a configurable sink. For enterprise SAP environments, the recommended sinks in order of preference:

### Azure Monitor / Log Analytics (Recommended for Azure-hosted agents)

```python
class AzureMonitorSink:
    """
    Writes audit events to Azure Log Analytics workspace.
    Use Immutable Storage policy on the workspace to prevent deletion.
    """
    def __init__(self, workspace_id: str, shared_key: str, log_type: str = "SAPAgentAudit"):
        self.workspace_id = workspace_id
        self.shared_key = shared_key
        self.log_type = log_type

    def write(self, event: Dict[str, Any]) -> None:
        # Use Azure Monitor Data Collector API
        # Implement signature generation per Azure docs
        body = json.dumps([event])
        # POST to https://{workspace_id}.ods.opinsights.azure.com/api/logs
        # with Log-Type: {self.log_type}
        pass
```

### Structured Local Log (Fallback / Development)

```python
import logging
import json

class StructuredFileSink:
    """
    Writes NDJSON (newline-delimited JSON) to a local file.
    Suitable for development and as a secondary fallback sink.
    Production: use append-only storage with restricted delete permissions.
    """
    def __init__(self, filepath: str):
        self.filepath = filepath

    def write(self, event: Dict[str, Any]) -> None:
        with open(self.filepath, "a", encoding="utf-8") as f:
            f.write(json.dumps(event) + "\n")
```

**Sink requirements for SOX environments:**
- Immutable or append-only storage — no delete or overwrite permissions for agent service accounts
- Retention minimum 7 years — align with your SOX records retention policy
- Access logging on the sink itself — who accessed the audit log is itself auditable
- Geographic residency — confirm log storage region meets data residency requirements

---

## Audit Reconstruction Query

Given a session ID, a complete audit trail should be reconstructable in under 5 minutes. Example KQL query for Azure Log Analytics:

```kql
SAPAgentAudit_CL
| where session_id_s == "close-cycle-2026-05-Q1-001"
| order by sequence_number_d asc
| project
    sequence_number_d,
    timestamp_t,
    agent_id_s,
    layer_s,
    ['payload_action_type_s'],
    ['payload_action_name_s'],
    ['payload_outcome_status_s'],
    ['payload_outcome_sap_document_number_s'],
    ['payload_reasoning_conclusion_s'],
    ['payload_approval_received_approver_id_s'],
    event_hash_s,
    previous_event_hash_s
```

---

## Audit Considerations

**1. "Show me every action the agent took during the April close."**
Query Layer 2 events for `session_id = "close-cycle-2026-04-*"`, ordered by sequence number. Every tool call, BAPI invocation, escalation, and completion is present with timestamp, parameters, and outcome.

**2. "Why did the agent post this specific document?"**
Cross-reference the SAP document number in the Layer 2 action log to find the event ID. Then retrieve the Layer 3 reasoning event with the same or preceding sequence number in the same session. The reasoning log shows what data the agent analyzed and what conclusion it reached.

**3. "Who authorized this posting?"**
The action log event for the posting contains an `approval_id`. Retrieve the Layer 4 approval event with that ID. It contains the approver identity, role, timestamp, channel, and hash verification status.

**4. "How do I know the logs haven't been tampered with?"**
Run `AuditChainVerifier.verify_chain()` against the session's events. Any deletion, modification, or insertion will produce a hash mismatch or chain break. Present the verification report.

**5. "Are these logs retained long enough?"**
Show the Azure Monitor workspace retention policy or equivalent, set to 2,557 days (7 years). Show that the agent service account has no delete permissions on the log workspace.

**6. "What if the logging system fails mid-session?"**
The logger catches sink failures without raising — agent execution continues. Sink failures are logged locally as `AUDIT_SINK_FAILURE` critical events. A secondary sink (local NDJSON file) should be configured. Sink failure monitoring should alert the close team immediately.

---

## Common Mistakes

| Mistake | Why It's Dangerous | Correct Approach |
|---|---|---|
| Logging only SAP system events | Cannot explain agent reasoning or prove human authorization | All four layers required |
| Mutable log storage | Logs can be altered after the fact — audit trail is unreliable | Append-only, immutable storage with access controls |
| No hash chain | Deletions are undetectable | Hash-chain every event; verify chain during audit |
| Logging PII in reasoning payloads | Data privacy violation; audit log becomes a liability | Summarize sensitive data; never log raw PII |
| Raising on sink failure | Agent execution halts mid-close when logging fails | Catch sink errors; use secondary fallback; alert and continue |
| No session summary event | Cannot quickly assess what the agent did in aggregate | Always log session start and end with action statistics |
| Shared log stream across agents | Cannot isolate one agent's audit trail | Per-agent session IDs; filterable by agent_id |
| Retention under 7 years | SOX requires 7-year retention for financial records | Set retention policy explicitly; verify quarterly |

---

## Related Patterns

- **P001** — Agent Permission Scoping Against SAP Authorization Objects
- **P002** — Human Approval Gates for High-Risk SAP Transaction Codes
- **P004** — Agent Failure Handling During Financial Period-End Close

---

## Changelog

| Version | Date | Change |
|---------|------|--------|
| 1.0.0 | 2026-05 | Initial release |

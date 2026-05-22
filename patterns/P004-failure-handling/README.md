# P004 — Agent Failure Handling During Financial Period-End Close

**Category:** Resilience  
**Risk Level:** High  
**SAP Version:** S/4HANA 2020+  
**Applies To:** Financial Close, General Ledger, Intercompany, FX Revaluation, Period-End Batch Processes

---

## The Problem

Period-end close operates under the most unforgiving time constraint in enterprise finance. Month-end, quarter-end, and year-end close windows are measured in hours, not days. When an AI agent fails mid-workflow in this environment, the consequences are not just technical — they are financial and reputational.

The failure modes that matter most in agentic financial close systems are not crashes. Crashes are obvious and recoverable. The dangerous failure modes are:

**Partial execution** — the agent posts some journal entries but not others before failing. The ledger is now in an inconsistent state. Which entries posted? Which didn't? Is the trial balance reliable?

**Silent failure** — the agent encounters an error, logs it, and stops. Nothing else in the workflow knows it stopped. Downstream agents that depend on its output proceed with stale or missing data.

**Duplicate execution** — the agent is retried after a failure, re-executes actions that already completed, and posts duplicate journal entries. SAP may or may not reject them depending on document type and configuration.

**Inconsistent rollback** — the agent attempts to reverse its partial work but the reversal itself fails, leaving the ledger in a state that is neither fully posted nor fully clean.

**Timeout during approval gate** — an agent pauses waiting for human approval (P002), the close window passes, and now the agent is holding a half-completed workflow with an expired context.

Each of these failure modes requires a different recovery strategy. A single "retry on failure" approach handles none of them correctly.

The core capability for agents in period-close workflows is reliability under failure — agents that can checkpoint progress and resume safely mid-flow, paired with validation and reconciliation, turning what used to be manual work into governed, repeatable automation. This pattern defines how to build that reliability.

---

## SAP Context

### Period-Lock Tables and Posting Period Controls

SAP controls which periods are open for posting via the Posting Period Variant, maintained in transaction `OB52`. Understanding period lock behavior is essential for failure recovery:

| Scenario | SAP Behavior | Recovery Implication |
|---|---|---|
| Period is open | Normal posting allowed | Standard retry is safe |
| Period is closed mid-recovery | Postings rejected with error AA390 | Recovery must reopen period or escalate |
| Year-end close initiated (`FAGLGVTR`) | Balance carryforward locked | No new postings until carryforward complete |
| Document already posted (duplicate) | SAP returns existing document number or error | Idempotency key check required before retry |

### BAPIs with Built-in Rollback Support

Some SAP BAPIs support transactional rollback natively. Know which ones do and which don't:

| BAPI | Rollback Support | Notes |
|---|---|---|
| `BAPI_ACC_DOCUMENT_POST` | Yes — `BAPI_TRANSACTION_ROLLBACK` | Must be called in same LUW |
|  `BAPI_ACC_DOCUMENT_REV_POST` | Yes — `BAPI_TRANSACTION_ROLLBACK` | Reversal can itself be rolled back |
| `FAGL_FC_VAL` (FX revaluation) | Partial — session-level | Batch session must be cancelled via SM37 |
| `F110` (Payment run) | No — payments may already be transmitted | Requires manual intervention |
| `FAGLGVTR` (Balance carryforward) | No | Irreversible — requires SAP support |

### Logical Unit of Work (LUW)

SAP's transactional model uses the Logical Unit of Work (LUW). All BAPI calls within a single LUW are committed together via `BAPI_TRANSACTION_COMMIT` or rolled back via `BAPI_TRANSACTION_ROLLBACK`. Agents must respect LUW boundaries — spanning a rollback across multiple LUWs is not possible and will leave partial postings in place.

---

## Failure Classification Framework

Before defining recovery strategies, classify every agent failure into one of four categories:

### Class 1 — Pre-Execution Failure
The agent failed before any SAP action was taken. No ledger impact.

*Examples:* Data retrieval timeout, permission check failure, approval gate timeout before posting, network error before BAPI call.

*Recovery:* Safe to retry from the last checkpoint. No rollback required.

---

### Class 2 — Mid-Execution Failure (Reversible)
The agent completed some SAP actions but not all. The completed actions are reversible via `BAPI_TRANSACTION_ROLLBACK` or document reversal.

*Examples:* BAPI call succeeded for company code 1000 but timed out for company code 2000. Agent posted accrual but failed before posting the offsetting entry.

*Recovery:* Roll back completed actions within the LUW if possible. If LUW is closed, reverse posted documents. Restart from the checkpoint preceding the failed action.

---

### Class 3 — Mid-Execution Failure (Irreversible)
The agent completed actions that cannot be reversed — either because they triggered downstream processes or because the reversal window has passed.

*Examples:* Payment run (`F110`) partially transmitted. Balance carryforward (`FAGLGVTR`) initiated. FX revaluation batch partially posted.

*Recovery:* No automated recovery. Immediate escalation to human close team. Freeze downstream agents. Document exact state for manual remediation.

---

### Class 4 — Silent Failure
The agent stopped without raising an exception. Downstream agents received no signal.

*Examples:* Agent process killed by infrastructure, network partition mid-session, out-of-memory termination.

*Recovery:* Detect via heartbeat timeout. Reconstruct state from checkpoints. Classify as Class 1, 2, or 3 and apply appropriate recovery.

---

## The Governance Design

### Step 1 — Checkpoint Design

A checkpoint is a persisted record of an agent's progress at a specific point in its workflow. Checkpoints enable recovery without re-executing completed steps.

**Checkpoint granularity rules:**
- Write a checkpoint **before** every Tier 2 or Tier 3 action (P002)
- Write a checkpoint **after** every successful SAP posting (with document number)
- Write a checkpoint at every **entity boundary** (completion of one company code before starting the next)
- Never write a checkpoint **during** an open LUW — only after commit

**Checkpoint content:**

```json
{
  "checkpoint_id": "uuid",
  "session_id": "close-cycle-2026-05-Q1-001",
  "agent_id": "gl-reconciliation-agent-001",
  "timestamp": "2026-05-19T23:47:12.334Z",
  "workflow_step": "post_accruals",
  "sequence_number": 12,
  "state": {
    "entities_completed": ["1000", "2000"],
    "entities_pending": ["3000", "4000"],
    "current_entity": null,
    "documents_posted": [
      {"company_code": "1000", "document_number": "100000001", "fiscal_year": "2026"},
      {"company_code": "2000", "document_number": "200000047", "fiscal_year": "2026"}
    ],
    "documents_pending": [],
    "approval_ids": ["uuid-approval-1", "uuid-approval-2"]
  },
  "recovery_class": null,
  "is_terminal": false
}
```

### Step 2 — Idempotency Key Design

Every SAP posting action must carry an idempotency key — a deterministic identifier that allows the system to detect whether an action has already been executed before retrying.

```
Idempotency Key Format:
{session_id}:{agent_id}:{workflow_step}:{entity}:{action_type}:{content_hash}

Example:
close-2026-05-Q1-001:gl-agent-001:post_accruals:1000:BAPI_ACC_DOCUMENT_POST:sha256(params)
```

Before every posting retry, check the idempotency store:
- **Key not found** — safe to post
- **Key found, status = success** — posting already completed, use existing document number, skip
- **Key found, status = in_progress** — previous attempt may be running or crashed, check SAP directly
- **Key found, status = failed** — previous attempt failed cleanly, safe to retry

### Step 3 — Heartbeat and Dead Agent Detection

Silent failures (Class 4) require active detection. Every agent must emit a heartbeat signal at regular intervals. The orchestrator monitors heartbeats and triggers recovery when a heartbeat is missed.

```
Heartbeat interval:    30 seconds
Missed heartbeat threshold:  3 consecutive misses (90 seconds)
Recovery trigger:      Orchestrator classifies failure and initiates recovery
```

### Step 4 — Recovery Decision Tree

When a failure is detected, the recovery coordinator applies this decision logic:

```
1. Retrieve last checkpoint for the failed agent session
2. Determine failure class:
   a. No checkpoint exists → Class 1 (pre-execution) → safe retry from start
   b. Checkpoint exists, no documents posted → Class 1 → retry from checkpoint
   c. Checkpoint exists, documents posted, LUW open → Class 2 → rollback LUW, retry from checkpoint
   d. Checkpoint exists, documents posted, LUW closed → Class 2 → reverse documents, retry from checkpoint
   e. Irreversible action in posted documents → Class 3 → freeze, escalate, await manual
   f. No heartbeat, checkpoint exists → Class 4 → reconstruct state, apply 2a-2e

3. Before retry:
   - Verify posting period still open (OB52)
   - Check idempotency store for all pending actions
   - Confirm approval gates are still valid (P002)
   - Notify close team of recovery action being taken

4. After successful recovery:
   - Write terminal checkpoint
   - Resume downstream agents
   - Log recovery event with full classification and actions taken (P003)
```

### Step 5 — Downstream Agent Freeze Protocol

When a Class 3 failure occurs — or any failure where recovery outcome is uncertain — downstream agents must be frozen immediately to prevent cascading errors.

```
Freeze trigger:   Class 3 failure OR recovery duration > 15 minutes
Freeze scope:     All agents in the same close session that depend on the failed agent's output
Freeze method:    Orchestrator sets session state to FROZEN in shared state store
Freeze duration:  Until human close team explicitly unfreezes
Downstream behavior: Frozen agents poll session state; do not proceed until ACTIVE
```

---

## Reference Implementation

### Python: Checkpoint Manager, Idempotency Store, and Recovery Coordinator

```python
import hashlib
import json
import uuid
import logging
from dataclasses import dataclass, field, asdict
from datetime import datetime, timezone
from typing import Optional, Dict, Any, List
from enum import Enum

logger = logging.getLogger(__name__)


class FailureClass(Enum):
    CLASS_1_PRE_EXECUTION = "class_1_pre_execution"
    CLASS_2_REVERSIBLE = "class_2_reversible"
    CLASS_3_IRREVERSIBLE = "class_3_irreversible"
    CLASS_4_SILENT = "class_4_silent"


class RecoveryAction(Enum):
    RETRY_FROM_START = "retry_from_start"
    RETRY_FROM_CHECKPOINT = "retry_from_checkpoint"
    ROLLBACK_AND_RETRY = "rollback_and_retry"
    REVERSE_AND_RETRY = "reverse_and_retry"
    FREEZE_AND_ESCALATE = "freeze_and_escalate"


class IdempotencyStatus(Enum):
    NOT_FOUND = "not_found"
    IN_PROGRESS = "in_progress"
    SUCCESS = "success"
    FAILED = "failed"


class SessionState(Enum):
    ACTIVE = "active"
    FROZEN = "frozen"
    RECOVERING = "recovering"
    COMPLETED = "completed"
    FAILED = "failed"


@dataclass
class Checkpoint:
    """Persisted agent progress record."""
    checkpoint_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    session_id: str = ""
    agent_id: str = ""
    timestamp: str = field(
        default_factory=lambda: datetime.now(timezone.utc).isoformat()
    )
    workflow_step: str = ""
    sequence_number: int = 0
    state: Dict[str, Any] = field(default_factory=dict)
    recovery_class: Optional[str] = None
    is_terminal: bool = False

    def to_dict(self) -> Dict:
        return asdict(self)


@dataclass
class IdempotencyRecord:
    """Tracks whether a specific action has already been executed."""
    key: str
    status: IdempotencyStatus
    created_at: str = field(
        default_factory=lambda: datetime.now(timezone.utc).isoformat()
    )
    completed_at: Optional[str] = None
    sap_document_number: Optional[str] = None
    error: Optional[str] = None


class IdempotencyKey:
    """Generates deterministic idempotency keys for SAP posting actions."""

    @staticmethod
    def generate(
        session_id: str,
        agent_id: str,
        workflow_step: str,
        company_code: str,
        action_type: str,
        params: Dict[str, Any],
    ) -> str:
        content_hash = hashlib.sha256(
            json.dumps(params, sort_keys=True).encode()
        ).hexdigest()[:16]
        return (
            f"{session_id}:{agent_id}:{workflow_step}"
            f":{company_code}:{action_type}:{content_hash}"
        )


class CheckpointStore:
    """
    Persists and retrieves agent checkpoints.
    In production: backed by Azure Cosmos DB, Redis, or
    any durable key-value store with strong consistency.
    """

    def __init__(self, backend):
        self.backend = backend

    def save(self, checkpoint: Checkpoint) -> None:
        key = f"checkpoint:{checkpoint.session_id}:{checkpoint.agent_id}:{checkpoint.sequence_number}"
        self.backend.set(key, json.dumps(checkpoint.to_dict()))
        # Also update the "latest" pointer for fast retrieval
        latest_key = f"checkpoint:latest:{checkpoint.session_id}:{checkpoint.agent_id}"
        self.backend.set(latest_key, json.dumps(checkpoint.to_dict()))

    def get_latest(self, session_id: str, agent_id: str) -> Optional[Checkpoint]:
        key = f"checkpoint:latest:{session_id}:{agent_id}"
        data = self.backend.get(key)
        if not data:
            return None
        d = json.loads(data)
        return Checkpoint(**d)

    def get_all(self, session_id: str, agent_id: str) -> List[Checkpoint]:
        pattern = f"checkpoint:{session_id}:{agent_id}:*"
        keys = self.backend.scan(pattern)
        checkpoints = []
        for key in keys:
            if "latest" not in key:
                data = self.backend.get(key)
                if data:
                    checkpoints.append(Checkpoint(**json.loads(data)))
        return sorted(checkpoints, key=lambda c: c.sequence_number)


class RecoveryCoordinator:
    """
    Detects agent failures, classifies them, and
    determines the correct recovery action.
    """

    # Transaction codes / BAPIs that are irreversible
    IRREVERSIBLE_ACTIONS = {
        "F110",           # Payment run — may have transmitted
        "FAGLGVTR",       # Balance carryforward
        "BAPI_TRANSACTION_COMMIT",  # After commit with no rollback path
    }

    def __init__(
        self,
        checkpoint_store: CheckpointStore,
        idempotency_store,
        sap_client,
        notification_service,
        audit_logger,
    ):
        self.checkpoint_store = checkpoint_store
        self.idempotency_store = idempotency_store
        self.sap_client = sap_client
        self.notification_service = notification_service
        self.audit_logger = audit_logger

    def handle_failure(
        self,
        session_id: str,
        agent_id: str,
        failure_type: str,
        error: Optional[Exception] = None,
    ) -> RecoveryAction:
        """
        Main entry point for failure handling.
        Returns the recovery action to take.
        """
        logger.error(
            "AGENT_FAILURE_DETECTED",
            extra={
                "session_id": session_id,
                "agent_id": agent_id,
                "failure_type": failure_type,
                "error": str(error) if error else None,
            }
        )

        # Retrieve last checkpoint
        checkpoint = self.checkpoint_store.get_latest(session_id, agent_id)

        # Classify the failure
        failure_class = self._classify(checkpoint, failure_type)

        # Determine recovery action
        action = self._determine_action(failure_class, checkpoint)

        # Log the recovery decision
        self.audit_logger.log_action(
            action_type="failure_recovery",
            action_name=action.value,
            parameters={
                "failure_class": failure_class.value,
                "checkpoint_id": checkpoint.checkpoint_id if checkpoint else None,
                "failure_type": failure_type,
            },
            outcome_status="recovery_initiated",
            risk_tier="approve",
            permission_check_passed=True,
            duration_ms=0,
        )

        # For Class 3 — freeze immediately and escalate
        if failure_class == FailureClass.CLASS_3_IRREVERSIBLE:
            self._freeze_session(session_id)
            self.notification_service.escalate_to_close_team(
                session_id=session_id,
                agent_id=agent_id,
                failure_class=failure_class,
                checkpoint=checkpoint,
                message=(
                    f"CRITICAL: Agent {agent_id} failed with irreversible actions. "
                    f"Session {session_id} is FROZEN. Manual intervention required."
                )
            )

        return action

    def _classify(
        self,
        checkpoint: Optional[Checkpoint],
        failure_type: str,
    ) -> FailureClass:
        """Classify failure based on checkpoint state."""

        # Silent failure — no exception, just stopped
        if failure_type == "heartbeat_timeout":
            return FailureClass.CLASS_4_SILENT

        # No checkpoint — nothing was executed
        if checkpoint is None:
            return FailureClass.CLASS_1_PRE_EXECUTION

        # No documents posted yet
        posted = checkpoint.state.get("documents_posted", [])
        if not posted:
            return FailureClass.CLASS_1_PRE_EXECUTION

        # Check if any irreversible actions were taken
        actions_taken = checkpoint.state.get("actions_taken", [])
        for action in actions_taken:
            if action.get("action_type") in self.IRREVERSIBLE_ACTIONS:
                return FailureClass.CLASS_3_IRREVERSIBLE

        # Documents posted but reversible
        return FailureClass.CLASS_2_REVERSIBLE

    def _determine_action(
        self,
        failure_class: FailureClass,
        checkpoint: Optional[Checkpoint],
    ) -> RecoveryAction:
        """Map failure class to recovery action."""
        if failure_class == FailureClass.CLASS_1_PRE_EXECUTION:
            if checkpoint is None:
                return RecoveryAction.RETRY_FROM_START
            return RecoveryAction.RETRY_FROM_CHECKPOINT

        if failure_class == FailureClass.CLASS_2_REVERSIBLE:
            # Check if LUW is still open (within same SAP session)
            # If LUW open: rollback. If closed: reverse documents.
            luw_open = checkpoint.state.get("luw_open", False)
            if luw_open:
                return RecoveryAction.ROLLBACK_AND_RETRY
            return RecoveryAction.REVERSE_AND_RETRY

        if failure_class in (
            FailureClass.CLASS_3_IRREVERSIBLE,
            FailureClass.CLASS_4_SILENT,
        ):
            return RecoveryAction.FREEZE_AND_ESCALATE

        return RecoveryAction.FREEZE_AND_ESCALATE  # Default safe

    def _freeze_session(self, session_id: str) -> None:
        """Freeze all agents in the session."""
        logger.critical(
            "SESSION_FROZEN",
            extra={"session_id": session_id}
        )
        # Set session state to FROZEN in shared state store
        # All agents poll this state before proceeding


class AgentCheckpointer:
    """
    Helper class used by agents to write checkpoints
    at the correct points in their workflow.
    """

    def __init__(
        self,
        session_id: str,
        agent_id: str,
        checkpoint_store: CheckpointStore,
        idempotency_store,
    ):
        self.session_id = session_id
        self.agent_id = agent_id
        self.checkpoint_store = checkpoint_store
        self.idempotency_store = idempotency_store
        self._sequence = 0
        self._state: Dict[str, Any] = {
            "entities_completed": [],
            "entities_pending": [],
            "current_entity": None,
            "documents_posted": [],
            "actions_taken": [],
            "approval_ids": [],
            "luw_open": False,
        }

    def before_action(self, workflow_step: str, action_type: str) -> Checkpoint:
        """Write checkpoint before a Tier 2/3 action."""
        self._state["luw_open"] = True
        return self._write(workflow_step, action_type + "_pre")

    def after_action(
        self,
        workflow_step: str,
        action_type: str,
        company_code: str,
        document_number: Optional[str] = None,
        idempotency_key: Optional[str] = None,
    ) -> Checkpoint:
        """Write checkpoint after a successful SAP posting."""
        self._state["luw_open"] = False
        if document_number:
            self._state["documents_posted"].append({
                "company_code": company_code,
                "document_number": document_number,
                "action_type": action_type,
                "timestamp": datetime.now(timezone.utc).isoformat(),
            })
        self._state["actions_taken"].append({
            "action_type": action_type,
            "company_code": company_code,
            "idempotency_key": idempotency_key,
            "completed_at": datetime.now(timezone.utc).isoformat(),
        })
        return self._write(workflow_step, action_type + "_post")

    def entity_completed(self, workflow_step: str, entity: str) -> Checkpoint:
        """Write checkpoint at entity boundary."""
        if entity in self._state["entities_pending"]:
            self._state["entities_pending"].remove(entity)
        self._state["entities_completed"].append(entity)
        self._state["current_entity"] = None
        return self._write(workflow_step, "entity_complete")

    def check_idempotency(self, key: str) -> IdempotencyStatus:
        """Check whether an action has already been executed."""
        record = self.idempotency_store.get(key)
        if record is None:
            return IdempotencyStatus.NOT_FOUND
        return record.status

    def _write(self, workflow_step: str, event: str) -> Checkpoint:
        self._sequence += 1
        checkpoint = Checkpoint(
            session_id=self.session_id,
            agent_id=self.agent_id,
            workflow_step=f"{workflow_step}:{event}",
            sequence_number=self._sequence,
            state=dict(self._state),
        )
        self.checkpoint_store.save(checkpoint)
        return checkpoint


# --- Usage Example ---

def post_period_close_accruals(
    checkpointer: AgentCheckpointer,
    recovery_coordinator: RecoveryCoordinator,
    sap_client,
    entities: List[str],
    session_id: str,
    agent_id: str,
):
    """
    Example: posting accruals across multiple entities
    with checkpointing and idempotency protection.
    """
    checkpointer._state["entities_pending"] = list(entities)

    for entity in entities:
        checkpointer._state["current_entity"] = entity

        # Generate idempotency key for this posting
        idem_key = IdempotencyKey.generate(
            session_id=session_id,
            agent_id=agent_id,
            workflow_step="post_accruals",
            company_code=entity,
            action_type="BAPI_ACC_DOCUMENT_POST",
            params={"company_code": entity, "period": "04", "year": "2026"},
        )

        # Check idempotency before posting
        idem_status = checkpointer.check_idempotency(idem_key)

        if idem_status == IdempotencyStatus.SUCCESS:
            logger.info(f"Skipping {entity} — already posted (idempotency check)")
            checkpointer.entity_completed("post_accruals", entity)
            continue

        # Checkpoint before action
        checkpointer.before_action("post_accruals", "BAPI_ACC_DOCUMENT_POST")

        try:
            result = sap_client.call_bapi(
                "BAPI_ACC_DOCUMENT_POST",
                parameters={"company_code": entity},
            )
            doc_number = result["document_number"]

            # Checkpoint after successful posting
            checkpointer.after_action(
                workflow_step="post_accruals",
                action_type="BAPI_ACC_DOCUMENT_POST",
                company_code=entity,
                document_number=doc_number,
                idempotency_key=idem_key,
            )
            checkpointer.entity_completed("post_accruals", entity)

        except Exception as e:
            # Classify and handle the failure
            recovery_action = recovery_coordinator.handle_failure(
                session_id=session_id,
                agent_id=agent_id,
                failure_type="execution_error",
                error=e,
            )

            if recovery_action == RecoveryAction.FREEZE_AND_ESCALATE:
                raise  # Let orchestrator handle

            # For retry actions — raise to trigger retry from checkpoint
            raise
```

---

## Audit Considerations

**1. "How do you know which documents posted before the agent failed?"**
The checkpoint written after each successful posting contains the SAP document number. The audit log (P003) contains the same information with the full action context. Cross-reference by session ID.

**2. "How do you prevent duplicate postings during recovery?"**
The idempotency store is checked before every posting action during recovery. If the idempotency key is found with status SUCCESS, the posting is skipped and the existing document number is used.

**3. "What happened to the agents that were waiting on the failed agent's output?"**
Show the downstream agent freeze log — every agent that was frozen, when it was frozen, and the session state record that triggered the freeze. Frozen agents produce no output and take no actions.

**4. "How was the failed session recovered?"**
Show the recovery coordinator log: failure detection timestamp, failure classification, recovery action selected, checkpoint used as recovery point, and the re-execution log from that point forward.

**5. "Was human approval required for the recovery actions?"**
Yes — any recovery action that involves posting, reversing, or re-executing a Tier 3 action requires a new approval cycle (P002). The approval record for the recovery action is in the audit log.

---

## Common Mistakes

| Mistake | Why It's Dangerous | Correct Approach |
|---|---|---|
| Retry without idempotency check | Duplicate journal entries posted | Always check idempotency store before every retry |
| Auto-approving recovery actions | Recovery bypasses the approval gate | Recovery postings go through the same approval gate as original postings |
| Treating all failures as Class 1 | Irreversible actions get retried — creating double postings or compounding errors | Classify every failure before choosing recovery strategy |
| No downstream freeze on Class 3 | Downstream agents proceed with incomplete data — cascade failure | Freeze all dependent agents immediately on Class 3 detection |
| Checkpoint after LUW open | If agent fails between checkpoint and commit, checkpoint shows success that didn't happen | Only checkpoint after `BAPI_TRANSACTION_COMMIT` confirms |
| No heartbeat monitoring | Silent failures go undetected for hours | Heartbeat every 30 seconds; miss threshold 90 seconds |
| Single checkpoint store | Checkpoint store failure = no recovery path | Primary + secondary checkpoint store; local fallback |
| Recovering without notifying close team | Human close team doesn't know recovery happened | Always notify close team of any Class 2, 3, or 4 failure and its resolution |

---

## Related Patterns

- **P001** — Agent Permission Scoping Against SAP Authorization Objects
- **P002** — Human Approval Gates for High-Risk SAP Transaction Codes
- **P003** — Audit Logging for Agentic Workflows in SAP

---

## Changelog

| Version | Date | Change |
|---------|------|--------|
| 1.0.0 | 2026-05 | Initial release |

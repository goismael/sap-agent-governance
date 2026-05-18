# P001 — Agent Permission Scoping Against SAP Authorization Objects

**Category:** Identity & Access  
**Risk Level:** Critical  
**SAP Version:** S/4HANA 2020+  
**Applies To:** Financial Close, Accounts Payable, General Ledger, Intercompany

---

## The Problem

When an AI agent connects to SAP S/4HANA, it typically authenticates as a service user. Without deliberate scoping, that service user may inherit broad authorization profiles — often copied from a functional user or an admin account "just to get the integration working."

The result: an agent that can read, write, post, reverse, and configure — across all company codes, all fiscal periods, and all document types — when it only needs to read a trial balance.

This is not a theoretical risk. It is the default outcome when no one explicitly designs the agent's authorization scope. And in SAP, over-permissioned service accounts are one of the most common audit findings in financial close environments.

The challenge is that SAP's authorization model was designed for human users operating through transaction codes. Agents don't use transaction codes in the traditional sense — they call OData services, BAPIs, or RFC function modules directly. This creates a mismatch: the usual PFCG role-building process assumes a menu of transactions, but agents bypass the menu entirely.

This pattern addresses how to scope agent permissions correctly against SAP authorization objects — regardless of how the agent accesses the system.

---

## SAP Authorization Concepts Required

Before applying this pattern, ensure you understand these foundational SAP concepts:

### Authorization Objects
The core unit of SAP access control. An authorization object defines a set of fields that together control access to a specific area of SAP functionality. Example: `F_BKPF_BUK` controls access to accounting documents by company code.

Key fields in `F_BKPF_BUK`:
- `BUKRS` — Company code
- `ACTVT` — Activity (01=Create, 02=Change, 03=Display, 06=Delete, 08=Display Change Documents)

### PFCG Roles
Authorization objects are grouped into roles via transaction `PFCG`. Each role generates an authorization profile that is assigned to a user (or service user). Agents authenticate as SAP service users and must be assigned PFCG roles just like human users.

### SU24 — Authorization Object Defaults
Transaction `SU24` maps transaction codes and OData services to their required authorization objects. When an agent calls an OData service, the authorization checks triggered are defined here. Understanding SU24 mappings for the OData services your agent uses is essential for correct scoping.

### Activity Codes (ACTVT)
Most authorization objects include an `ACTVT` field. The most important values:
| Code | Meaning |
|------|---------|
| 01 | Create |
| 02 | Change |
| 03 | Display / Read |
| 06 | Delete |
| 08 | Display change documents |
| 70 | Issue (used in some FI objects) |

**For read-only agents, ACTVT=03 only. Never assign 01, 02, or 06 unless the agent's function explicitly requires it.**

---

## Agent Role Classification

Before scoping permissions, classify each agent by its function. Every agent in your system must fit exactly one classification:

| Classification | What the Agent Does | Permitted Activities |
|----------------|--------------------|--------------------|
| **Reader** | Extracts data, reads documents, queries balances | ACTVT=03 (Display) only |
| **Analyzer** | Reads data, performs calculations, produces reports | ACTVT=03 (Display) only |
| **Recommender** | Reads data, produces structured recommendations for human review | ACTVT=03 (Display) only |
| **Executor** | Posts documents, reverses entries, applies corrections | ACTVT=01, 02 — scoped strictly by document type and company code |
| **Orchestrator** | Coordinates other agents, manages workflow state | No direct SAP access; communicates via internal agent APIs only |

> ⚠️ **Critical Rule:** An agent that is classified as Reader, Analyzer, or Recommender must never be assigned an Executor's authorization profile. Classification determines the authorization scope — not convenience, not "just in case," not "we might need it later."

---

## The Governance Design

### Step 1 — Map Agent Functions to OData Services and BAPIs

For each agent, document every SAP touchpoint:

```
Agent: Trial Balance Reader
SAP Touchpoints:
  - OData: /sap/opu/odata/sap/API_GLACCOUNT_DOCUMENT_SRV
  - BAPI: BAPI_GL_GETGLACCBALANCE
  - RFC: RFC_READ_TABLE (restricted to BSEG, BKPF, SKA1)

Agent: Journal Entry Poster
SAP Touchpoints:
  - BAPI: BAPI_ACC_DOCUMENT_POST
  - OData: /sap/opu/odata/sap/API_JOURNALENTRYITEMBASIC_SRV
```

### Step 2 — Identify Required Authorization Objects Per Touchpoint

Use SU24 to identify which authorization objects are checked for each OData service or BAPI your agent uses. For financial close agents, the most common objects are:

| Authorization Object | Controls | Relevant For |
|---------------------|---------|-------------|
| `F_BKPF_BUK` | Accounting documents by company code | All FI agents |
| `F_BKPF_BLA` | Accounting documents by document type | Executor agents |
| `F_BKPF_GSB` | Accounting documents by business area | Multi-entity agents |
| `F_BKPF_KOA` | Accounting documents by account type | GL/AP/AR agents |
| `F_SKA1_BUK` | GL account master by company code | Reader agents |
| `F_LFA1_BUK` | Vendor master by company code | AP agents |
| `F_BSEG_GL4` | GL account assignment in document items | Executor agents |
| `K_CCA` | Cost center accounting | Controlling agents |
| `S_RFC` | RFC access control | All agents using RFC |
| `S_SERVICE` | OData service access | All agents using OData |

### Step 3 — Build Dedicated PFCG Roles Per Agent Classification

Do not share authorization profiles across agents of different classifications. Create one PFCG role per agent classification, scoped to the minimum required authorization objects and organizational values.

**Naming Convention:**
```
Z_AGENT_[CLASSIFICATION]_[DOMAIN]_[VERSION]

Examples:
Z_AGENT_READER_FI_GL_V1     — GL Reader Agent
Z_AGENT_READER_FI_AP_V1     — AP Reader Agent  
Z_AGENT_EXEC_FI_POST_V1     — Financial Posting Executor Agent
Z_AGENT_ORCH_FINCLOSE_V1    — Financial Close Orchestrator
```

### Step 4 — Scope Organizational Values Explicitly

Every authorization object that supports organizational fields must have those fields scoped explicitly — never set to `*` (all values) for executor agents.

```
F_BKPF_BUK for Z_AGENT_EXEC_FI_POST_V1:
  BUKRS: 1000, 2000          ← Specific company codes only
  ACTVT: 01                  ← Create only (not 02=Change, not 06=Delete)

F_BKPF_BLA for Z_AGENT_EXEC_FI_POST_V1:
  BLART: SA, AB              ← G/L account document, Accounting document only
  ACTVT: 01                  ← Create only

Never:
  BUKRS: *                   ← All company codes — prohibited for executor agents
  ACTVT: *                   ← All activities — prohibited for any agent
```

### Step 5 — Assign Roles to Dedicated Service Users

Each agent must authenticate to SAP using a dedicated service user — never a named user account, never a shared integration user.

```
Service User Naming:
SVC_AGENT_[CLASSIFICATION]_[DOMAIN]

Examples:
SVC_AGENT_READER_GL         — Assigned Z_AGENT_READER_FI_GL_V1
SVC_AGENT_READER_AP         — Assigned Z_AGENT_READER_FI_AP_V1
SVC_AGENT_EXEC_POST         — Assigned Z_AGENT_EXEC_FI_POST_V1
```

Service user configuration in SU01:
- User type: **System** (type S) — not Dialog
- Password: Managed by secrets manager (Azure Key Vault, AWS Secrets Manager), rotated on schedule
- Validity period: Set to project duration — never open-ended
- Lock status: Locked by default; unlocked only during active agent sessions where technically required

---

## Reference Implementation

### Python: Permission Scope Validator

This utility validates that an agent's runtime behavior stays within its declared permission scope before executing any SAP call.

```python
from dataclasses import dataclass
from typing import Set, Dict, Optional
from enum import Enum
import logging

logger = logging.getLogger(__name__)


class AgentClassification(Enum):
    READER = "reader"
    ANALYZER = "analyzer"
    RECOMMENDER = "recommender"
    EXECUTOR = "executor"
    ORCHESTRATOR = "orchestrator"


class SAPActivity(Enum):
    CREATE = "01"
    CHANGE = "02"
    DISPLAY = "03"
    DELETE = "06"
    DISPLAY_CHANGES = "08"


# Permitted activities per classification — enforced at runtime
PERMITTED_ACTIVITIES: Dict[AgentClassification, Set[SAPActivity]] = {
    AgentClassification.READER: {SAPActivity.DISPLAY},
    AgentClassification.ANALYZER: {SAPActivity.DISPLAY},
    AgentClassification.RECOMMENDER: {SAPActivity.DISPLAY},
    AgentClassification.EXECUTOR: {SAPActivity.CREATE},  # Scoped further by document type
    AgentClassification.ORCHESTRATOR: set(),  # No direct SAP access
}


@dataclass
class AgentPermissionScope:
    """
    Declares the permission scope for an agent instance.
    Must be defined at agent initialization — not at runtime.
    """
    agent_id: str
    classification: AgentClassification
    permitted_company_codes: Set[str]
    permitted_document_types: Optional[Set[str]]  # Required for EXECUTOR agents
    permitted_odata_services: Set[str]
    permitted_bapis: Set[str]
    permitted_rfc_modules: Set[str]


class AgentPermissionGuard:
    """
    Runtime enforcement of agent permission scope.
    Call check_permission() before every SAP interaction.
    Raises PermissionError on violation — never silently allows.
    """

    def __init__(self, scope: AgentPermissionScope):
        self.scope = scope
        self._violations: list = []

    def check_odata_call(self, service: str, entity: str, activity: SAPActivity) -> None:
        """Validate an OData service call before execution."""
        violations = []

        if service not in self.scope.permitted_odata_services:
            violations.append(
                f"OData service '{service}' not in permitted scope for agent "
                f"'{self.scope.agent_id}'"
            )

        permitted = PERMITTED_ACTIVITIES[self.scope.classification]
        if activity not in permitted:
            violations.append(
                f"Activity '{activity.value}' not permitted for "
                f"{self.scope.classification.value} agent '{self.scope.agent_id}'"
            )

        if violations:
            self._log_and_raise(violations)

    def check_bapi_call(
        self,
        bapi: str,
        company_code: str,
        document_type: Optional[str] = None
    ) -> None:
        """Validate a BAPI call before execution."""
        violations = []

        if bapi not in self.scope.permitted_bapis:
            violations.append(
                f"BAPI '{bapi}' not in permitted scope for agent '{self.scope.agent_id}'"
            )

        if company_code not in self.scope.permitted_company_codes:
            violations.append(
                f"Company code '{company_code}' not in permitted scope for agent "
                f"'{self.scope.agent_id}'"
            )

        if (
            self.scope.classification == AgentClassification.EXECUTOR
            and document_type is not None
            and self.scope.permitted_document_types is not None
            and document_type not in self.scope.permitted_document_types
        ):
            violations.append(
                f"Document type '{document_type}' not in permitted scope for executor "
                f"agent '{self.scope.agent_id}'"
            )

        if violations:
            self._log_and_raise(violations)

    def check_rfc_call(self, module: str) -> None:
        """Validate an RFC call before execution."""
        if module not in self.scope.permitted_rfc_modules:
            self._log_and_raise([
                f"RFC module '{module}' not in permitted scope for agent "
                f"'{self.scope.agent_id}'"
            ])

    def _log_and_raise(self, violations: list) -> None:
        """Log all violations to audit trail, then raise."""
        for v in violations:
            logger.error(
                "AGENT_PERMISSION_VIOLATION",
                extra={
                    "agent_id": self.scope.agent_id,
                    "classification": self.scope.classification.value,
                    "violation": v,
                }
            )
            self._violations.append(v)
        raise PermissionError(
            f"Agent permission scope violation for '{self.scope.agent_id}': "
            + "; ".join(violations)
        )

    @property
    def violation_count(self) -> int:
        return len(self._violations)


# --- Usage Example ---

def example_usage():
    # Define scope at agent initialization
    gl_reader_scope = AgentPermissionScope(
        agent_id="gl-reader-agent-001",
        classification=AgentClassification.READER,
        permitted_company_codes={"1000", "2000"},
        permitted_document_types=None,  # Readers don't post — no doc types needed
        permitted_odata_services={
            "/sap/opu/odata/sap/API_GLACCOUNT_DOCUMENT_SRV",
            "/sap/opu/odata/sap/API_JOURNALENTRYITEMBASIC_SRV",
        },
        permitted_bapis={
            "BAPI_GL_GETGLACCBALANCE",
        },
        permitted_rfc_modules={
            "RFC_READ_TABLE",
        },
    )

    guard = AgentPermissionGuard(gl_reader_scope)

    # This passes — display activity, permitted service, permitted company code
    guard.check_odata_call(
        service="/sap/opu/odata/sap/API_GLACCOUNT_DOCUMENT_SRV",
        entity="GLAccountLineItem",
        activity=SAPActivity.DISPLAY,
    )

    # This raises PermissionError — reader agent cannot create
    try:
        guard.check_odata_call(
            service="/sap/opu/odata/sap/API_GLACCOUNT_DOCUMENT_SRV",
            entity="GLAccountLineItem",
            activity=SAPActivity.CREATE,
        )
    except PermissionError as e:
        print(f"Correctly blocked: {e}")
```

---

## Audit Considerations

When an internal or external auditor reviews your agentic AI implementation, expect these questions:

**1. "How do you know the agent only has access to what it needs?"**
Answer with: the PFCG role documentation for each service user, the SU24 mapping for each OData service/BAPI used, and the runtime permission guard logs showing no scope violations.

**2. "Can an agent post a document without human approval?"**
This is answered by P002 (Approval Gates), but the foundation is here: executor agents have posting authority only after runtime permission checks confirm the request is within declared scope — and approval gates are called before the BAPI is invoked.

**3. "What happens if someone changes the service user's role assignment?"**
Document your change management process for service user role assignments. Changes to `SVC_AGENT_*` users must go through SAP change control (ideally locked in a transport), not manual SU01 edits in production.

**4. "How do you detect if an agent exceeds its permissions?"**
The runtime permission guard raises and logs every violation. Those logs must feed into your SIEM or monitoring platform. Alert on any `AGENT_PERMISSION_VIOLATION` log event.

---

## Common Mistakes

| Mistake | Why It's Dangerous | Correct Approach |
|---------|-------------------|-----------------|
| Copying a functional user's role to the service user | Functional users accumulate roles over time; they always have more than needed | Build dedicated roles from scratch using SU24 mappings |
| Setting `BUKRS: *` for executor agents | Agent can post to any company code, including those it should never touch | Explicit company code list only |
| Using a Dialog user (type A) for agent authentication | Dialog users can log into SAP GUI; exposes the session to misuse | System user (type S) only |
| Shared service user across multiple agents | A single compromise exposes all agents; audit trail becomes untraceble | One service user per agent |
| Never rotating service user passwords | Credential exposure goes undetected | Secrets manager with scheduled rotation |
| Assigning `ACTVT: *` "temporarily" | Temporary always becomes permanent | ACTVT list must be explicit and reviewed quarterly |

---

## Related Patterns

- **P002** — Human Approval Gates for High-Risk SAP Transaction Codes
- **P003** — Audit Logging for Agentic Workflows in SAP
- **P004** — Agent Failure Handling During Financial Period-End Close

---

## Changelog

| Version | Date | Change |
|---------|------|--------|
| 1.0.0 | 2026-05 | Initial release |

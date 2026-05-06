# Commit Points, Idempotency, and Retry-Safe Side Effects in Orchestrated Agent Workflows

*A deep technical guide for engineers building HubSpot booking integrations and similar orchestrated systems.*

---

## Table of Contents

1. [Introduction](#introduction)
2. [Core Concepts](#core-concepts)
   - [Commit Points](#commit-points)
   - [Idempotency and Idempotency Keys](#idempotency-and-idempotency-keys)
   - [Retry-Safe Side Effects](#retry-safe-side-effects)
3. [The Booking Workflow: A Concrete Example](#the-booking-workflow-a-concrete-example)
4. [Failure Scenarios and Retry Behavior](#failure-scenarios-and-retry-behavior)
5. [REST vs MCP Mode: Where Responsibility Lives](#rest-vs-mcp-mode-where-responsibility-lives)
6. [Architectural Patterns](#architectural-patterns)
   - [Saga Pattern](#saga-pattern)
   - [Transactional Outbox](#transactional-outbox)
   - [Durable Execution](#durable-execution)
7. [Practical Implementation](#practical-implementation)
8. [Summary](#summary)

---

## Introduction

Modern agent-orchestrated systems, like the `conversion_engine` described here, coordinate sequences of side-effecting operations across external APIs. These aren't pure computations—they write to HubSpot, fire webhooks, create records. When networks fail or agents crash mid-execution, the question isn't *if* a retry will happen, but *what state* the external world will be in when it does.

This article answers the central engineering question:

> **Where exactly should we define the commit point in the agent execution flow so that all side effects are idempotent and retry-safe, and how should this design differ between REST mode and MCP mode?**

We'll build up from first principles, walk through concrete code, and map everything to established patterns.

---

## Core Concepts

### Commit Points

A **commit point** is the moment in a workflow where the system decides: *"I have done enough that this operation is considered complete."* Before the commit point, all work is tentative—a failure means we start over cleanly. After the commit point, a failure means we need to *continue*, not *restart*.

Think of it like a database transaction's `COMMIT` statement. Before `COMMIT`, a crash rolls everything back. After `COMMIT`, the data is durable—a crash doesn't undo it, and the next action picks up from that guaranteed state.

In an agent workflow, you rarely have a single atomic commit. Instead, you have a **sequence of commit points**—one per side-effecting operation—forming a chain:

```
[pending] → COMMIT₁ → [contact_upserted] → COMMIT₂ → [note_written] → COMMIT₃ → [booking_linked]
```

Each commit point is a checkpoint. After passing it, retries of subsequent steps don't need to re-execute earlier ones.

**Why commit points matter in agents:**

- Agent frameworks (LangChain, AutoGen, custom orchestrators) often retry tool calls automatically.
- LLM-driven orchestrators may re-plan and re-issue operations after a perceived failure.
- Network timeouts are ambiguous: the server may have *succeeded* even if the client never got the response.

Without explicit commit points and idempotency, every ambiguous timeout risks a duplicate write.

---

### Idempotency and Idempotency Keys

An operation is **idempotent** if applying it multiple times produces the same result as applying it once. `GET` requests are naturally idempotent. `PUT` to set a field to a fixed value is idempotent. `POST` to create a new record is *not* idempotent by default—it creates a duplicate every time.

An **idempotency key** is a unique token attached to an operation that lets the server (or your system) recognize: *"I've already processed this exact request."* On the second call with the same key, the server returns the original result without re-executing the side effect.

```
First call:  POST /contacts  {idempotency_key: "booking-42-upsert"} → 201 Created {id: "hs_001"}
Second call: POST /contacts  {idempotency_key: "booking-42-upsert"} → 200 OK {id: "hs_001"}  ← same result, no duplicate
```

**Idempotency keys should be:**
- Deterministic: derived from the operation's input data, not random.
- Scoped: tied to a specific booking, attempt, and operation type.
- Stable: the same key must be generated on retry.

A good structure: `{booking_id}:{operation_type}:{content_hash}`

Example: `booking-42:contact-upsert:a3f9c1`

---

### Retry-Safe Side Effects

A **side effect** is any operation that changes state outside your system: writing to HubSpot, sending an email, charging a card. A **retry-safe side effect** is one where retrying—even if the first attempt partially succeeded—leaves the world in exactly the correct final state.

Making side effects retry-safe requires combining:

1. **Idempotency** at the operation level (same key → same result)
2. **State tracking** in your orchestrator (don't retry what you've already committed)
3. **Compensating actions** for operations that can't be made idempotent (rollback logic)

The critical insight: **retry-safety is a property of the system, not just the API call.** Even if HubSpot's API isn't idempotent for a given endpoint, your orchestrator can make the workflow retry-safe by tracking what's been committed and skipping already-completed steps.

---

## The Booking Workflow: A Concrete Example

Our target workflow:

```
booking → contact_upsert → note_append → booking_linkage
```

Each step is a side-effecting API call. Let's map the full state machine:

```
┌─────────────┐
│   PENDING   │  ← workflow starts
└──────┬──────┘
       │ attempt contact upsert
       ▼
┌─────────────────────┐
│  CONTACT_COMMITTED  │  ← COMMIT POINT 1: contact_id stored durably
└──────────┬──────────┘
           │ attempt note creation
           ▼
┌──────────────────────┐
│   NOTE_COMMITTED     │  ← COMMIT POINT 2: note_id stored durably
└──────────┬───────────┘
           │ attempt booking linkage
           ▼
┌───────────────────────┐
│  BOOKING_COMMITTED    │  ← COMMIT POINT 3: linkage confirmed
└──────────┬────────────┘
           │
           ▼
┌──────────────────┐
│    COMPLETED     │
└──────────────────┘
```

Each commit point is reached only after **durably recording** the operation's result (e.g., the HubSpot contact ID written to your database or state store). The commit is not the API call—it's the local write that acknowledges the API call succeeded.

---

## Failure Scenarios and Retry Behavior

### Failure Before Commit Point 1 (Contact Upsert)

```
booking → [contact upsert FAILS] → retry

State: PENDING
Action: Retry contact upsert with same idempotency key
Result: Clean retry — no partial state exists
```

Because we haven't committed anything, a retry re-executes the full operation. With a proper idempotency key, if the API call actually succeeded but the response was lost, the retry returns the original result.

### Failure Between Commit Points 1 and 2

```
booking → contact upsert ✓ → COMMIT₁ stored → [note append FAILS] → retry

State: CONTACT_COMMITTED (contact_id = "hs_001")
Action: Skip contact upsert (already committed), retry note append only
Result: No duplicate contact; note append retried safely
```

This is where state tracking pays off. The orchestrator reads `CONTACT_COMMITTED`, skips step 1, and retries from step 2.

### Failure After All Commits

```
booking → contact ✓ → COMMIT₁ → note ✓ → COMMIT₂ → linkage ✓ → COMMIT₃ → [final write FAILS]

State: BOOKING_COMMITTED
Action: Mark as COMPLETED on retry
Result: All side effects already done; retry just updates local status
```

### The Ambiguous Timeout Problem

The most dangerous failure mode: your client times out, but you don't know if the server succeeded.

```
Client:  POST /contacts  → ⏱️ TIMEOUT (no response received)
Server:  ✓ Contact created {id: "hs_001"}  (client never saw this)

Without idempotency key:
  Client retries → POST /contacts → ✓ Creates SECOND contact {id: "hs_002"} ← DUPLICATE!

With idempotency key:
  Client retries → POST /contacts + key "booking-42-contact-upsert"
  Server: "I already processed this key" → returns {id: "hs_001"} ← SAFE
```

---

## REST vs MCP Mode: Where Responsibility Lives

This is the core architectural question for `conversion_engine`. The two modes shift idempotency responsibility to different layers.

### REST Mode

In REST mode, your orchestrator calls HubSpot's API directly. **The orchestrator owns idempotency** end-to-end:

```
Orchestrator
  ├── Generates idempotency keys
  ├── Attaches keys to HTTP headers (Idempotency-Key: ...)
  ├── Tracks operation state (PENDING → COMMITTED)
  ├── Implements retry logic with backoff
  └── Reads committed state to skip completed steps
```

**Responsibilities:**

| Concern | Lives in |
|---|---|
| Idempotency key generation | Orchestrator |
| Key attachment to requests | Orchestrator (HTTP header) |
| State persistence | Orchestrator (DB/Redis) |
| Retry logic | Orchestrator |
| Duplicate detection | Server-side (HubSpot honors the key) |

### MCP Mode

In MCP mode, the orchestrator calls **tools** (functions) that execute the actual API calls. The tool layer sits between the orchestrator and HubSpot. **Idempotency responsibility is split:**

```
Orchestrator
  ├── Generates idempotency keys and passes them as tool arguments
  ├── Tracks operation state (PENDING → COMMITTED)
  └── Retries tool calls on failure

Tool Layer (MCP)
  ├── Receives idempotency key from orchestrator
  ├── Attaches key to underlying API call
  ├── May implement local deduplication cache
  └── Returns deterministic result for same key
```

**Key difference:** In MCP mode, the tool interface must explicitly accept and thread idempotency keys. If a tool doesn't support idempotency keys in its interface, the MCP layer must implement its own deduplication before calling the underlying API.

| Concern | REST Mode | MCP Mode |
|---|---|---|
| Idempotency key generation | Orchestrator | Orchestrator |
| Key attachment | Orchestrator → HTTP | Orchestrator → Tool arg → HTTP |
| Local dedup cache | Optional | Often necessary |
| State persistence | Orchestrator | Orchestrator (tools are stateless) |
| Retry logic | Orchestrator | Orchestrator (tools don't retry) |

**MCP tools should be stateless and idempotent by contract.** The orchestrator must not assume a tool call "remembers" anything—each call should be self-contained.

---

## Architectural Patterns

### Saga Pattern

The **Saga pattern** models a long-running workflow as a sequence of local transactions, each with a corresponding **compensating transaction** that undoes its effect if a later step fails.

For our booking workflow:

```
Forward transactions:
  T1: upsert_contact()       → compensate: delete_contact()
  T2: append_note()          → compensate: delete_note()
  T3: link_booking()         → compensate: unlink_booking()

Saga execution:
  T1 ✓ → T2 ✓ → T3 ✗ (fails)
  
  Compensate in reverse:
  C2: delete_note() ✓ → C1: delete_contact() ✓
  
  System returns to pre-saga state.
```

Two saga variants:

- **Choreography**: Each step publishes events; the next step listens. Decoupled but hard to observe.
- **Orchestration**: A central coordinator (your agent) explicitly calls each step and manages compensation. Easier to reason about—and what `conversion_engine` already does.

**Important:** Compensation is not rollback. External systems have already been modified. Compensation is a *forward* action that reverses the effect. This means compensating transactions must also be idempotent and retry-safe.

---

### Transactional Outbox

The **transactional outbox** pattern solves a fundamental problem: how do you atomically record "I'm about to call this API" and then reliably make the call?

The naive approach fails:

```
# WRONG - not atomic:
result = call_hubspot_api()   # ← crash here means we don't know if it succeeded
db.save(result)               # ← this never runs
```

The outbox approach:

```
# Step 1: Write intent to outbox (atomic with your local DB transaction)
db.transaction():
  outbox.insert({
    id: "op-42-contact-upsert",
    status: "PENDING",
    payload: {...},
    idempotency_key: "booking-42-contact-upsert"
  })

# Step 2: Outbox relay reads pending entries and executes them
# (separate process/thread, runs continuously)
for entry in outbox.get_pending():
  result = call_hubspot_api(entry.payload, key=entry.idempotency_key)
  outbox.mark_committed(entry.id, result)
```

The outbox relay guarantees at-least-once delivery. The idempotency key guarantees exactly-once effect.

---

### Durable Execution

**Durable execution** engines (Temporal, Durable Task Framework, AWS Step Functions) provide native support for:

- Persisting workflow state at each step
- Automatically replaying from the last committed state after a crash
- Managing retries, timeouts, and compensation

In these systems, your workflow code looks like normal sequential code, but the runtime handles all the persistence and replay. The "commit point" is implicit—it happens at each `await` or `yield` in the workflow.

For `conversion_engine` without a full durable execution engine, you can implement a lightweight version using a state store (Redis, Postgres) and an explicit state machine.

---

## Practical Implementation

### Operation State Schema

```python
from enum import Enum
from dataclasses import dataclass, field
from typing import Optional, Dict, Any
import hashlib
import json
import time

class OperationStatus(Enum):
    PENDING = "pending"
    IN_FLIGHT = "in_flight"
    COMMITTED = "committed"
    FAILED = "failed"
    COMPENSATED = "compensated"

@dataclass
class OperationRecord:
    """Tracks a single side-effecting operation in the workflow."""
    operation_id: str          # Unique, deterministic ID
    booking_id: str
    operation_type: str        # "contact_upsert", "note_append", "booking_linkage"
    idempotency_key: str       # Sent to external API
    status: OperationStatus
    result: Optional[Dict[str, Any]] = None   # API response stored on commit
    error: Optional[str] = None
    attempts: int = 0
    created_at: float = field(default_factory=time.time)
    committed_at: Optional[float] = None

def make_idempotency_key(booking_id: str, operation_type: str, payload: dict) -> str:
    """
    Generate a deterministic idempotency key from operation inputs.
    The same inputs always produce the same key — critical for retries.
    """
    content = json.dumps({
        "booking_id": booking_id,
        "operation_type": operation_type,
        "payload": payload
    }, sort_keys=True)
    content_hash = hashlib.sha256(content.encode()).hexdigest()[:8]
    return f"{booking_id}:{operation_type}:{content_hash}"
```

### State Store

```python
import sqlite3
import json
from typing import Optional

class WorkflowStateStore:
    """
    Lightweight durable state store for operation records.
    In production, replace with Redis or Postgres.
    """

    def __init__(self, db_path: str = ":memory:"):
        self.conn = sqlite3.connect(db_path, check_same_thread=False)
        self._init_schema()

    def _init_schema(self):
        self.conn.execute("""
            CREATE TABLE IF NOT EXISTS operations (
                operation_id TEXT PRIMARY KEY,
                booking_id TEXT NOT NULL,
                operation_type TEXT NOT NULL,
                idempotency_key TEXT UNIQUE NOT NULL,
                status TEXT NOT NULL,
                result TEXT,
                error TEXT,
                attempts INTEGER DEFAULT 0,
                created_at REAL,
                committed_at REAL
            )
        """)
        self.conn.commit()

    def upsert(self, record: OperationRecord):
        self.conn.execute("""
            INSERT INTO operations VALUES (?,?,?,?,?,?,?,?,?,?)
            ON CONFLICT(operation_id) DO UPDATE SET
              status=excluded.status,
              result=excluded.result,
              error=excluded.error,
              attempts=excluded.attempts,
              committed_at=excluded.committed_at
        """, (
            record.operation_id, record.booking_id, record.operation_type,
            record.idempotency_key, record.status.value,
            json.dumps(record.result) if record.result else None,
            record.error, record.attempts, record.created_at, record.committed_at
        ))
        self.conn.commit()

    def get(self, operation_id: str) -> Optional[OperationRecord]:
        row = self.conn.execute(
            "SELECT * FROM operations WHERE operation_id = ?", (operation_id,)
        ).fetchone()
        if not row:
            return None
        return OperationRecord(
            operation_id=row[0], booking_id=row[1], operation_type=row[2],
            idempotency_key=row[3], status=OperationStatus(row[4]),
            result=json.loads(row[5]) if row[5] else None,
            error=row[6], attempts=row[7], created_at=row[8], committed_at=row[9]
        )

    def get_by_key(self, idempotency_key: str) -> Optional[OperationRecord]:
        row = self.conn.execute(
            "SELECT * FROM operations WHERE idempotency_key = ?", (idempotency_key,)
        ).fetchone()
        if not row:
            return None
        return self.get(row[0])
```

### REST Mode: Idempotent HubSpot Client

```python
import requests
import time
import logging
from typing import Dict, Any, Optional

logger = logging.getLogger(__name__)

class IdempotentHubSpotClient:
    """
    HubSpot REST client that attaches idempotency keys and retries safely.
    In REST mode, the orchestrator owns idempotency end-to-end.
    """

    BASE_URL = "https://api.hubapi.com"

    def __init__(self, api_key: str, max_retries: int = 3, backoff_base: float = 1.0):
        self.api_key = api_key
        self.max_retries = max_retries
        self.backoff_base = backoff_base

    def _headers(self, idempotency_key: Optional[str] = None) -> Dict[str, str]:
        headers = {
            "Authorization": f"Bearer {self.api_key}",
            "Content-Type": "application/json"
        }
        if idempotency_key:
            headers["Idempotency-Key"] = idempotency_key
        return headers

    def _call_with_retry(
        self,
        method: str,
        path: str,
        payload: Dict[str, Any],
        idempotency_key: str
    ) -> Dict[str, Any]:
        """
        Execute an API call with retry logic.
        The idempotency key makes retries safe — the server deduplicates.
        """
        url = f"{self.BASE_URL}{path}"
        last_error = None

        for attempt in range(self.max_retries):
            try:
                response = requests.request(
                    method, url,
                    json=payload,
                    headers=self._headers(idempotency_key),
                    timeout=10
                )

                # 2xx = success (including idempotent replays)
                if response.status_code in (200, 201):
                    return response.json()

                # 4xx client errors are not retryable (except 429)
                if response.status_code == 429:
                    retry_after = int(response.headers.get("Retry-After", 5))
                    logger.warning(f"Rate limited. Waiting {retry_after}s.")
                    time.sleep(retry_after)
                    continue

                if 400 <= response.status_code < 500:
                    raise ValueError(
                        f"Client error {response.status_code}: {response.text}"
                    )

                # 5xx server errors are retryable
                last_error = Exception(
                    f"Server error {response.status_code}: {response.text}"
                )

            except requests.exceptions.Timeout:
                # Timeout is ambiguous! Server may have succeeded.
                # The idempotency key handles this — retry safely.
                last_error = Exception("Request timed out")
                logger.warning(
                    f"Timeout on attempt {attempt + 1}. "
                    f"Retrying with same idempotency key: {idempotency_key}"
                )

            except requests.exceptions.ConnectionError as e:
                last_error = e

            # Exponential backoff
            wait = self.backoff_base * (2 ** attempt)
            logger.info(f"Retrying in {wait:.1f}s (attempt {attempt + 1}/{self.max_retries})")
            time.sleep(wait)

        raise Exception(
            f"All {self.max_retries} attempts failed. Last error: {last_error}"
        )

    def upsert_contact(self, email: str, properties: Dict, idempotency_key: str) -> Dict:
        return self._call_with_retry(
            "POST",
            "/crm/v3/objects/contacts/upsert",
            {"properties": {"email": email, **properties}},
            idempotency_key
        )

    def append_note(self, contact_id: str, body: str, idempotency_key: str) -> Dict:
        return self._call_with_retry(
            "POST",
            "/crm/v3/objects/notes",
            {
                "properties": {"hs_note_body": body},
                "associations": [{"to": {"id": contact_id}, "types": [{"category": "HUBSPOT_DEFINED", "typeId": 202}]}]
            },
            idempotency_key
        )

    def link_booking(self, contact_id: str, booking_id: str, idempotency_key: str) -> Dict:
        return self._call_with_retry(
            "POST",
            f"/crm/v3/objects/contacts/{contact_id}/associations/booking/{booking_id}",
            {},
            idempotency_key
        )
```

### MCP Mode: Tool Wrapper with Deduplication

```python
from typing import Callable, Any

class IdempotentMCPToolWrapper:
    """
    Wraps MCP tool calls to add local deduplication.
    In MCP mode, tools are stateless — the orchestrator passes idempotency keys
    as arguments, and this wrapper prevents re-execution for committed keys.
    """

    def __init__(self, state_store: WorkflowStateStore):
        self.store = state_store

    def execute_tool(
        self,
        tool_fn: Callable,
        operation_id: str,
        booking_id: str,
        operation_type: str,
        payload: Dict[str, Any]
    ) -> Dict[str, Any]:
        """
        Execute a tool call idempotently.
        Returns cached result if already committed.
        """
        idempotency_key = make_idempotency_key(booking_id, operation_type, payload)

        # Check if already committed — skip if so
        existing = self.store.get_by_key(idempotency_key)
        if existing and existing.status == OperationStatus.COMMITTED:
            logger.info(
                f"[MCP] Operation {operation_type} already committed. "
                f"Returning cached result."
            )
            return existing.result

        # Create or update operation record
        record = existing or OperationRecord(
            operation_id=operation_id,
            booking_id=booking_id,
            operation_type=operation_type,
            idempotency_key=idempotency_key,
            status=OperationStatus.PENDING
        )
        record.status = OperationStatus.IN_FLIGHT
        record.attempts += 1
        self.store.upsert(record)

        try:
            # Pass idempotency key into the tool as an argument
            result = tool_fn(**payload, idempotency_key=idempotency_key)

            # COMMIT: durably record the result before returning
            record.status = OperationStatus.COMMITTED
            record.result = result
            record.committed_at = time.time()
            self.store.upsert(record)

            logger.info(f"[MCP] Committed {operation_type} for booking {booking_id}")
            return result

        except Exception as e:
            record.status = OperationStatus.FAILED
            record.error = str(e)
            self.store.upsert(record)
            raise
```

### The Main Orchestrator

```python
class BookingOrchestrator:
    """
    Orchestrates the full booking workflow with commit-point tracking.
    Supports both REST and MCP backends.
    """

    def __init__(
        self,
        state_store: WorkflowStateStore,
        mode: str = "REST",
        hubspot_client: Optional[IdempotentHubSpotClient] = None,
        mcp_wrapper: Optional[IdempotentMCPToolWrapper] = None
    ):
        self.store = state_store
        self.mode = mode
        self.hubspot = hubspot_client
        self.mcp = mcp_wrapper

    def _make_op_id(self, booking_id: str, operation_type: str) -> str:
        return f"{booking_id}:{operation_type}"

    def execute_booking_workflow(
        self,
        booking_id: str,
        contact_email: str,
        contact_props: Dict,
        note_body: str
    ) -> Dict[str, Any]:
        """
        Execute the full booking → contact upsert → note → linkage workflow.
        Idempotent: safe to call multiple times for the same booking_id.
        """
        logger.info(f"Starting booking workflow for {booking_id} in {self.mode} mode")
        results = {}

        # ── STEP 1: Contact Upsert ──────────────────────────────────────────
        contact_op_id = self._make_op_id(booking_id, "contact_upsert")
        contact_record = self.store.get(contact_op_id)

        if contact_record and contact_record.status == OperationStatus.COMMITTED:
            logger.info(f"[SKIP] Contact already committed: {contact_record.result}")
            results["contact"] = contact_record.result
        else:
            results["contact"] = self._execute_contact_upsert(
                booking_id, contact_op_id, contact_email, contact_props
            )

        contact_id = results["contact"]["id"]

        # ── COMMIT POINT 1 REACHED ─────────────────────────────────────────
        logger.info(f"[COMMIT 1] Contact committed: {contact_id}")

        # ── STEP 2: Note Append ────────────────────────────────────────────
        note_op_id = self._make_op_id(booking_id, "note_append")
        note_record = self.store.get(note_op_id)

        if note_record and note_record.status == OperationStatus.COMMITTED:
            logger.info(f"[SKIP] Note already committed: {note_record.result}")
            results["note"] = note_record.result
        else:
            results["note"] = self._execute_note_append(
                booking_id, note_op_id, contact_id, note_body
            )

        note_id = results["note"]["id"]

        # ── COMMIT POINT 2 REACHED ─────────────────────────────────────────
        logger.info(f"[COMMIT 2] Note committed: {note_id}")

        # ── STEP 3: Booking Linkage ────────────────────────────────────────
        link_op_id = self._make_op_id(booking_id, "booking_linkage")
        link_record = self.store.get(link_op_id)

        if link_record and link_record.status == OperationStatus.COMMITTED:
            logger.info(f"[SKIP] Linkage already committed")
            results["linkage"] = link_record.result
        else:
            results["linkage"] = self._execute_booking_linkage(
                booking_id, link_op_id, contact_id
            )

        # ── COMMIT POINT 3 REACHED ─────────────────────────────────────────
        logger.info(f"[COMMIT 3] Booking fully linked. Workflow complete.")
        return results

    def _execute_contact_upsert(
        self, booking_id, op_id, email, props
    ) -> Dict:
        payload = {"email": email, "properties": props}
        idem_key = make_idempotency_key(booking_id, "contact_upsert", payload)

        record = OperationRecord(
            operation_id=op_id, booking_id=booking_id,
            operation_type="contact_upsert", idempotency_key=idem_key,
            status=OperationStatus.IN_FLIGHT
        )
        self.store.upsert(record)

        if self.mode == "REST":
            result = self.hubspot.upsert_contact(email, props, idem_key)
        else:  # MCP
            result = self.mcp.execute_tool(
                tool_fn=mcp_tool_upsert_contact,
                operation_id=op_id, booking_id=booking_id,
                operation_type="contact_upsert", payload=payload
            )

        # Commit: write result durably BEFORE returning
        record.status = OperationStatus.COMMITTED
        record.result = result
        record.committed_at = time.time()
        self.store.upsert(record)
        return result

    def _execute_note_append(
        self, booking_id, op_id, contact_id, body
    ) -> Dict:
        payload = {"contact_id": contact_id, "body": body}
        idem_key = make_idempotency_key(booking_id, "note_append", payload)

        record = OperationRecord(
            operation_id=op_id, booking_id=booking_id,
            operation_type="note_append", idempotency_key=idem_key,
            status=OperationStatus.IN_FLIGHT
        )
        self.store.upsert(record)

        if self.mode == "REST":
            result = self.hubspot.append_note(contact_id, body, idem_key)
        else:
            result = self.mcp.execute_tool(
                tool_fn=mcp_tool_append_note,
                operation_id=op_id, booking_id=booking_id,
                operation_type="note_append", payload=payload
            )

        record.status = OperationStatus.COMMITTED
        record.result = result
        record.committed_at = time.time()
        self.store.upsert(record)
        return result

    def _execute_booking_linkage(
        self, booking_id, op_id, contact_id
    ) -> Dict:
        payload = {"contact_id": contact_id, "booking_id": booking_id}
        idem_key = make_idempotency_key(booking_id, "booking_linkage", payload)

        record = OperationRecord(
            operation_id=op_id, booking_id=booking_id,
            operation_type="booking_linkage", idempotency_key=idem_key,
            status=OperationStatus.IN_FLIGHT
        )
        self.store.upsert(record)

        if self.mode == "REST":
            result = self.hubspot.link_booking(contact_id, booking_id, idem_key)
        else:
            result = self.mcp.execute_tool(
                tool_fn=mcp_tool_link_booking,
                operation_id=op_id, booking_id=booking_id,
                operation_type="booking_linkage", payload=payload
            )

        record.status = OperationStatus.COMMITTED
        record.result = result
        record.committed_at = time.time()
        self.store.upsert(record)
        return result
```

### Saga Compensation (Rollback on Failure)

```python
class SagaCompensator:
    """
    Implements compensating transactions for the booking saga.
    Called when a step fails and we need to undo committed steps.
    """

    def __init__(
        self,
        state_store: WorkflowStateStore,
        hubspot_client: IdempotentHubSpotClient
    ):
        self.store = state_store
        self.hubspot = hubspot_client

    def compensate_booking(self, booking_id: str):
        """
        Undo committed operations in reverse order.
        Each compensation is itself idempotent.
        """
        logger.warning(f"Starting compensation for booking {booking_id}")

        # Compensate in REVERSE order
        operations_to_compensate = [
            "booking_linkage",
            "note_append",
            "contact_upsert"
        ]

        for op_type in operations_to_compensate:
            op_id = f"{booking_id}:{op_type}"
            record = self.store.get(op_id)

            if record and record.status == OperationStatus.COMMITTED:
                comp_key = make_idempotency_key(
                    booking_id, f"compensate_{op_type}", record.result or {}
                )
                self._compensate_operation(op_type, record, comp_key)
                record.status = OperationStatus.COMPENSATED
                self.store.upsert(record)
                logger.info(f"Compensated {op_type} for booking {booking_id}")

    def _compensate_operation(
        self, op_type: str, record: OperationRecord, comp_key: str
    ):
        result = record.result or {}
        if op_type == "booking_linkage":
            contact_id = result.get("contact_id")
            booking_id = record.booking_id
            # Unlink the booking association
            self.hubspot._call_with_retry(
                "DELETE",
                f"/crm/v3/objects/contacts/{contact_id}/associations/booking/{booking_id}",
                {},
                comp_key
            )
        elif op_type == "note_append":
            note_id = result.get("id")
            self.hubspot._call_with_retry(
                "DELETE", f"/crm/v3/objects/notes/{note_id}", {}, comp_key
            )
        elif op_type == "contact_upsert":
            # Be careful: only delete if we created the contact (not updated existing)
            if result.get("created", False):
                contact_id = result.get("id")
                self.hubspot._call_with_retry(
                    "DELETE",
                    f"/crm/v3/objects/contacts/{contact_id}",
                    {},
                    comp_key
                )
```

---

## Summary

Here's the complete mental model for commit points and idempotency in orchestrated workflows:

| Concept | Definition | Applied to conversion_engine |
|---|---|---|
| **Commit point** | Moment when operation result is durably recorded locally | After each HubSpot write is confirmed and stored in state DB |
| **Idempotency key** | Deterministic token that makes an operation repeatable | `{booking_id}:{op_type}:{payload_hash}` |
| **Retry-safe side effect** | Operation that produces identical outcome on re-execution | Any HubSpot call sent with a consistent idempotency key |
| **Saga** | Sequence of local transactions with compensating rollbacks | `upsert → note → link`, compensated in reverse on failure |
| **Transactional outbox** | Write intent before execution, relay confirms completion | Outbox table in DB; relay calls HubSpot asynchronously |
| **Durable execution** | Runtime that persists and replays workflow state | Temporal, or lightweight state machine in Postgres/Redis |

**The golden rule:** Your commit point is not the API call—it's the local write that confirms the API call succeeded. Everything before that write is tentative. Everything after it is durable.

In **REST mode**, the orchestrator owns the full idempotency stack: key generation, HTTP header attachment, state tracking, and retry logic.

In **MCP mode**, the orchestrator still generates and tracks keys, but passes them into tools as arguments. Tools must be stateless; local deduplication caches in the tool wrapper prevent duplicate execution when a committed key is replayed.

---

*This article was written to address real engineering tradeoffs in dual-backend agent orchestration. The code is production-oriented pseudocode and should be adapted to your specific storage backend, API client, and error handling requirements.*
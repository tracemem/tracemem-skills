---
name: decision-envelope
description: Using the decision envelope pattern for structured thinking.
---

# Skill: TraceMem Decision Envelopes

## Purpose
This skill teaches how to correctly manage the lifecycle of a Decision Envelope. The Decision Envelope is the mandatory boundary for all governed operations in TraceMem.

## Use `decision_record` Instead?

If you only need to **document or record a decision** (e.g., architecture choice, dependency selection, tradeoff) and do NOT need data product operations (reads, writes, policy evaluations, or approvals), use `decision_record` instead of the full envelope workflow. It handles everything in one call. See the **decision-recording** skill.

## When to Use
- Every time you intend to perform a task that requires reading private data or affecting system state.
- At the very beginning of a new task or workflow.
- When an existing decision has been closed and you need to perform follow-up actions (start a new decision).

## When NOT to Use
- When you only need to document/record a decision without data operations -- use `decision_record` instead.
- Do not nest decisions (e.g., do not open a decision if you are already inside one, unless explicitly starting a sub-task that requires independent audit).
- Do not open a decision for purely computational tasks (e.g., formatting text) that require no external data or side effects.

## Core Rules
- **⚠️ CRITICAL: ALL Operations MUST Be Within a Decision Envelope**: You CANNOT perform ANY of these operations without an active decision_id:
  - ❌ `decision_read` - requires decision_id
  - ❌ `decision_write` - requires decision_id  
  - ❌ `decision_evaluate` - requires decision_id
  - ❌ `decision_request_approval` - requires decision_id
  - ✅ `products_list` and `product_get` - do NOT require decision_id (discovery tools)
- **One Decision, One Lifecycle**: A decision must be explicitly created (`open`), operated upon, and then explicitly closed (`commit` or `abort`).
- **Mandatory Intents**: You must provide a structured `intent` string (e.g., `customer.onboarding.verification`) that describes *why* this decision exists.
- **Mandatory Automation Mode**: You must certify the `automation_mode` (only these values: `propose`, `approve`, `override`, `autonomous`).
- **Close Required**: A decision left open is a "zombie" decision. You *must* ensure `decision_close` is called in `finally` blocks or error handlers.

## Common Workflow Patterns

### Pattern 1: Read-Only Workflow
```
Purpose: Retrieve customer information for support

1. decision_create
   - intent: "customer.lookup.support"
   - automation_mode: "autonomous"
   → Returns: decision_id

2. product_get (optional but recommended)
   - product: "customers_v1"
   → Returns: schema and allowed_purposes

3. decision_read (REQUIRES decision_id from step 1)
   - decision_id: <from step 1>
   - product: "customers_v1"
   - purpose: "support_context"
   - query: {"customer_id": "1001"}
   → Returns: customer records

4. decision_close (REQUIRED)
   - decision_id: <from step 1>
   - action: "commit"

❌ WRONG: Calling decision_read without step 1 will FAIL
```

### Pattern 2: Insert Workflow
```
Purpose: Create a new customer order

1. decision_create
   - intent: "order.create.customer"
   - automation_mode: "autonomous"
   → Returns: decision_id

2. product_get
   - product: "orders_v1"
   → Returns: schema with required fields

3. decision_write (REQUIRES decision_id from step 1)
   - decision_id: <from step 1>
   - product: "orders_v1"
   - purpose: "order_creation"
   - operation: "insert"  ← INSERT creates NEW records
   - mutation: {
       "records": [{
         "customer_id": 1001,
         "total_amount": 99.99,
         "status": "pending"
       }]
     }
   → Returns: created order

4. decision_close (REQUIRED)
   - decision_id: <from step 1>
   - action: "commit"

❌ WRONG: Calling decision_write without step 1 will FAIL
```

### Pattern 3: Update Workflow
```
Purpose: Update order status

1. decision_create
   - intent: "order.status.update"
   - automation_mode: "approve"
   → Returns: decision_id

2. decision_read (read current state)
   - decision_id: <from step 1>
   - product: "orders_v1"
   - purpose: "order_update"
   - query: {"order_id": "12345"}
   → Returns: current order data

3. decision_evaluate (check if update allowed)
   - decision_id: <from step 1>
   - policy_id: "order_status_change_v1"
   - inputs: {"current_status": "pending", "new_status": "shipped"}
   → Returns: allowed/denied

4. decision_write (REQUIRES decision_id from step 1)
   - decision_id: <from step 1>
   - product: "orders_v1"
   - purpose: "order_update"
   - operation: "update"  ← UPDATE modifies EXISTING records
   - keys: {"order_id": "12345"}  ← NEW: Explicit key specification
   - mutation: {
       "fields": {
         "status": "shipped"    ← What to change
       }
     }

5. decision_close (REQUIRED)
   - decision_id: <from step 1>
   - action: "commit"

❌ WRONG: Any operation without decision_id will FAIL
```

### Pattern 4: Delete Workflow
```
Purpose: Delete expired draft order

1. decision_create
   - intent: "order.draft.cleanup"
   - automation_mode: "autonomous"
   → Returns: decision_id

2. decision_read (verify before delete)
   - decision_id: <from step 1>
   - product: "orders_v1"
   - purpose: "data_cleanup"
   - query: {"order_id": "67890", "status": "draft"}

3. decision_write (REQUIRES decision_id from step 1)
   - decision_id: <from step 1>
   - product: "orders_v1"
   - purpose: "data_cleanup"
   - operation: "delete"  ← DELETE removes records
   - keys: {"order_id": "67890"}  ← NEW: Explicit key specification

4. decision_close (REQUIRED)
   - decision_id: <from step 1>
   - action: "commit"
```

### Pattern 5: Approval Workflow
```
Purpose: Request approval for large discount

1. decision_create
   - intent: "discount.exception.request"
   - automation_mode: "approve"
   → Returns: decision_id

2. decision_evaluate
   - decision_id: <from step 1>
   - policy_id: "discount_cap_v1"
   - inputs: {"discount": 0.30, "customer_tier": "standard"}
   → Returns: requires_approval

3. decision_request_approval (REQUIRES decision_id from step 1)
   - decision_id: <from step 1>
   - title: "30% Discount Request"
   - message: "Customer requesting exception"
   → Returns: approval_id

4. [Wait for approval - poll decision_get]

5. decision_close (REQUIRED)
   - decision_id: <from step 1>
   - action: "commit" or "abort" (based on approval result)
```

## Correct Usage Pattern

1. **Create the Envelope**:
   Call `decision_create` with:
   - `intent`: A dot-separated string (e.g., `financial.report.generate`).
   - `automation_mode`: One of `propose`, `approve`, `override`, `autonomous`.
   - `actor`: Your agent identity.
   
   *Result*: You receive a `decision_id`.

2. **Operate within the Envelope**:
   Perform all `decision_read`, `decision_evaluate`, and `decision_write` calls using the `decision_id`.

3. **Close the Envelope**:
   - If successful: Call `decision_close` with `action: "commit"`.
   - If error/aborted: Call `decision_close` with `action: "rollback"` (or `abort` if strictly tracking abandonment).

   *Note*: Writes are only permanently applied when the decision is committed (depending on system configuration, but logically, the decision is not "done" until committed).

## Common Mistakes
- **Attempting operations without decision envelope**: Calling `decision_read`, `decision_write`, `decision_evaluate`, or `decision_request_approval` without first calling `decision_create` will FAIL. The decision envelope is MANDATORY for ALL these operations.
- **Forgetting to close**: Leaving decisions open forever consumes resources and confuses auditors.
- **Vague Intents**: Using generic intents like `task.do` or `agent.act`. Use specific, domain-relevant intents (`user.password.reset`, `invoice.payment.process`).
- **Wrong Automation Mode**: Claiming `autonomous` when the task requires human oversight, or `propose` when you intend to execute immediately. Remember: only `propose`, `approve`, `override`, or `autonomous` are valid.

## Alternative: One-Shot Decision Recording

For decisions that **do not** require data product operations (reads, writes, evaluations, approvals), use `decision_record` instead of the full envelope workflow. This is ideal for recording architecture decisions, dependency choices, schema changes, and other engineering decisions.

```
Tool: decision_record
Parameters:
  - title: "Use goroutines with worker pool pattern"
  - category: "architecture"
  - context: "The pipeline needs concurrent processing..."
  - decision: "Implement bounded worker pool using channels..."
  - rationale: "Worker pools provide backpressure naturally..."
  - alternatives_considered: [{option: "...", rejected_because: "..."}]
  - tags: ["go", "concurrency"]
Returns: Committed decision with trace and snapshot
```

**When to use `decision_record` vs full envelope:**
- `decision_record`: Pure architectural/design decisions, no data operations needed
- Full envelope (`decision_create` → operations → `decision_close`): Decisions involving data reads, writes, policy evaluations, or approval workflows

You can also search past decisions with `decision_search` before recording new ones to find precedent or decisions to supersede.

## Safety Notes
- **Audited Lifecycle**: The timestamps of open and close are recorded. Long-running open decisions may trigger alerts.
- **Fail Closed**: If your process crashes, the decision remains open (and potentially locks resources). Always wrap your workflow in a try/finally block to ensure `decision_close` is attempted.

---
name: writing-data
description: Instructions for writing and efficiently storing data in TraceMem.
---

# Skill: TraceMem Writing Data (Governed Writes)

## Purpose
This skill explains how to modify data (insert, update, delete) within TraceMem's governance model. Writing is a high-stakes operation that often triggers policy checks.

## When to Use
- When you need to create records, update status, or delete resources.
- After you have read the current state and determined a change is necessary.

## When NOT to Use
- Do not write if you are in `propose` mode (unless prototyping in a sandbox, but generally `propose` implies read-only).
- Do not write without first evaluating relevant policies (unless you are sure the Data Product has no attached blocking policies).

## Core Rules
- **Verify Policy First**: Before writing, it is best practice to call `decision_evaluate` to check if your proposed action is allowed.
- **One Operation Per Product**: Use `_insert`, `_update`, or `_delete` products appropriately. DO NOT try to `insert` on an `update` product.
- **Flat vs Nested**: You can use a flat structure (operation + fields) for function-like products (e.g. `create_incident`) or nested `records` for batch table operations.
- **Governance**: Writes are the primary target for human approvals. Be prepared for a write to be blocked.
- **Purpose**: Like reads, writes require a valid `purpose`.

## Correct Usage Pattern

1. **Check Policy (Recommended)**:
   Call `decision_evaluate` with your proposed inputs. If outcome is `deny`, stop. If `requires_exception`, request approval.

2. **Execute Write**:
   Call `decision_write` with:
   - `product`: The write-capable data product.
   - `purpose`: Valid purpose.
   - `operation`: one of `insert`, `update`, `delete` (top-level parameter).
   - `keys` (NEW, optional): For update/delete, explicitly specify which fields are keys (recommended).
   - `mutation`: Contains the data to write.
     - For batch writes: use `records` array.
     - For single record: can use `data` object (insert only) or `records` with one element.
     - For single update/delete with keys object: use `fields` object.
   
   *Example 1 (Batch/Table Write)*:
   ```json
   {
     "product": "orders_insert",
     "operation": "insert",
     "mutation": {
       "records": [{"user_id": 5, "item": "sku-123"}]
     }
   }
   ```

   *Example 2 (Single Record Insert)*:
   ```json
   {
     "product": "create_incident_v1",
     "operation": "insert",
     "mutation": {
       "data": {
         "customer_id": "1001",
         "title": "System Down",
         "severity": "high"
       }
     }
   }
   ```

   *Example 3 (Update - NEW Recommended Format)*:
   ```json
   {
     "product": "orders_update",
     "operation": "update",
     "keys": {"order_id": 123},
     "mutation": {
       "fields": {"status": "shipped"}
     }
   }
   ```
   
   *Example 3b (Update - Batch Format)*:
   ```json
   {
     "product": "orders_update",
     "operation": "update",
     "keys": ["order_id"],
     "mutation": {
       "records": [
         {"order_id": 123, "status": "shipped"},
         {"order_id": 124, "status": "delivered"}
       ]
     }
   }
   ```

   **Note**: For backward compatibility, `operation` can still be placed inside `mutation`, but this is deprecated. Always use the top-level `operation` parameter in new code.

3. **Verify Result**:
   Check the response for `status: "executed"`. If the product has `return_created: true`, capture the returned IDs.

## Common Mistakes
- **Ignoring Policy**: Attempting to write immediately without checking policy. TraceMem will block you if a policy denies it, but it's better to ask permission (`evaluate`) than forgiveness.
- **Confusing Operations**: Trying to `delete` using an `insert` product.
- **Partial Updates**: Assuming `update` behaves like a patch (merging fields). Check the Data Product definition; usually updates replace specific fields or require full records depending on configuration.

## Safety Notes
- **Idempotency**: Use `idempotency_key` if there is a risk of retrying the same write multiple times (e.g., network timeout).
- **Commit**: Remember that writes are part of the decision. If you `rollback` the decision (or fail to close it with commit), the writes may be rolled back (depending on connector implementation), but typically TraceMem connectors aim for immediate consistency within the decision scope. *Correction*: In most TraceMem connectors, writes execute immediately but are *logically* bracketed by the decision trace. Always assume "Fail Closed" means the action might have happened if the network call succeeded; the decision trace just records it as "failed" workflow. *However*, strictly governed connectors might buffer writes until commit. **Assume writes are real and immediate unless specified otherwise.**

---
name: overview
description: Overview of TraceMem core features, rules, and best practices.
---

# Skill: TraceMem Overview and Mental Model

## Purpose
This skill provides the foundational mental model for using TraceMem. It explains what TraceMem is, why an agent should use it, and how to think about its core primitives: Decision Envelopes and Decision Traces.

## When to Use
- When you are an AI agent performing actions that affect the real world (reading private data, changing state, spending money).
- When you need to prove *why* you did something to a human or auditor.
- When you encounter instructions to "use TraceMem", "open a decision", or "check policy".
- When you need to persist memory or context across different execution sessions (TraceMem is your long-term memory).

## When NOT to Use
- For scratchpad operations, intermediate chain-of-thought, or temporary variable storage that does not need audit.
- For purely internal data processing that has no external side effects and strictly public data.
- When low-latency (<10ms) direct database access is required (TraceMem adds governance overhead).

## Core Rules
- **TraceMem is the Courtroom**: You are the agent/defendant. You must present evidence (reads), cite laws (policies), and record your verdict (outcome).
- **All Actions Require a Decision**: You cannot read or write governed data without first opening a Decision Envelope.
- **You are Untrusted**: TraceMem assumes you (the agent) might hallucinate or be compromised. You must explicitly log your reasoning and prove your actions are safe.
- **Traces are Forever**: Everything you do in TraceMem is recorded in an immutable append-only trace. Do not store secrets or PII in purpose fields or context summaries.

## Quick Start Workflow

### CRITICAL: Decision Envelope is MANDATORY

**The Decision Envelope is NOT optional.** ALL TraceMem operations MUST happen within a decision envelope:
- ✓ Reading data (`decision_read`)
- ✓ Writing data (`decision_write` - insert/update/delete)
- ✓ Evaluating policies (`decision_evaluate`)
- ✓ Requesting approvals (`decision_request_approval`)

**You CANNOT perform any of these operations without an active decision_id.**

### Basic Workflow (Read Data Example)

Here's the exact sequence for reading customer data:

```
Step 1: Create Decision Envelope
Tool: decision_create
Parameters:
  - intent: "customer.lookup.support"
  - automation_mode: "autonomous"  (must be: propose, approve, override, or autonomous)
Returns: decision_id (e.g., "TMEM_abc123...")

Step 2: Get Product Metadata (Discovery - does NOT require decision envelope)
Tool: product_get
Parameters:
  - product: "customers_v1"
Returns: Product metadata including:
  - exposed_schema: field names and types
  - allowed_purposes: valid purposes for this product
  - example_queries: sample query structures
Note: This gets INFORMATION ABOUT the product, not the actual data

Step 3: Read Actual Data (REQUIRES decision_id from Step 1)
Tool: decision_read
Parameters:
  - decision_id: "TMEM_abc123..."  (from Step 1)
  - product: "customers_v1"
  - purpose: "support_context"  (must be in allowed_purposes from Step 2)
  - query: {"customer_id": "1001"}  (use field names from schema in Step 2)
Returns: Actual customer records/data from the database

Step 4: Close Decision Envelope (REQUIRED)
Tool: decision_close
Parameters:
  - decision_id: "TMEM_abc123..."  (from Step 1)
  - action: "commit"  (or "abort" if cancelled)
```

### OpenCode vs Core MCP Tools

**OpenCode Convenience Tools** (in OpenCode plugin):
- `tracemem_open` → convenience wrapper for `decision_create` with action mapping
- `tracemem_note` → convenience wrapper for `decision_add_context`

**Core MCP Tools** (always available):
- `decision_create`, `decision_read`, `decision_write`, `decision_evaluate`, `decision_request_approval`, `decision_close`
- `decision_record` -- record a complete decision in one shot (no data operations needed)
- `decision_search` -- search previous decisions by text, category, or tags
- `products_list`, `product_get`, `capabilities_get`

**Important**: Even when using OpenCode convenience tools, the same rules apply - all operations must be within a decision envelope.

## Correct Usage Pattern

1. **Start with a Decision**: Before doing any work, you MUST open a "Decision Envelope" using `decision_create`. This is your mandatory workspace container.
2. **Discover Data Products**: Use `products_list` to find available data products and `product_get` to get metadata about a specific product (schema, purposes). These discovery tools do NOT require a decision envelope.
3. **Access Actual Data**: Use `decision_read` to read actual data records or `decision_write` to insert/update/delete data. You do not use SQL or API calls directly. ALL data access requires the decision_id from step 1.
4. **Check Policies**: You check if your intended action is allowed by evaluating "Policies" using `decision_evaluate` (requires decision_id).
5. **Respect Approvals**: If a policy requires human approval, you pause and wait using `decision_request_approval` (requires decision_id). You never bypass denials.
6. **Close the Decision**: When finished, you MUST call `decision_close` with "commit" (if successful) or "abort" (if stopped/cancelled).

## Common Mistakes
- **Treating TraceMem as a database**: It is not a generic database; it is a governance log with attached storage.
- **Skipping the Decision Envelope**: Trying to read/write data without an active `decision_id` will FAIL. The decision envelope is MANDATORY.
- **Thinking you have authority**: You (the agent) never authorize actions. You *request* them. Policies and humans authorize them.
- **Attempting operations outside decision envelope**: Operations like `decision_read`, `decision_write`, `decision_evaluate`, and `decision_request_approval` REQUIRE an active decision_id. You CANNOT call them without first calling `decision_create`.

## Safety Notes
- **Immutable Record**: Never put sensitive credentials (API keys, passwords) into any TraceMem field (intent, purpose, context). These records are often readable by broad audiences for audit.
- **Governance Layer**: Do not try to bypass TraceMem to "save time". All bypassed actions are unauditable and effectively "rogue" behavior.

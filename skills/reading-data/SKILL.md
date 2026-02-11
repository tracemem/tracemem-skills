---
name: reading-data
description: Instructions for reading data from TraceMem memory.
---

# Skill: TraceMem Reading Data (Data Products)

## Purpose
This skill teaches how to safely and correctly read data using TraceMem Data Products. Direct database selection or API calls are prohibited; all data access must be mediated by TraceMem.

## When to Use
- When you need to retrieve information (database rows, API results, resource state).
- When you need to search or query for specific records.

## When NOT to Use
- Do not use for writing data (use `decision_write` / `insert` product).
- Do not try to use standard SQL libraries or HTTP clients for governed data.

## Core Rules
- **Decision Envelope REQUIRED**: You MUST have an active `decision_id` from `decision_create` before calling `decision_read`. Reading data without a decision envelope will FAIL.
- **Data Products Only**: You must access data via a named "Data Product".
- **Operation Specificity**: A Data Product usually supports only ONE operation. Use a "read" product (e.g., `customers_read`) for reading.
- **Purpose is Mandatory**: You must declare a `purpose` string for every read. This purpose must be in the product's `allowed_purposes` list.
- **Least Privilege**: Only request the data you strictly need.

## Before You Read - MANDATORY Checklist

Follow these steps IN ORDER before attempting to read data:

### ✓ Step 1: Create Decision Envelope (REQUIRED)
```
Tool: decision_create
Why: ALL TraceMem operations require a decision envelope
Parameters:
  - intent: "customer.lookup.support"
  - automation_mode: "autonomous" (or propose/approve/override)
Returns: decision_id → Save this for steps 3-5
```

### ✓ Step 2: List Available Products (Discovery)
```
Tool: products_list
Why: Find which data products exist and are accessible
Parameters:
  - purpose: "support_context" (optional filter)
Returns: List of products with names and descriptions
Note: This returns a CATALOG of available products, not the data itself
```

### ✓ Step 3: Get Product Metadata (Discovery)
```
Tool: product_get
Why: Get metadata about a specific product - its schema and rules
Parameters:
  - product: "customers_v1"
Returns:
  - exposed_schema: [{name: "customer_id", type: "integer"}, ...]
  - allowed_purposes: ["support_context", "order_validation", ...]
  - example_queries: Sample query structures
Note: This returns METADATA about the product (schema, rules)
      This does NOT return actual data records
      This does NOT require a decision_id
```

### ✓ Step 4: Construct Query Using Schema Fields
```
From Step 3, you learned:
  - Field names: customer_id, name, email, tier
  - Allowed purposes: support_context, order_validation

Build query object:
  query: {"customer_id": "1001"}  ← Use exact field names from schema
```

### ✓ Step 5: Read Actual Data (REQUIRES decision_id from Step 1)
```
Tool: decision_read
Why: Access the ACTUAL DATA RECORDS from the product
Parameters:
  - decision_id: "TMEM_abc123..." (from Step 1 - REQUIRED)
  - product: "customers_v1"
  - purpose: "support_context" (must be in allowed_purposes from Step 3)
  - query: {"customer_id": "1001"} (using field names from Step 3)
  - allow_multiple: false (default) or true
Returns: records array with ACTUAL DATA from database
  Example: [{customer_id: 1001, name: "Acme Corp", email: "contact@acme.com"}]
```

### ✓ Step 6: Close Decision Envelope (REQUIRED)
```
Tool: decision_close
Parameters:
  - decision_id: "TMEM_abc123..." (from Step 1)
  - action: "commit" (successful) or "abort" (cancelled)
```

## Correct Usage Pattern

1. **Identify the Product**:
   Use `products_list` to find the correct Data Product for your need (e.g., `orders_read_v1`). Check its schema and `allowed_purposes`.

2. **Execute Read**:
   Call `decision_read` with:
   - `decision_id`: Your current open decision.
   - `product`: The name of the data product.
   - `purpose`: A valid purpose string (e.g., `verification`).
   - `query`: The key-value filter (e.g., `{"user_id": "123"}`).

3. **Handle Results**:
   The response contains `records` (list) and `summary`.
   
   *Example*:
   ```json
   {
     "records": [{"id": 1, "status": "active"}],
     "summary": {"rows": 1}
   }
   ```

## Common Mistakes
- **Missing decision envelope**: Calling `decision_read` without first calling `decision_create`. This will FAIL - the decision envelope is MANDATORY.
- **Implicit Purpose**: Failing to provide a purpose or guessing one not in the allowed list.
- **Wrong Product**: Trying to read from a product designed for `insert`.
- **Unbounded Queries**: Querying without filters (TraceMem may limit result sets by default).
- **Skipping product_get**: Not checking the schema and allowed_purposes before reading, leading to errors about invalid field names or purposes.

## Safety Notes
- **Audit Trails**: Your query parameters and the summary of what you read are recorded in the immutable trace.
- **Sensitive Data**: Avoid putting PII in the *query* parameters if possible, unless the Data Product is explicitly designed for PII lookups.
- **Read-Before-Write**: Always read the current state of an object before attempting to modify it, to ensure your decision is based on fresh data.

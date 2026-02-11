---
name: traces-and-audit
description: Auditing memory traces and debugging.
---

# Skill: TraceMem Traces and Audit

## Purpose
This skill explains the concept of the Decision Trace as an artifact. Understanding this helps you write better "evidence" into the system.

## When to Use
- When you need to understand *what* TraceMem is actually recording.
- When generating reports or answering questions about past actions ("Why did you do that?").

## When NOT to Use
- You generally do not "use" this skill to execute actions, but to inform *how* you execute them.

## Core Rules
- **The Trace is the Truth**: If it's not in the trace, it didn't happen (legally/audit-wise).
- **Append-Only**: You cannot go back and fix history.
- **Complete Picture**: A trace includes your ID, the time, the policy version, the data schema version, and the exact outcomes.

## Correct Usage Pattern

1. **Design for Readability**:
   When running a decision, imagine a human reading the trace 6 months later.
   - "Why did this agent delete this user?"
   - Look at the `intent`, look at the `context` you added, look at the `policy` result.
   - If the trace answers the question, you succeeded.

2. **Linking**:
   If you chain decisions (one decision triggers another workflow), reference the parent `decision_id` in the child's `metadata` or `context`.

## Searching Past Decisions

Use `decision_search` to query your agent's previous decisions:

- **Find precedent**: Search by text, category, or tags before making a new decision
- **Check supersession chains**: Results include `supersedes` and `superseded_by` indicators -- follow the chain to find the current active decision
- **Filter by status**: Use `status: "committed"` to find only finalized decisions

```
Tool: decision_search
Parameters:
  - query: "authentication"  (free-text search)
  - category: "architecture"  (optional)
  - tags: ["jwt", "auth"]  (optional, all must match)
  - status: "committed"  (optional)
  - limit: 10  (optional, default 20, max 100)
```

This is particularly valuable for:
- Answering "Why did we decide X?" questions
- Avoiding duplicate or contradictory decisions
- Building on prior context when making related decisions

## Common Mistakes
- **Phantom Actions**: Doing side effects (like calling an external API) *without* recording it in TraceMem or via a Data Product. This creates "dark matter" — actions that have no record.
- **Incomplete Evidence**: Reading data via a side-channel (not a Data Product) and then acting on it. The trace will show the action but not the data that justified it.
- **Not searching before deciding**: Always check `decision_search` for existing decisions on the same topic before recording a new one.

## Safety Notes
- **Exoneration**: A good trace protects *you* (the agent). If a policy was wrong, the trace proves you followed the policy correctly. If data was bad, the trace proves you acted on the bad data you were given.

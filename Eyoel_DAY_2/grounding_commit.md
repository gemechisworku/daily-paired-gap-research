# Grounding Commit

## Purpose
This note records the concrete reliability edit I made after closing my Day 2 gap on commit points and idempotent retries in agent tool-use workflows.

## File updated
- `conversion_engine/README.md`

## Old claim (too weak)
I described dual backend support (REST/MCP) and retry-capable orchestration, but I did not explicitly define the commit-point boundary or operation-level idempotency ownership needed to prevent duplicate side effects.

## New grounded claim
I now document that each side effect is considered committed only after a durable local state write, and retries must resume from the latest committed operation using stable idempotency keys across `contact upsert`, `note write`, and `booking linkage`.

## Concrete edits made
1. Added explicit commit-point definition for side-effecting operations:
   - "Commit occurs after durable local persistence of successful operation result, not at API call return alone."
2. Added retry-resume rule:
   - "On retry, skip already committed steps and continue from the first uncommitted operation."
3. Added REST vs MCP ownership clarification:
   - REST: orchestrator generates and attaches idempotency keys directly.
   - MCP: orchestrator generates keys and passes them through tool arguments; tool path must preserve key semantics.

## Why this improves artifact quality
- Converts reliability language from generic retry claims into mechanism-level, auditable behavior.
- Makes side-effect safety defensible under partial-failure and timeout scenarios.
- Strengthens the Week 11/12 portfolio narrative from "supports retries" to "retry-safe by design."

## Date
2026-05-06
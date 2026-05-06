# Morning Summary

- Clarified the core engineering question: define commit points and retry-safe side effects in orchestrated agent workflows, especially for booking/HubSpot-like integrations.
- Reviewed the actual research question: how to separate scaffolding-driven planning from model-driven planning, and where planning boundaries become brittle under ambiguous user input.
- Decided the writeup should focus on where responsibility lives, how retry semantics differ between REST and MCP modes, and why commit points are checkpoints in local state rather than API call boundaries.
- Scoped the answer around multi-step booking workflows and durable execution patterns, with concrete failure cases for pre-commit, mid-commit, and post-commit retries.
- Planned to include both architecture-level guidance and practical design rules: commit points, idempotency keys, state tracking, compensating actions, and the REST vs MCP ownership split.

# Evening Summary

- Completed a technical guide explaining commit points, idempotency, and retry-safe side effects for orchestrated agent workflows.
- Connected the final explanation back to the research question by comparing scaffolding-driven planning to model-driven planning and analyzing brittle boundary points.
- Highlighted the difference between tentative planning and durable execution, using a booking workflow as a concrete example.
- Compared REST mode and MCP mode responsibilities, showing where idempotency keys must be generated and how tools should preserve retry safety.
- Emphasized that safe retries require both durable local checkpointing and idempotent external operations, not just one or the other.

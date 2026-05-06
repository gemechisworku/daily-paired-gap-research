1/5
Why commit points matter: in orchestrated booking workflows, the danger is not only failure, but whether retries duplicate external side effects.

2/5
A commit point is a local checkpoint after a side effect is durably recorded, not the API call itself. Before it, failures can restart cleanly; after it, the system must continue from the committed state.

3/5
Idempotency keys are essential for ambiguous timeouts. In REST mode, the orchestrator generates and attaches them. In MCP mode, tools must accept the same keys and return deterministic results for repeated calls.

4/5
Retry-safe side effects require both local state tracking and external uniqueness. If either is missing, retries can create duplicate contacts, duplicate bookings, or inconsistent workflow state.

5/5
The right architecture separates deterministic commit points from model-driven planning, makes every side effect retry-safe, and keeps clear ownership of responsibility in each mode.

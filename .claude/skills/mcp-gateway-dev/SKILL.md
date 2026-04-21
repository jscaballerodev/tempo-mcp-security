---
name: mcp-gateway-dev
description: Use when adding, modifying, or reviewing verification layers in the TEMPO Secure MCP Gateway (stack-hardened/gateway/). Covers the Decision dataclass pattern, FastAPI middleware ordering, audit integration, and the strict rule that no security features go in stack-naive/. Trigger whenever touching files under gateway/auth/, gateway/resource/, gateway/sql_firewall/, gateway/policy/, gateway/taint/, gateway/approval/, gateway/dlp/, or gateway/audit/.
---

# MCP Gateway Development

This skill encodes the conventions for the Secure MCP Gateway — the FastAPI service that sits between AI clients and MCP servers.

## Core pattern: every verification layer is a module with one entry point

Each defensive layer (auth, resource, sql_firewall, policy, taint, approval, dlp) lives in its own directory under `stack-hardened/gateway/` and exposes a single function:

```python
from gateway.core.decision import Decision, Verdict
from gateway.core.context import RequestContext

def verify(ctx: RequestContext) -> Decision:
    """
    Inspect the current request. Return a Decision.
    Never mutate ctx; return modifications via Decision.metadata.
    Never raise; wrap all exceptions into Decision(verdict=DENY, reason="<module>.internal_error").
    """
    ...
```

## The Decision dataclass

Every module returns this exact type. Never invent alternative return shapes.

```python
from dataclasses import dataclass, field
from enum import Enum
from typing import Any

class Verdict(Enum):
    ALLOW = "allow"
    DENY = "deny"
    REQUIRE_APPROVAL = "require_approval"
    REDACT = "redact"

@dataclass
class Decision:
    verdict: Verdict
    reason: str              # format: "<module>.<condition>", e.g. "sql_firewall.non_select"
    metadata: dict[str, Any] = field(default_factory=dict)
```

## Reason code convention

`<module_name>.<specific_condition>`. Examples that exist:

- `auth.missing_header`
- `auth.signature_invalid`
- `auth.expired`
- `resource.audience_mismatch`
- `sql_firewall.non_select`
- `sql_firewall.forbidden_table`
- `sql_firewall.forbidden_schema`
- `policy.role_sensitivity_mismatch`
- `policy.missing_scope`
- `provenance.untrusted_to_restricted`
- `risk.bulk_read`
- `dlp.redacted_pii`

When adding a new condition, add the reason code to `gateway/core/reasons.py` as a constant. The audit writer validates reason codes against this registry.

## Pipeline ordering (do not reorder casually)

From `gateway/core/pipeline.py`:

```
1. auth.verify          — is the caller who they claim?
2. resource.verify      — does the token belong on this MCP?
3. sql_firewall.verify  — is the payload syntactically safe?
4. policy.verify        — does the role + scope permit this action? (OPA)
5. provenance.verify    — is the action influenced by untrusted data? (OPA)
6. risk.verify          — is the action high-risk? (OPA)
7. approval.route       — if require_approval, enqueue and return 202
8. forward              — send to MCP server
9. dlp.filter           — scrub output
10. audit.write         — record decision (ALWAYS, even on early exit)
```

The order is security-significant: cheap checks before expensive ones, authentication before authorization, and audit writes *always* happen in a `finally` block.

## Audit: every exit path writes exactly one row

Wrap the whole pipeline in a single `finally` that writes to `audit.entries`. Silent denies are bugs. Test this by asserting request count == audit row count after every attack run.

```python
async def handle_request(req):
    ctx = RequestContext.build(req)
    decision = Decision(Verdict.DENY, "core.not_evaluated")
    try:
        decision = await run_pipeline(ctx)
        return to_http_response(decision)
    finally:
        await audit.write(ctx, decision)
```

## Testing pattern

Every module has three test categories in `tests/gateway/<module>/`:

- `test_allows_legitimate.py` — proves the module doesn't over-block.
- `test_blocks_malicious.py` — proves the module catches what it's meant to.
- `test_failure_modes.py` — network timeouts, OPA unreachable, Keycloak JWKS stale. Fail-closed in every case.

Before integrating a module into the pipeline, all three categories must pass in isolation.

## The naive stack is off-limits

Never add security features to `stack-naive/`. If code appears to belong there, stop. The naive stack's job is to be vulnerable. See `CLAUDE.md` § "Two-Stack Convention" and `THREAT_MODEL.md`.

## LLMs are not in the decision path

The gateway is deterministic. LLMs (Ollama, Claude API) live upstream in the dashboard's NL→SQL microservice. Their output is *untrusted input* to the gateway. Never put an LLM call inside `gateway/` code.

## Reference documents

- `CLAUDE.md` at repo root — architectural rules
- `THREAT_MODEL.md` — attacks the gateway must defend against
- `docs/stages/CURRENT_STAGE.md` — which layer is being built now
- `docs/stages/STAGE_3.md` through `STAGE_6.md` — per-layer specifications

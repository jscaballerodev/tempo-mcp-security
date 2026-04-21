# STAGE 4 — OPA Policy Engine

## Goal

Integrate Open Policy Agent (OPA) as the policy decision point. Migrate role-based decisions out of the SQL firewall into Rego policies. Block Attack 3 (privilege escalation) on the hardened stack.

## Why OPA Belongs Separately

Policy separation from enforcement is a foundational principle of zero-trust architecture. OPA provides a declarative language (Rego), policy-as-code, and the ability to update rules without redeploying the gateway. The proposal (section 2.3) commits to this.

## Prerequisites

- Stage 3 complete.
- Attacks 1 and 2 block on hardened, succeed on naive.

## Tasks

### 1. OPA Bundle Structure

`policies/`:

```
policies/
├── bundle.manifest
├── access/
│   ├── main.rego            # entry point
│   ├── roles.rego           # role → sensitivity tier mapping
│   ├── scopes.rego          # scope → operation mapping
│   └── tables.rego          # table → sensitivity lookup
├── risk/
│   ├── main.rego
│   └── scoring.rego         # risk score calculation
└── provenance/
    └── main.rego            # stub, populated in Stage 5
```

### 2. Core Access Policy

`policies/access/main.rego`:

```rego
package tempo.access

import future.keywords.in

default allow := false
default decision := {"verdict": "deny", "reason": "policy.default_deny"}

# Input shape (documented in policies/README.md):
# {
#   "subject": {"id": "...", "roles": ["marketing_analyst"], "scopes": ["mcp:read"]},
#   "resource": {"mcp_uri": "mcp://tempo-finance", "tool": "query_database"},
#   "action": {"type": "select", "tables": ["app.finance_transactions"]},
#   "context": {"taint": "trusted", "ts": "..."}
# }

allow if {
    role_allows_all_tables
    scope_permits_action
}

decision := {"verdict": "allow", "reason": "policy.allow"} if allow

decision := {"verdict": "deny", "reason": reason} if {
    not allow
    reason := deny_reason
}

deny_reason := "policy.role_sensitivity_mismatch" if {
    some tbl in input.action.tables
    not role_may_access(input.subject.roles, tbl)
}

deny_reason := "policy.missing_scope" if {
    not scope_permits_action
}
```

### 3. Role-to-Sensitivity Mapping

`policies/access/roles.rego`:

```rego
package tempo.access

role_sensitivity_caps := {
    "marketing_analyst": {"public", "internal"},
    "finance_analyst":   {"public", "internal", "confidential"},
    "hr_admin":          {"public", "internal", "restricted"},
    "executive":         {"public", "internal", "confidential", "restricted", "secret"},
    "platform_admin":    {"public", "internal", "confidential", "restricted", "secret"},
}

role_may_access(roles, table) if {
    some role in roles
    tier := table_tier[table]
    tier in role_sensitivity_caps[role]
}

role_allows_all_tables if {
    every t in input.action.tables {
        role_may_access(input.subject.roles, t)
    }
}
```

Table tiers are injected into OPA as **data** (not policy) via a sidecar. The gateway periodically reads `app.table_sensitivity` and pushes it to OPA's data API. This way the policy doesn't need to be redeployed when table tags change.

### 4. Risk Scoring

`policies/risk/scoring.rego`:

```rego
package tempo.risk

default score := 0

score := s if {
    s := mutating_score + bulk_score + cross_domain_score + taint_score
}

mutating_score := 30 if input.action.type in {"insert", "update", "delete"}
mutating_score := 0

bulk_score := 20 if input.action.estimated_rows > 1000
bulk_score := 10 if input.action.estimated_rows > 100
bulk_score := 0

cross_domain_score := 15 if count({d | d := input.action.tables[_].domain}) > 1
cross_domain_score := 0

taint_score := 40 if input.context.taint == "untrusted"
taint_score := 20 if input.context.taint == "mixed"
taint_score := 0

verdict := "require_approval" if score >= 40
verdict := "allow" if score < 40
verdict := "deny" if score >= 80
```

Thresholds are tuned during Stage 7 evaluation.

### 5. Gateway OPA Client

`stack-hardened/gateway/policy/`:

- `policy/client.py` — HTTP client that posts the gateway's built input document to `http://opa:8181/v1/data/tempo/access/decision` and similar for risk.
- `policy/input_builder.py` — constructs the input document from the current request's `SubjectContext`, parsed SQL (to extract tables), and taint metadata (stub now).
- `policy/enforcer.py` — orchestrates access decision then risk decision; returns a `Decision`.

The gateway caches OPA responses keyed on `(subject_id, tables_tuple, action_type)` with a short TTL (5s) to reduce latency. Never cache denies for longer — if policy changes, denies must update quickly.

### 6. Move Role Logic Out of SQL Firewall

The SQL firewall's table allowlist check from Stage 3 was a placeholder. Remove it. The firewall now only enforces *syntactic* rules (statement type, schema forbidden patterns, row limits). Semantic rules (who may access what) are entirely OPA's job.

This separation is important. It's also a common bug source — test thoroughly that both layers still run in the right order and that removing the firewall's role check doesn't reopen attack 1.

### 7. Policy Testing

`policies/tests/`:
- Use `opa test` for unit tests on Rego.
- Every role × tier combination tested.
- Edge cases: empty roles array, unknown role, missing table in `table_sensitivity` data.
- CI runs `opa test policies/` before `pytest`.

### 8. Enable Attack 3

Remove the `expected_fail_until_stage(4)` marker from `test_03_privilege_escalation.py`.

Expected result on hardened: `marketing_analyst` asking for `exec_compensation` → gateway builds input → OPA returns `{"verdict": "deny", "reason": "policy.role_sensitivity_mismatch"}` → gateway returns 403 → audit row written.

### 9. Approval Queue Scaffolding

The risk scoring policy may return `require_approval`. The approval queue UI and workflow is Stage 5, but the backend shape should be decided now:
- Create `audit.approval_queue` table with columns: `id, ts, subject, mcp_uri, input_sql, risk_score, status, decided_by, decided_at`.
- When OPA returns `require_approval`, the gateway writes to this queue and returns HTTP 202 with a `Location: /approvals/{id}` header.
- The dashboard polls this endpoint. Actual approval UI is Stage 5.

## Acceptance Criteria

- [ ] `opa test policies/ -v` passes all unit tests.
- [ ] OPA container loads the bundle at startup and exposes `/v1/data/tempo/access/decision`.
- [ ] Gateway pushes `table_sensitivity` data to OPA on startup; test by hitting `/v1/data/tempo/access/table_tier`.
- [ ] `pytest attacks/test_03_privilege_escalation.py --stack=hardened` — blocked with reason `policy.role_sensitivity_mismatch`.
- [ ] `pytest attacks/ --stack=hardened` — 3/5 blocked (Attacks 1, 2, 3); 4 and 5 still expected-fail.
- [ ] Legitimate-use suite still passes.
- [ ] Gateway latency overhead: measured in `evaluation/out/latency_stage4.json`. Expected increase over Stage 3: 2–5ms per request for OPA round-trip.
- [ ] Removing a table's tier from `app.table_sensitivity` and re-syncing to OPA causes subsequent queries against that table to be denied.

## Deliverable Commit

Tag: `stage-4-complete`
Message: `feat(gateway): OPA integration, RBAC via Rego — A03 blocked`

## What This Stage Is NOT

- Not implementing taint tracking (Stage 5).
- Not implementing the approval UI (Stage 5; backend scaffolded here).
- Not implementing DLP (Stage 6).

## Notes for Claude Code

- Rego is deceptively simple; test ALL `default` branches. Forgetting a `default` value produces `undefined` in surprising places.
- Keep policies pure. Never make network calls from within Rego. All external data (like table tiers) must be pushed in as OPA data.
- The gateway → OPA call should have a 200ms timeout. If OPA is unreachable, fail closed (deny the request). Log this distinctly.
- Use OPA's decision log feature to capture every policy evaluation. These logs feed into the evaluation report in Stage 7.
- Version the policy bundle. Every commit to `policies/` should bump the version in `bundle.manifest`. The audit row should record which policy version decided.

## Advance to Stage 5

Update `docs/stages/CURRENT_STAGE.md` to `STAGE_5`.

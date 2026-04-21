---
name: rego-policy-authoring
description: Use when writing, modifying, or testing Open Policy Agent (OPA) policies in policies/ for the TEMPO gateway. Covers the canonical input document shape, the default-deny pattern, how to use OPA data vs policy, the decision envelope convention, and policy unit testing with opa test. Trigger whenever editing any .rego file or policies/bundle.manifest.
---

# Rego Policy Authoring

All policies for the TEMPO gateway follow these conventions. Deviating from them breaks the gateway's OPA client.

## Canonical input document

Every policy receives input in this exact shape. The gateway's `policy/input_builder.py` produces it.

```json
{
  "subject": {
    "id": "user-uuid-or-sub",
    "roles": ["marketing_analyst"],
    "scopes": ["mcp:read"],
    "token_id": "jti-claim"
  },
  "resource": {
    "mcp_uri": "mcp://tempo-finance",
    "tool": "query_database"
  },
  "action": {
    "type": "select",
    "tables": ["app.finance_transactions"],
    "estimated_rows": 42,
    "mutating": false
  },
  "context": {
    "taint": "trusted",
    "taint_source": null,
    "ts": "2026-04-21T10:30:00Z"
  }
}
```

If your policy needs a new field, add it to the input_builder first, document it here, then write the policy.

## Decision envelope

Every policy module's top-level `decision` rule returns this shape:

```rego
package tempo.<domain>

default decision := {"verdict": "deny", "reason": "<domain>.default_deny"}

decision := {"verdict": "allow", "reason": "<domain>.<condition>"} if { ... }
decision := {"verdict": "deny", "reason": "<domain>.<condition>"} if { ... }
decision := {"verdict": "require_approval", "reason": "<domain>.<condition>"} if { ... }
```

The gateway calls `POST /v1/data/tempo/<domain>/decision`. No other endpoints are consumed.

## Default-deny is mandatory

Every module starts with:

```rego
default decision := {"verdict": "deny", "reason": "<domain>.default_deny"}
```

Forgetting this means `decision` can be `undefined` in edge cases, which the gateway interprets as fail-closed — but the audit row will say `policy.undefined` instead of a meaningful reason. Always set the default.

## Policy vs data — the rule

**Policy** (`.rego` files) contains *rules*. It's what changes when permissions change.

**Data** (pushed to OPA via the Data API) contains *facts*. Table sensitivity tiers, role-to-tier mappings that change frequently, allowlists.

Rule of thumb: if the gateway's own internal state changes it, it's data. If a developer changes it, it's policy. Table tiers are data because they're read from `app.table_sensitivity` at runtime. Role-to-tier caps are policy because they express authorization intent.

Push data with:

```python
client.put_document("/v1/data/tempo/access/table_tier", {
    "app.marketing_campaigns": "public",
    "app.hr_salaries": "restricted",
    ...
})
```

Consume it in Rego as:

```rego
tier := data.tempo.access.table_tier[table_name]
```

## Never make network calls from Rego

No `http.send`. No external data fetches during evaluation. If a policy needs external input, it comes in via `input` (per-request) or `data` (pushed ahead of time).

## Unit testing policies

Every `.rego` file has a sibling `_test.rego`. Test file naming: `main_test.rego` tests `main.rego`. Run with `opa test policies/ -v`.

Template:

```rego
package tempo.access_test

import data.tempo.access

test_marketing_analyst_denied_for_restricted if {
    result := access.decision with input as {
        "subject": {"roles": ["marketing_analyst"], "scopes": ["mcp:read"]},
        "action": {"tables": ["app.hr_salaries"]},
    } with data.tempo.access.table_tier as {"app.hr_salaries": "restricted"}

    result.verdict == "deny"
    result.reason == "policy.role_sensitivity_mismatch"
}

test_executive_allowed_for_secret if {
    result := access.decision with input as {
        "subject": {"roles": ["executive"], "scopes": ["mcp:read"]},
        "action": {"tables": ["app.exec_compensation"]},
    } with data.tempo.access.table_tier as {"app.exec_compensation": "secret"}

    result.verdict == "allow"
}
```

Test both sides of every rule: the condition that should allow AND the adjacent condition that should deny.

## Common traps

**Trap 1: the `some` quantifier scope.** `some x in input.action.tables` creates a local `x`. If you use it inside `count({...})` constructions, the scoping can surprise you. Test with empty arrays and single-element arrays.

**Trap 2: partial rules that both match.** If two `decision := ...` rules both evaluate true, OPA returns a set of decisions and the gateway errors. Make conditions mutually exclusive — usually by adding negations.

**Trap 3: data shape mismatch.** If the gateway pushes `{"app.foo": "public"}` but the policy reads `data.tempo.access.table_tier["foo"]` (without the schema prefix), the lookup fails silently and you get `undefined`. Normalize naming at the push step.

**Trap 4: forgetting bundle version bump.** Every policy change bumps the version in `policies/bundle.manifest`. The audit log records which version decided. Reviewers check this.

## Bundle structure

```
policies/
├── bundle.manifest                 # { "revision": "v0.5.0", ... }
├── access/
│   ├── main.rego
│   ├── main_test.rego
│   └── roles.rego
├── risk/
│   ├── main.rego
│   └── main_test.rego
└── provenance/
    ├── main.rego
    └── main_test.rego
```

## Reference documents

- `THREAT_MODEL.md` — the attacks policies must block
- `docs/stages/STAGE_4.md` — initial access + risk policies
- `docs/stages/STAGE_5.md` — provenance policy
- OPA docs: https://www.openpolicyagent.org/docs/

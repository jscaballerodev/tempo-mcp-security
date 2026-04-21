# STAGE 5 — Taint Tracking and Approval Gates

## Goal

Implement provenance-aware decisions and the human-in-the-loop approval workflow. Block Attack 4 (indirect prompt injection) and Attack 5 (bulk exfiltration) on the hardened stack.

## Why This Is the Novel Layer

Most MCP security projects stop at OAuth + RBAC. **Taint tracking** is what elevates this project from integration exercise to research contribution. The core idea: any data retrieved from a lower-trust source (documents, tickets, external pages) is tagged `untrusted`, and tool calls whose generation was influenced by untrusted content carry that taint forward. The gateway treats tainted requests to sensitive resources as higher-risk, routing them through approval.

This is a simplified, practical take on information-flow tracking — the full theory is in the literature on non-interference and lattice-based security, but a working demo of even the simplified version is significant for an advanced cybersecurity capstone.

## Prerequisites

- Stage 4 complete.
- Attacks 1, 2, 3 blocked on hardened.

## Tasks

### 1. Taint Model

Define three trust tiers:

| Tier | Source examples |
|---|---|
| `trusted` | Direct user keyboard input in the dashboard, authenticated user's explicit query |
| `mixed` | User input containing quoted/pasted content from elsewhere |
| `untrusted` | Content retrieved from documents, tickets, external MCPs, web pages |

Taint propagates: if an LLM's input contains any `untrusted` fragment, its output is tainted `untrusted`. Mixed inputs produce `mixed` output unless a trusted-only fragment is independently verifiable.

### 2. Dashboard-Side Taint Injection

The dashboard is the origin of taint labels. Modify `stack-hardened/dashboard/`:

- User types directly → `trusted`.
- User pastes text → `mixed`.
- AI client retrieves a document/ticket before composing the SQL request → `untrusted`.
- Every call to the gateway includes an HTTP header `X-Tempo-Taint: trusted|mixed|untrusted` and `X-Tempo-Taint-Source: <reference>` describing where the untrusted content came from.

The header is signed with a short-lived HMAC key shared between dashboard and gateway so a malicious client cannot downgrade taint by lying in the header. Key rotates per session.

### 3. Gateway Taint Module

`stack-hardened/gateway/taint/`:

- `taint/extractor.py` — reads and verifies the signed taint header. If missing or invalid, default to `untrusted`. Fail closed.
- `taint/propagator.py` — builds the `context.taint` field for the OPA input document.
- Unit test: malformed, missing, expired, and wrong-signature headers all produce `untrusted`.

### 4. Provenance Policy

`policies/provenance/main.rego`:

```rego
package tempo.provenance

default decision := {"verdict": "allow"}

# Untrusted calls to restricted+ resources require approval.
decision := {"verdict": "require_approval", "reason": "provenance.untrusted_to_restricted"} if {
    input.context.taint == "untrusted"
    some tbl in input.action.tables
    tier := data.tempo.access.table_tier[tbl]
    tier in {"restricted", "secret"}
}

# Mixed calls to secret resources require approval.
decision := {"verdict": "require_approval", "reason": "provenance.mixed_to_secret"} if {
    input.context.taint == "mixed"
    some tbl in input.action.tables
    data.tempo.access.table_tier[tbl] == "secret"
}

# Untrusted cross-domain calls require approval regardless of tier.
decision := {"verdict": "require_approval", "reason": "provenance.untrusted_cross_domain"} if {
    input.context.taint == "untrusted"
    count({d | d := data.tempo.access.table_domain[input.action.tables[_]]}) > 1
}
```

### 5. Gateway Pipeline Update

Insert provenance check after access check and before risk scoring:

```
1. auth
2. resource
3. sql_firewall
4. access policy (tempo.access)
5. provenance policy (tempo.provenance)  ← NEW
6. risk policy (tempo.risk)
7. approval routing (if require_approval)
8. forward to MCP
9. audit
```

If any policy returns `deny`, stop immediately. If any returns `require_approval`, skip forwarding and go straight to approval routing.

### 6. Approval Workflow Backend

`stack-hardened/gateway/approval/`:

- `approval/queue.py` — enqueue, dequeue, update status. Uses `audit.approval_queue` from Stage 4.
- `approval/webhook.py` — when status changes to `approved`, the queued request is replayed through the pipeline with a special header `X-Tempo-Approved: <approval_id>` signed by the gateway's own key.
- `approval/policy.py` — who may approve what:
  - `marketing_analyst` cannot approve anything.
  - `finance_analyst` may approve own-domain `confidential` tier only.
  - `hr_admin` may approve `restricted` in HR domain.
  - `executive` may approve `secret`.
  - Self-approval is forbidden. Requester ≠ approver.

### 7. Approval UI

`stack-hardened/dashboard/src/routes/approvals/`:

- `/approvals` — list page showing pending approvals visible to the logged-in user.
- `/approvals/[id]` — detail page showing:
  - Requester identity.
  - Time of request.
  - The original prompt and the generated SQL.
  - Taint tier and source.
  - Risk score and the rule that triggered.
  - The table(s) being accessed and their tiers.
  - **Approve** and **Deny** buttons with a mandatory free-text justification.
- Every approver action calls the gateway and produces an audit row.
- Approvals expire after 15 minutes; expired approvals are auto-denied.

### 8. Attack 4 — Prompt Injection Defense

With taint tracking in place, Attack 4 now flows:

1. AI client retrieves `support_tickets` row containing injection payload. Dashboard tags retrieved content as `untrusted`.
2. LLM produces SQL querying `hr_salaries`. Dashboard sees the SQL was generated from context containing untrusted content → tags the outgoing gateway call with `X-Tempo-Taint: untrusted`.
3. Gateway: auth passes, resource passes, SQL firewall passes (SELECT on a known table), access policy passes for `hr_admin`... wait, but the caller is `marketing_analyst`. Actually access policy fails first at step 4 with `role_sensitivity_mismatch`.

The test needs to be written so the caller has access to `hr_salaries` by role (use an `hr_analyst` role) but the query is influenced by untrusted content. This better demonstrates taint's unique value.

Adjust `THREAT_MODEL.md` Attack 4 accordingly: the actor is an `hr_analyst` who asks a benign question, but the retrieved ticket injects a query for `exec_compensation` which they may also legitimately access as hr_analyst. Taint catches it; RBAC would not. This is the cleanest demonstration.

### 9. Attack 5 — Bulk Exfiltration Defense

The risk policy's `bulk_score` already triggers `require_approval` for large reads. Wire Attack 5 to assert the 202 response and the approval queue entry. If the test simulates an approver denying the request, assert no data reaches the caller.

### 10. Remove Expected-Fail Markers

Remove `expected_fail_until_stage(5)` from `test_04_prompt_injection.py` and `test_05_bulk_exfiltration.py`.

## Acceptance Criteria

- [ ] Dashboard sends signed taint headers on all gateway calls.
- [ ] Gateway rejects requests with missing, invalid, or expired taint signatures — treats them as `untrusted`.
- [ ] `opa test policies/provenance/` passes.
- [ ] `pytest attacks/test_04_prompt_injection.py --stack=hardened` — returns 202 `require_approval` with reason `provenance.untrusted_to_restricted` or `provenance.untrusted_cross_domain`.
- [ ] `pytest attacks/test_05_bulk_exfiltration.py --stack=hardened` — returns 202 `require_approval` with reason from risk policy.
- [ ] Full attack suite: `--stack=hardened` blocks 5/5, `--stack=naive` succeeds 5/5.
- [ ] Approval UI functional: log in as `hr_admin`, see pending approval, approve with justification, watch request complete.
- [ ] Expired approvals auto-deny; test by setting the TTL to 1 second in a test fixture.
- [ ] Self-approval blocked: requester cannot approve own request.
- [ ] Audit rows for approved-then-replayed requests contain both the original request ID and the approval ID.

## Deliverable Commit

Tag: `stage-5-complete`
Message: `feat(gateway): taint tracking and approval workflow — A04 and A05 gated`

## What This Stage Is NOT

- Not implementing DLP on responses (Stage 6).
- Not cryptographically signing audit logs (Stage 6).
- Not integrating external MCPs (Stage 6).

## Notes for Claude Code

- Taint signing is the critical integrity property. Use HMAC-SHA256 with a key that rotates on dashboard login. The gateway fetches the current key from Keycloak's user session store or a small dedicated service.
- Test the taint-downgrade attack explicitly: a malicious dashboard tries to send `X-Tempo-Taint: trusted` when the actual content was untrusted. The signature check must fail.
- Approval UI must show the full SQL that will execute. Approvers need to see what they're authorizing; "approve request 47" tells them nothing.
- Keep approval decisions in the audit log forever. Approvals are legally significant records.
- The approval queue is a backdoor if misconfigured. Test extensively: cross-role approvals, stale approvals, replay of approved requests, approval of deleted requests.

## Advance to Stage 6

Update `docs/stages/CURRENT_STAGE.md` to `STAGE_6`.

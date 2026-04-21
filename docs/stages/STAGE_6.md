# STAGE 6 — Signed Audit Log, DLP, and External MCP Integration

## Goal

Harden the audit trail cryptographically, add output DLP (data loss prevention), and integrate one real external MCP server (GitHub MCP recommended) to demonstrate the gateway works across heterogeneous backends.

## Why This Stage Is Polish, Not Foundation

After Stage 5, all five attacks are defended. This stage adds the properties that make the system *credible in a real enterprise*: tamper-evident audit, protection against residual data leakage in responses, and a proof that the gateway generalizes beyond the internal TEMPO MCP.

## Prerequisites

- Stage 5 complete. All 5 attacks block or gate on hardened.

## Tasks

### 1. Append-Only Signed Audit Log

`stack-hardened/gateway/audit/`:

- `audit/signer.py` — each audit row is signed with an HMAC chain. Row N's signature covers row N's content + row N-1's signature, forming a hash chain. Any tampering with a past row invalidates all subsequent signatures.
- `audit/verifier.py` — CLI tool `tempo-audit-verify` that walks the entire chain and reports any inconsistency.
- Postgres-level: revoke `DELETE` and `UPDATE` on `audit.entries` from the gateway's DB role. Only `INSERT` is permitted. A separate admin role (used by a dedicated rotation job, not by the gateway) may archive old rows with a full chain export.
- Gateway DB role cannot read its own audit table either; this prevents a compromised gateway from selectively reading and re-signing a tampered history.

### 2. DLP Output Filter

`stack-hardened/gateway/dlp/`:

- `dlp/patterns.py` — regex patterns for common sensitive data:
  - SSN: `\b\d{3}-\d{2}-\d{4}\b`
  - US credit card (Luhn-checked): implement actual Luhn validation to reduce false positives.
  - Email outside org domain.
  - Phone numbers (E.164 and US formats).
  - API keys (patterns for AWS, GitHub, Stripe, etc.).
- `dlp/classifier.py` — lightweight named-entity recognition as a second pass. Options:
  - spaCy with a small model (`en_core_web_sm`).
  - Presidio (Microsoft's open-source PII tool) if deployment allows.
- `dlp/redactor.py` — replaces detected matches with typed placeholders: `[REDACTED:SSN]`, `[REDACTED:EMAIL]`.
- `dlp/policy.py` — policy for what to do per tier:
  - `public` output: no DLP.
  - `internal`: light DLP (only credentials and keys).
  - `confidential`: full DLP.
  - `restricted`/`secret`: DLP + audit marker noting what was redacted.

DLP runs *after* the MCP response is received and *before* it returns to the caller. A `dlp.redactions` column is added to the audit row summarizing what was redacted without storing the sensitive values themselves.

### 3. Gateway Normalization Layer (for external MCPs)

`stack-hardened/gateway/normalize/`:

Different MCP servers expose different tools. To apply consistent policy, map each external tool into an internal action shape:

```python
class NormalizedAction:
    action_type: Literal["read", "write", "list", "search", "execute"]
    resources: list[str]              # e.g., ["github://repo/issues"]
    estimated_impact: Literal["low", "medium", "high"]
    sensitivity_hint: str | None
```

For GitHub MCP, map tools like `create_issue` → `{action_type: "write", resources: ["github://..."], estimated_impact: "medium"}`. This shape becomes the `action` field in the OPA input document.

### 4. GitHub MCP Integration

- Configure the official GitHub MCP server as a hardened backend.
- Register `mcp://github-tempo-org` in `mcp_registry.yaml` with scopes `github:read`, `github:write`.
- Add a Keycloak identity broker federation to GitHub OAuth, so a user logged into Keycloak can obtain a GitHub token scoped to the tempo org.
- Extend Rego `access` policy with GitHub-specific rules:
  - Only `platform_admin` may invoke `delete_repository`.
  - Any `github:write` action carries `mutating_score=30` in risk policy.
- Extend the dashboard with a simple "GitHub ops" panel (search issues, view PR status) for demo purposes.

### 5. Atlassian Rovo MCP (Optional)

If time permits, repeat step 4 for the Atlassian Rovo MCP. This is the third integration mentioned in the proposal. Skip if timeline is tight — one external MCP sufficiently demonstrates generalization.

### 6. Rate Limiting and Anomaly Detection

`stack-hardened/gateway/ratelimit/`:

- Token-bucket per subject + per MCP URI. Configurable limits in `mcp_registry.yaml`.
- After N consecutive denies from the same subject in a 60-second window, trigger a 5-minute subject lockout. Audit row: `reason=ratelimit.deny_storm`.
- Simple anomaly detection: track the median queries-per-minute per subject. If current rate > 5× median and absolute rate > 20/min, flag the session for security review (audit entry only; no automatic action).

### 7. Legitimate-Use Regression Suite Expansion

Expand `attacks/test_legitimate_use.py` with a substantial set (20+) of known-good queries per role. This is how you prove the hardening doesn't over-block. Every future stage runs this suite; regressions block merge.

### 8. Evaluation Matrix Script Enhancement

`evaluation/run_matrix.py` now produces:
- Attack success/block rate per stack per attack.
- Latency histogram per pipeline stage.
- Audit completeness: every request → exactly one audit row (or approval → N rows, where N is well-defined).
- DLP effectiveness: synthetic injected PII in responses, measure detection rate.
- Chain integrity: runs `tempo-audit-verify`, reports any chain break.

## Acceptance Criteria

- [ ] `tempo-audit-verify` returns 0 exit code on a clean log; fails with clear diagnostic when a row is manually tampered with.
- [ ] Revoking `UPDATE`/`DELETE` from the gateway DB role does not break any legitimate flow.
- [ ] DLP redacts fake SSNs, credit cards (Luhn-valid), and out-of-domain emails in responses. Verified by a DLP-specific test with synthetic payloads.
- [ ] GitHub MCP integration: a logged-in user can list issues in the tempo org through the gateway. Unauthorized users cannot.
- [ ] Unauthorized GitHub operations (e.g., `delete_repository` from non-admin) are denied with OPA reason.
- [ ] Rate limiter triggers lockout after threshold; verified by test.
- [ ] `evaluation/run_matrix.py` produces a complete report in `evaluation/out/` that covers all 5 attacks, both stacks, all layers, latency, DLP, and audit integrity.
- [ ] Legitimate-use regression suite: 100% pass rate on hardened stack.

## Deliverable Commit

Tag: `stage-6-complete`
Message: `feat(gateway): signed audit chain, DLP, GitHub MCP integration`

## What This Stage Is NOT

- Not the final report (Stage 7).
- Not adding new attacks (threat model is frozen).
- Not adding new external MCPs beyond GitHub (and optionally Atlassian).

## Notes for Claude Code

- Signed audit chains are easy to get wrong. Test by manually tampering and re-running the verifier. Test edge cases: row 1 (no predecessor), gap in ids, duplicate ids.
- Postgres `REVOKE DELETE, UPDATE ON audit.entries FROM gateway_role` and then verify by attempting the operation — it must fail.
- DLP patterns must be tested against real-world-like data. Ship an `evaluation/dlp_corpus.json` with 500+ synthetic examples covering both positives and tricky negatives (e.g., numbers that look like SSNs but aren't, emails inside code blocks).
- GitHub MCP's auth flow is OAuth 2.0 device or authorization code. Use device flow for the demo if browser redirects become tricky in Docker.
- The normalization layer is easy to under-engineer. Err on the side of explicit per-tool mappings rather than clever generic rules. The audit log should show exactly which normalization rule fired.

## Advance to Stage 7

Update `docs/stages/CURRENT_STAGE.md` to `STAGE_7`.

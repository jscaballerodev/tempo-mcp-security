# STAGE 3 — Hardened Gateway: Foundation Layers

## Goal

Stand up the Secure MCP Gateway with three defensive layers: **authentication (Keycloak + token validation)**, **resource indicator checking (RFC 8707)**, and the **SQL firewall (sqlglot-based)**. At the end of this stage, Attack 1 (SQL injection) and Attack 2 (token replay) are blocked on hardened while still succeeding on naive.

## Why These Three First

Attacks 1 and 2 are the lowest-complexity, highest-value wins. They don't require OPA, taint tracking, or approval workflows. Demonstrating them working early builds confidence in the architecture and gives a presentable mid-project demo.

## Prerequisites

- Stage 2 complete (`stage-2-complete` tag).
- Attack suite passes in naive mode; hardened-mode tests are expected-fail.

## Tasks

### 1. Keycloak Realm Configuration

Write `keycloak/realm-tempo.json` — a realm export that contains:
- Realm: `tempo`
- Clients:
  - `tempo-dashboard` (public client, PKCE required, redirect URI to hardened dashboard)
  - `tempo-gateway` (confidential client, service account, used for token introspection)
- Scopes: `mcp:read`, `mcp:write`, `mcp:admin`
- Resource server registrations for each MCP URI: `mcp://tempo-marketing`, `mcp://tempo-finance`, `mcp://tempo-hr`
- Roles: `marketing_analyst`, `finance_analyst`, `hr_admin`, `executive`, `platform_admin`
- Test users, one per role, with known passwords in `.env.example`

Import this realm automatically on Keycloak startup via the container's `--import-realm` flag.

### 2. Gateway Auth Module

`stack-hardened/gateway/auth/`:

- `auth/middleware.py` — FastAPI middleware that extracts the `Authorization: Bearer <token>` header, rejects requests without one.
- `auth/validator.py` — validates JWT signature against Keycloak's JWKS, checks `exp`, `iat`, `iss`, `typ`. Uses `python-jose` or `authlib`.
- `auth/introspect.py` — fallback to RFC 7662 introspection endpoint for opaque tokens.
- `auth/context.py` — builds a `SubjectContext` dataclass (subject id, roles, scopes, token id, issued at) that downstream modules consume.

The `verify()` entry point returns a `Decision`. On failure, return `deny` with reason like `auth.signature_invalid`, `auth.expired`, `auth.missing_scope`.

### 3. Gateway Resource Module (RFC 8707)

`stack-hardened/gateway/resource/`:

- `resource/validator.py` — given a parsed token and a requested MCP URI, enforce that the token's `aud` (audience) or `resource` claim matches the target.
- Configuration: a static map `mcp_registry.yaml` listing each MCP server's URI and its allowed scopes.
- Reject with reason `resource.audience_mismatch` if the token's resource claim does not include the requested MCP URI.

This is what blocks Attack 2. Write a targeted unit test before integrating.

### 4. Gateway SQL Firewall Module

`stack-hardened/gateway/sql_firewall/`:

- `sql_firewall/parser.py` — parses SQL with `sqlglot.parse_one(sql, dialect="postgres")`. Rejects statements that fail to parse as a single top-level statement.
- `sql_firewall/rules.py` — enforces:
  - **Statement type allowlist.** Default: `SELECT` only. `INSERT`/`UPDATE`/`DELETE` require role `*_admin` and explicit scope `mcp:write`.
  - **Table allowlist per role.** Lookup `app.table_sensitivity`; cross-reference with the role's permitted tiers from OPA (stubbed in this stage, fully integrated in Stage 4).
  - **Forbidden schema access.** Block all references to `pg_catalog`, `information_schema`, `audit.*`.
  - **Row limit injection.** If query is `SELECT` with no `LIMIT`, inject `LIMIT 100`. Track original vs modified SQL in audit.
  - **Comment stripping.** Strip SQL comments before parse to prevent comment-based bypasses.
- `sql_firewall/firewall.py` — orchestrates the rules and returns a `Decision` with a `modified_sql` field when a limit was injected.

Write extensive unit tests. SQL parsing is the most bug-prone module in the gateway.

### 5. Gateway Core Pipeline

`stack-hardened/gateway/core/pipeline.py`:

The request flow, in strict order:

```
1. auth.middleware     → extract token, 401 if missing
2. auth.validator      → validate JWT, 401 if invalid
3. resource.validator  → check audience, 401 if mismatch
4. sql_firewall        → parse & validate SQL, 403 if violation
5. [Stage 4] policy    → OPA decision
6. [Stage 5] taint     → check provenance
7. [Stage 5] approval  → route if high risk
8. forward to MCP server
9. [Stage 6] dlp       → filter output
10. audit write
11. return response
```

In Stage 3, steps 5–7 are stubs that always return `allow`. Steps 9–10 are minimal (basic audit writes, no DLP).

### 6. Hardened MCP Server

`stack-hardened/mcp-server-postgres/`:
- Same functional code as the naive MCP server.
- Binds to an internal network interface only — **not reachable from the dashboard directly**.
- Trusts the gateway implicitly (gateway is the only network neighbor). The gateway forwards pre-validated SQL.
- Uses a Postgres role with `SELECT` only by default; a separate elevated role is used only when the gateway explicitly signals a write operation via a per-request connection.

### 7. Minimal Audit Writer

`stack-hardened/gateway/audit/writer.py`:
- Writes to `audit.entries` table.
- Schema: `id, ts, subject, token_id, mcp_uri, tool, input_sql, modified_sql, verdict, reason, latency_ms`.
- Every pipeline exit produces exactly one row. Use a `finally` block.

Full append-only signing comes in Stage 6; Stage 3 just produces the rows.

### 8. Hardened Dashboard Login Flow

`stack-hardened/dashboard/`:
- Copy the naive dashboard as a starting point.
- Add Keycloak login via OIDC code+PKCE.
- Route API calls through the gateway, attaching the user's access token.
- The stack toggle in the header becomes functional: switch between naive and hardened backends.

### 9. Enable Hardened Attack Tests

Remove the `expected_fail_until_stage(3)` marker from Attack 1 and Attack 2 tests. Run the suite — these two attacks should now be blocked on hardened. Attacks 3, 4, 5 remain expected-fail until later stages.

## Acceptance Criteria

- [ ] `docker compose up` brings Keycloak with the `tempo` realm preloaded.
- [ ] Logging into the hardened dashboard with a test user produces a valid token that the gateway accepts.
- [ ] `pytest attacks/test_01_sql_injection.py --stack=hardened -v` — all 4 payloads blocked with reason in `sql_firewall.*`.
- [ ] `pytest attacks/test_02_token_replay.py --stack=hardened -v` — blocked with reason `resource.audience_mismatch`.
- [ ] `pytest attacks/ --stack=naive -v` — still 5/5 attacks successful (naive must not regress).
- [ ] Legitimate-use suite still passes on both stacks.
- [ ] `audit.entries` table contains one row per attack attempt against hardened, with correct verdicts.
- [ ] `python evaluation/run_matrix.py` shows a 2/5 block rate on hardened, 0/5 on naive.
- [ ] Gateway latency overhead per request: measured and logged in `evaluation/out/latency_stage3.json`.

## Deliverable Commit

Tag: `stage-3-complete`
Message: `feat(gateway): auth, resource binding, and SQL firewall — A01 and A02 blocked`

## What This Stage Is NOT

- Not integrating OPA for role-based decisions. The firewall's role-awareness is hardcoded; OPA comes in Stage 4.
- Not implementing taint tracking.
- Not adding approval workflows.
- Not implementing DLP.
- Not adding signed audit logs.

## Notes for Claude Code

- Test each module in isolation *before* wiring into the pipeline. SQL firewall bugs are especially common; build a fuzz harness with `sqlglot` round-tripping.
- Keycloak's JWKS endpoint is at `/realms/tempo/protocol/openid-connect/certs`. Cache keys with a short TTL — don't fetch on every request.
- Don't hardcode the Keycloak URL; use `OIDC_ISSUER` env var so the gateway works in both dev and a future staging environment.
- When the SQL firewall rewrites a query (injecting LIMIT), both the original and rewritten SQL must appear in the audit row. Tampering detection depends on this.
- Write at least one **legitimate query** unit test per defense layer. Preventing real users from working is worse than most attacks.

## Advance to Stage 4

Update `docs/stages/CURRENT_STAGE.md` to `STAGE_4`.

# STAGE 1 — Seed Data and Naive Stack

## Goal

Produce a working end-to-end naive stack: SvelteKit dashboard → MCP client → Postgres MCP server → Postgres with realistic seed data. No security controls anywhere. This is the target the attack suite will exercise in Stage 2.

## Why This Comes Before the Hardened Stack

Building the vulnerable version first serves three purposes: it proves the core functional path works, it creates a measurable baseline, and it forces the threat model to be concrete. You cannot attack what does not exist.

## Prerequisites

- Stage 0 complete (`stage-0-complete` tag exists).
- `docker compose up` brings all placeholder services online.

## Tasks

### 1. Postgres Schema with Sensitivity Tags

Write `seed/schema.sql`. Create two schemas: `app` and `audit`. In `app`, create the following tables, each with a comment containing its sensitivity tag:

| Table | Sensitivity | Fields (summary) |
|---|---|---|
| `marketing_campaigns` | `public` | id, name, channel, start_date, spend, impressions |
| `public_metrics` | `public` | metric_name, value, recorded_at |
| `customer_accounts` | `internal` | id, email, plan, created_at, region |
| `support_tickets` | `internal` | id, customer_id, subject, body, status, created_at |
| `finance_transactions` | `confidential` | id, account, amount, category, txn_date |
| `budget_lines` | `confidential` | id, department, line_item, annual_budget |
| `hr_salaries` | `restricted` | employee_id, base_salary, bonus, currency |
| `employee_pii` | `restricted` | employee_id, full_name, ssn, dob, home_address |
| `exec_compensation` | `secret` | executive_id, quarter, cash, equity, perks |
| `merger_notes` | `secret` | id, target_company, status, notes, updated_at |

Store sensitivity as both a SQL comment *and* in a sidecar table `app.table_sensitivity (table_name TEXT PRIMARY KEY, tier TEXT)` so the gateway can query it programmatically.

### 2. Seed Data

Write `seed/data/*.sql` files with **realistic but fake** TEMPO data. Guidelines:
- At least 50 rows per table, 500+ for `customer_accounts` and `support_tickets`.
- Deterministic: use fixed seed for any generation. The same `docker compose up` produces the same data.
- One `support_tickets` row must contain a deliberately planted prompt injection payload (for Attack 4 in Stage 2). Mark it with a comment but leave it in the data.
- Fake SSNs must be obvious test values (000-00-0001 pattern). Fake emails use `@example.test`.

Use a Python generator script `seed/generate.py` that emits the SQL files. Commit both the generator and the generated SQL.

### 3. Naive MCP Server

In `stack-naive/mcp-server-postgres/`, implement a minimal MCP server that exposes a single tool:

```
tool: query_database
params: { sql: string }
returns: { columns: [...], rows: [...] }
```

Requirements:
- Uses the official MCP Python SDK.
- Connects to Postgres with a role that has `SELECT, INSERT, UPDATE, DELETE` on `app.*` (deliberately over-privileged).
- **No authentication.** No token checks. Listens on an internal Docker network port.
- Executes whatever SQL it receives, verbatim.
- Returns results as JSON.
- Logs nothing. This is the baseline insecure state.

### 4. NL→SQL Microservice

In `stack-naive/nl2sql/`, build a small FastAPI service (not the gateway — a separate helper) that:
- Accepts `POST /generate { prompt: string, schema: string }`.
- Calls Claude API or a local Ollama model to produce SQL.
- Returns `{ sql: string, model: string, latency_ms: int }`.

This service is shared between both stacks — the security question is not "which LLM" but "what does the gateway do with the SQL." Keep it dumb and deterministic where possible (low temperature, fixed system prompt).

### 5. Naive Dashboard

In `stack-naive/dashboard/`, build a SvelteKit app with:
- A single page with a text input ("Ask a question about TEMPO data") and a "Run" button.
- On submit: call NL→SQL service, display the SQL, call the naive MCP server, display results as a table.
- A toggle switch "Stack: naive" in the header (disabled — will become meaningful in Stage 3 when the hardened dashboard exists).
- Zero auth. Anyone who can reach the page can run queries.

Use Chart.js or a lightweight charting lib to plot numeric results when they look time-series-like. This is the "insights about the data" feature from the original brief.

### 6. Integration Test

Write `tests/integration/test_naive_happy_path.py`:
- Boots the stack, waits for health.
- Sends the prompt `"how much did we spend on marketing last quarter?"`.
- Asserts SQL is generated, executed, and a numeric result is returned.
- Cleans up.

This is NOT an attack test — it is a smoke test proving the pipeline works. Attack tests come in Stage 2.

## Acceptance Criteria

- [ ] `docker compose up` brings all services online; health checks pass.
- [ ] Opening `http://localhost:5173` (naive dashboard) shows the input page.
- [ ] Asking "how much did we spend on marketing last quarter?" returns a SQL query, executes it, and displays a result.
- [ ] Asking "list all customers" returns actual customer rows.
- [ ] Running `psql` directly against the `app` schema shows all 10 tables populated.
- [ ] `app.table_sensitivity` contains one row per table with the correct tier.
- [ ] `pytest tests/integration/test_naive_happy_path.py` passes.
- [ ] The planted prompt injection payload exists in `support_tickets` and is retrievable.

## Deliverable Commit

Tag: `stage-1-complete`
Message: `feat(naive): end-to-end naive stack with seed data and NL-to-SQL`

## What This Stage Is NOT

- Not adding any authentication.
- Not adding any logging beyond what Postgres does by default.
- Not writing attack tests.
- Not implementing the hardened stack — only the naive one.
- Not building the approval UI.

## Notes for Claude Code

- The MCP SDK is under active development; check the version pinned in `pyproject.toml` before using any API.
- Keep the NL→SQL service stateless. No session memory. Every request is independent.
- Dashboard UI can be ugly — styling is not the point of this stage. Functional > pretty.
- If the LLM produces invalid SQL, surface the error in the dashboard rather than silently failing. This is useful for debugging in later stages.

## Advance to Stage 2

Update `docs/stages/CURRENT_STAGE.md` to `STAGE_2`.

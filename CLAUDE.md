# CLAUDE.md — Project Instructions for Claude Code

## Project Overview

This is **TEMPO MCP Security Gateway**, an advanced cybersecurity capstone project that demonstrates the difference between a naive MCP (Model Context Protocol) integration and a hardened one protected by a verification gateway. The project is a working prototype that backs the academic proposal in `docs/PROPOSAL.pdf`.

The project owner is a PhD-level developer working primarily in Python, TypeScript/Svelte, and Rego. Assume technical fluency. Prefer correctness and clarity over verbosity.

## Core Architectural Rule: Two-Stack Convention

The repository contains **two parallel stacks** that implement identical functionality with opposite security postures. Every feature added to one must have a corresponding (or deliberately absent) counterpart in the other.

```
stack-naive/      ← vulnerable reference implementation (the cautionary tale)
stack-hardened/   ← the contribution: gateway-protected version
```

Both stacks connect to the **same PostgreSQL instance** with the **same seed data**. They must be interchangeable from the dashboard's perspective — the only difference is whether requests flow through the Secure MCP Gateway.

**Never** add a security feature to `stack-naive/`. Its purpose is to be insecure. If you think a feature belongs in naive, stop and ask.

## Red Team First, Blue Team Second

The `attacks/` directory contains a pytest suite that is **append-only for tests**. The rule is strict:

- You may add new attack tests.
- You may **never** modify an existing attack test to make it pass.
- If an attack test fails against the hardened stack, fix the gateway, not the test.
- Each attack test runs in two modes: `--stack=naive` (asserts attack succeeded) and `--stack=hardened` (asserts attack was blocked and audited).

When asked to implement a defense, phrase the goal as *"make attack X fail against hardened without breaking legitimate-use tests"* — not *"add feature Y."*

## Repository Layout

```
tempo-mcp-security/
├── CLAUDE.md                    # this file
├── THREAT_MODEL.md              # canonical attack list — read before security work
├── docker-compose.yml           # postgres, keycloak, opa, both stacks
├── docs/
│   ├── PROPOSAL.pdf             # academic proposal (reference only)
│   └── stages/                  # per-stage instructions (STAGE_0.md … STAGE_7.md)
├── seed/
│   ├── schema.sql               # Postgres schema with sensitivity tags
│   └── data/                    # fake TEMPO data
├── stack-naive/
│   ├── mcp-server-postgres/     # unauthenticated MCP server
│   └── dashboard/               # SvelteKit, direct MCP connection
├── stack-hardened/
│   ├── gateway/                 # FastAPI: the contribution
│   │   ├── auth/                # Keycloak + token validation
│   │   ├── resource/            # RFC 8707 resource indicator checks
│   │   ├── policy/              # OPA client
│   │   ├── sql_firewall/        # sqlglot-based SQL parser/validator
│   │   ├── taint/               # provenance tracking
│   │   ├── approval/            # human-in-the-loop queue
│   │   ├── dlp/                 # output filtering
│   │   └── audit/               # signed append-only log
│   ├── mcp-server-postgres/     # same MCP, but only reachable via gateway
│   └── dashboard/               # SvelteKit + Keycloak login + approval UI
├── policies/                    # Rego files for OPA
├── attacks/                     # pytest suite — append-only
│   ├── conftest.py              # --stack flag, fixtures
│   ├── test_01_sql_injection.py
│   ├── test_02_token_replay.py
│   ├── test_03_privilege_escalation.py
│   ├── test_04_prompt_injection.py
│   └── test_05_bulk_exfiltration.py
└── evaluation/                  # scripts that produce report artifacts
```

## Technology Constraints

- **Gateway language:** Python 3.11+ with FastAPI. Do not switch to TypeScript mid-project.
- **Dashboard:** SvelteKit. Do not substitute React or Gradio for the main UI. Gradio is allowed only as an internal microservice for NL→SQL generation.
- **DB:** PostgreSQL 16. Use separate schemas for `app` and `audit`.
- **Identity:** Keycloak for local dev. Design auth code so an enterprise IdP swap (Okta, Auth0) is a config change.
- **Policy:** OPA with Rego. Policies live in `policies/` and are loaded as a bundle.
- **SQL parser:** `sqlglot`. Do not use regex for SQL validation.
- **Containerization:** Docker Compose for all services. Every stage must end with `docker compose up` producing a working demo.

## Coding Conventions

- Python: `ruff` + `black` + `mypy --strict`. Type hints everywhere.
- Commits: conventional commits (`feat:`, `fix:`, `test:`, `docs:`). One stage per feature branch.
- Each gateway verification step from the proposal (sections 2.4–2.5) lives in its own module. Do not merge them.
- Every defensive module exposes a single `verify()` or `enforce()` function returning a `Decision` dataclass: `allow | deny | require_approval | redact`.
- Log every decision to the audit schema, including denies. Silent denies are bugs.

## How to Work with This Project

When the user starts a new session, check which stage is active by reading `docs/stages/CURRENT_STAGE.md`. Work only on that stage's acceptance criteria unless explicitly told otherwise.

Before writing code:
1. Re-read the relevant `STAGE_N.md` from `docs/stages/`.
2. Re-read `THREAT_MODEL.md` if touching the gateway or attacks.
3. Check what's already implemented with `git log --oneline` and a quick tree view.

When finishing a stage:
1. Run the full attack suite in both modes and paste the matrix.
2. Update `docs/stages/CURRENT_STAGE.md` to the next stage.
3. Tag the commit `stage-N-complete`.

## What Not to Do

- Do not add authentication to the naive stack "just in case."
- Do not modify attack tests to make them pass.
- Do not introduce new technologies without updating this file first.
- Do not use an LLM inside the gateway's decision path. The gateway must be deterministic; LLMs belong in the dashboard's NL→SQL layer, upstream of the gateway.
- Do not commit real secrets. Use `.env.example` templates; actual `.env` files are gitignored.
- Do not "optimize" the naive stack. Resist the instinct.

## Reference Documents

- `THREAT_MODEL.md` — the five attacks, canonical and versioned
- `docs/PROPOSAL.pdf` — the academic proposal; cite section numbers when justifying design decisions
- `docs/stages/STAGE_*.md` — per-stage instructions and acceptance criteria

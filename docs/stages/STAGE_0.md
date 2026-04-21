# STAGE 0 — Repo Scaffolding and Threat Model

## Goal

Establish the repository structure, threat model, and empty service stubs. No functional code in this stage — the point is to commit to the architecture before writing anything that depends on it.

## Why This Stage First

A repeated failure mode in security projects is building features and only later discovering they don't compose. By fixing the directory layout, the two-stack convention, and the attack list *before* any code exists, every subsequent stage has a stable target.

## Prerequisites

- Docker and Docker Compose installed
- Python 3.11+, Node 20+, Rust toolchain optional
- Empty git repository initialized

## Tasks

1. **Write the root files.** Create `CLAUDE.md`, `THREAT_MODEL.md`, `README.md`, `.gitignore`, `.editorconfig`, `LICENSE`. Copy the canonical `CLAUDE.md` and `THREAT_MODEL.md` from the project owner's templates.

2. **Create the directory tree exactly as specified in `CLAUDE.md`.** Every directory should contain a `.gitkeep` or a stub file so the structure survives git.

3. **Write `docker-compose.yml` with placeholder services:**
   - `postgres` (official image, port 5432, volume-mounted)
   - `keycloak` (quay.io/keycloak/keycloak, dev mode, port 8080)
   - `opa` (openpolicyagent/opa, port 8181, bundle mount)
   - `naive-mcp` (placeholder: `python:3.11-slim` running `sleep infinity`)
   - `naive-dashboard` (placeholder: `node:20-slim` running `sleep infinity`)
   - `hardened-gateway` (placeholder)
   - `hardened-mcp` (placeholder)
   - `hardened-dashboard` (placeholder)

   Each service needs a health check, a defined network, and env vars loaded from `.env`.

4. **Create `.env.example`** with all needed variables but no real values. Add `.env` to `.gitignore`.

5. **Write `docs/stages/CURRENT_STAGE.md`** containing just the text `STAGE_0` initially. This file is the session pointer — future sessions start by reading it.

6. **Scaffold the pytest harness** in `attacks/` with `conftest.py` that exposes a `--stack=naive|hardened` flag. Do not write any attack tests yet. That's Stage 2.

7. **Scaffold the gateway module tree** in `stack-hardened/gateway/` with empty `__init__.py` files and a `Decision` dataclass in `gateway/core/decision.py`:

    ```python
    from dataclasses import dataclass
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
        reason: str
        metadata: dict[str, Any]
    ```

   Every verification module will return this type.

8. **Write a `Makefile`** with targets: `up`, `down`, `logs`, `attack-naive`, `attack-hardened`, `attack-both`, `clean`, `reset-db`.

## Acceptance Criteria

- [ ] `docker compose up` starts all services without errors (health checks pass).
- [ ] `docker compose down -v` cleans up completely.
- [ ] `git ls-files` shows the full tree matching `CLAUDE.md`.
- [ ] `pytest attacks/ --collect-only` runs without errors (no tests collected yet is fine).
- [ ] `make help` lists all targets.
- [ ] `docs/stages/CURRENT_STAGE.md` reads `STAGE_0`.

## Deliverable Commit

Tag: `stage-0-complete`
Message: `chore: scaffold repo, docker stack, threat model`

## Advance to Stage 1

Update `docs/stages/CURRENT_STAGE.md` to `STAGE_1`. Do not begin Stage 1 work until this stage's acceptance criteria are all checked.

## What This Stage Is NOT

- Not implementing any MCP server logic.
- Not writing attack tests.
- Not configuring Keycloak realms — only making the container boot.
- Not writing Rego policies.

---
name: attack-test-authoring
description: Use when writing, modifying, or debugging pytest attack tests in attacks/ for the TEMPO gateway project. Covers the dual-mode stack fixture pattern, the append-only rule for existing attacks, the expected-fail-until-stage marker, and the three required test functions per attack (succeeds-on-naive, blocked-on-hardened, audited-on-hardened). Trigger for any work under attacks/ or when adding a new threat scenario.
---

# Attack Test Authoring

The `attacks/` directory is the evaluation harness. Every test is structured to run against both stacks and produce measurable security evidence.

## The append-only rule

**You may add new attack tests. You may never modify an existing one to make it pass.**

If an attack test fails against the hardened stack, the fix is always in the gateway or policies, never in the test. If the threat model itself needs to change, that's a conversation with the project owner and a versioned update to `THREAT_MODEL.md`, not an inline test edit.

## Every attack test file has the same shape

```python
# attacks/test_0N_<snake_case_name>.py
import pytest
from attacks.harness import run_attack, AuditQuery

ATTACK_ID = "A0N_NAME"  # matches THREAT_MODEL.md


class TestAttackNN:
    """A0N — <short description>. See THREAT_MODEL.md § Attack N."""

    PAYLOAD = {...}  # the malicious input

    def test_succeeds_on_naive(self, naive_client):
        result = run_attack(naive_client, payload=self.PAYLOAD)
        assert result.data_leaked or result.destructive_action, (
            f"{ATTACK_ID} must succeed against naive stack — "
            "if this fails, the naive stack has grown an unwanted defense."
        )

    def test_blocked_on_hardened(self, hardened_client):
        result = run_attack(hardened_client, payload=self.PAYLOAD)
        assert not result.data_leaked
        assert not result.destructive_action
        assert result.status_code in (401, 403, 202)

    def test_audited_on_hardened(self, hardened_client, audit_db):
        run_attack(hardened_client, payload=self.PAYLOAD)
        entry = AuditQuery(audit_db).latest_for_subject(hardened_client.subject)
        assert entry is not None, "every attempt must produce an audit row"
        assert entry.verdict in ("deny", "require_approval")
        assert entry.reason.startswith(self.EXPECTED_REASON_PREFIX)
```

Three test functions. No fewer. The audit test is not optional — audit integrity is part of what's being evaluated.

## Stack fixtures

From `attacks/conftest.py`:

- `naive_client` — hits the naive MCP server directly, no auth.
- `hardened_client` — authenticates through Keycloak, hits the gateway. Receives a test user keyed to the attack's threat actor.
- `audit_db` — read-only SQLAlchemy session on the `audit` schema.

Never build your own client in a test. If the fixtures don't cover a case, extend them.

## The expected-fail-until-stage marker

When writing hardened-mode tests before the defense exists:

```python
@pytest.mark.expected_fail_until_stage(3)
def test_blocked_on_hardened(self, hardened_client):
    ...
```

This marker is defined in `attacks/conftest.py`. During Stage 2, all hardened-mode tests carry it and are reported but not counted as CI failures. When the defense lands in the stated stage, remove the marker. Forgetting to remove it is a bug — the test silently passes even when defenses regress.

## The legitimate-use suite

Every attack has a counterpart in `attacks/test_legitimate_use.py` that proves the defense doesn't over-block. If you add Attack 6, you also add at least three legitimate queries that *should* work post-defense. Over-blocking is a worse failure mode than most attacks because it guarantees the system isn't adopted.

## Non-determinism (Attack 4 specifically)

Prompt injection tests depend on LLM behavior and are flaky by nature. Mitigations required for any LLM-dependent attack test:

1. Pin model, temperature=0, seed when the API supports it.
2. Use `@pytest.mark.flaky(reruns=3)` — success on any of three runs counts.
3. If the model never falls for the injection across runs, the test is `xfail` with a note, not deleted. The attack is still a finding; the specific LLM just resisted on that day.
4. Log the LLM response in the test output so failures are diagnosable.

## Attack-specific data setup

Some attacks require planted data (Attack 4's malicious support ticket). Don't insert it in the test — it's in `seed/data/` from Stage 1, deterministically. Tests assume seed data is present and reset.

If your attack needs state that survives setup, add it to seed. If it needs per-test state, use a pytest fixture with explicit cleanup.

## Running the suite

```bash
# Naive only — run at every stage to confirm nothing's been accidentally secured
pytest attacks/ --stack=naive -v

# Hardened only — run at each stage to track block progress
pytest attacks/ --stack=hardened -v

# Both, produces the evaluation matrix
python evaluation/run_matrix.py
```

## Common traps

**Trap 1: flaky assertions on naive.** The naive stack can produce noisy data (large result sets, network timeouts). Assertions must be robust: test for *presence* of leaked data, not exact byte count.

**Trap 2: audit race condition.** The audit write happens in a finally block which may not complete before the HTTP response returns. Use `audit_db.wait_for_subject(subject, timeout=2)` rather than an immediate query.

**Trap 3: shared state between tests.** Attack 1's DROP TABLE test must run in an isolated database. Either use per-test Postgres schemas or reset between tests. Never assume ordering.

**Trap 4: asserting the wrong layer.** If Attack 3 is "blocked by SQL firewall" in the test but the gateway actually blocks it at the policy layer, the test is misleading. Assert on `reason.startswith("policy.")` rather than "`sql_firewall.`" unless you genuinely know which layer should catch it.

## Reference documents

- `THREAT_MODEL.md` — the five canonical attacks, each with `Defense layer owner`
- `docs/stages/STAGE_2.md` — initial suite construction
- `attacks/harness.py` — the `AttackClient`, `AttackResult`, `AuditQuery` types

# STAGE 7 — Evaluation, Demo, and Report

## Goal

Produce the artifacts that turn the working prototype into a defensible academic project: reproducible evaluation results, a live demo script, and a final writeup that maps directly to the proposal's objectives (section 2.1) and evaluation criteria (section 2.8).

## Prerequisites

- Stage 6 complete. All layers implemented. Attack suite green in both modes.

## Tasks

### 1. Evaluation Harness: One Command, Full Report

`make evaluate` should:

1. Reset Postgres to known seed state.
2. Run `opa test policies/` — record pass/fail.
3. Run `pytest attacks/ --stack=naive --json-report`.
4. Run `pytest attacks/ --stack=hardened --json-report`.
5. Run the legitimate-use regression suite on both stacks.
6. Run the DLP corpus evaluation (precision/recall).
7. Run `tempo-audit-verify` on the accumulated audit log.
8. Run latency benchmarks: 100 requests per pipeline stage, record p50/p95/p99.
9. Render all results into `evaluation/out/report.md` with embedded tables and charts.
10. Produce `evaluation/out/report.pdf` via pandoc.

The goal: one command, reproducible every time, from a clean `git clone`.

### 2. The Comparison Matrix

Render `evaluation/out/matrix.md`:

| Attack | Naive result | Hardened result | Reason | Layer responsible | Audit entry |
|---|---|---|---|---|---|
| A01 SQL Injection | ✗ data leaked | ✓ blocked | `sql_firewall.non_select` | SQL firewall | yes |
| A02 Token Replay | ✗ token accepted | ✓ blocked | `resource.audience_mismatch` | Auth + Resource | yes |
| A03 Privilege Escalation | ✗ rows returned | ✓ blocked | `policy.role_sensitivity_mismatch` | OPA | yes |
| A04 Prompt Injection | ✗ HR data leaked | ⧖ approval required | `provenance.untrusted_cross_domain` | Taint + OPA | yes |
| A05 Bulk Exfiltration | ✗ 1200 rows dumped | ⧖ approval required | `risk.bulk_read` | SQL firewall + OPA | yes |

Legend: ✗ attack succeeded, ✓ attack blocked, ⧖ gated through approval.

### 3. Latency Budget Report

Produce a latency table per pipeline stage:

| Stage | p50 | p95 | p99 |
|---|---|---|---|
| Token validation | 0.8 ms | 1.2 ms | 2.1 ms |
| Resource check | 0.1 ms | 0.2 ms | 0.3 ms |
| SQL firewall | 2.1 ms | 3.8 ms | 6.2 ms |
| OPA access | 3.2 ms | 5.1 ms | 8.4 ms |
| OPA risk | 2.8 ms | 4.6 ms | 7.1 ms |
| Taint extraction | 0.3 ms | 0.5 ms | 0.7 ms |
| DLP scan | 4.5 ms | 7.8 ms | 12.1 ms |
| Audit write | 1.9 ms | 3.1 ms | 5.2 ms |
| **Total gateway overhead** | **~16 ms** | **~26 ms** | **~42 ms** |

Actual numbers depend on hardware. Target: hardened p95 overhead under 50ms. If over, identify the hot path and optimize.

### 4. Audit Completeness Report

Count:
- Total gateway requests during evaluation run.
- Total audit rows written.
- Difference (should be zero).
- Decisions breakdown: allow / deny / require_approval / redacted.
- Chain integrity: verified or not.

### 5. DLP Precision and Recall

Run against `evaluation/dlp_corpus.json`. Report:
- Precision: of things flagged as PII, what fraction actually were.
- Recall: of actual PII, what fraction was flagged.
- False positive examples (for the writeup).
- False negative examples (for the "future work" section).

### 6. Mapping to Proposal

Write `docs/PROPOSAL_MAPPING.md` that explicitly ties each proposal section to the corresponding evaluation result:

| Proposal objective (§ 2.1) | Implementation | Evidence |
|---|---|---|
| Authenticate users through centralized IdP | Keycloak + Stage 3 auth module | `evaluation/out/auth_tests.json` |
| Validate OAuth/OIDC tokens | `gateway/auth/validator.py` | Attack 2 block, audit trail |
| Verify token intended for MCP resource | `gateway/resource/` (RFC 8707) | Attack 2 block |
| Tool-specific authorization | OPA policies | Attack 3 block |
| Human approval for high-risk | Stage 5 approval workflow | Attack 4/5 approval gates |
| Detailed audit trail | Signed chain, Stage 6 | `tempo-audit-verify` output |
| Adversarial evaluation | pytest attack suite | 5/5 block/gate rate |

This mapping is what the examiner will use. Make it airtight.

### 7. Demo Script

`docs/DEMO_SCRIPT.md` — a 15-minute live walkthrough:

1. **Setup (1 min).** Show `make up`, services green, both dashboards accessible.
2. **Naive baseline (3 min).** Log in as marketing analyst. Ask benign question. Show data returned. Ask "show me executive compensation." Show data leaked.
3. **Switch to hardened (2 min).** Toggle stack switch. Same user, same question. Show 403 + audit entry.
4. **Prompt injection demo (4 min).** Show the planted ticket. Ask a benign summarization question. Dashboard retrieves the ticket. LLM tries to query restricted data. Show approval prompt. Log in as HR admin in second browser. Show the approval UI with full context. Deny it. Show the audit trail.
5. **Attack suite in CI (2 min).** Run `make evaluate`. Show the matrix.
6. **Audit tampering (2 min).** Manually `UPDATE audit.entries` in psql, re-run `tempo-audit-verify`, show chain broken.
7. **External MCP (1 min).** GitHub issue list through gateway, then the same user attempting `delete_repository` and being denied.

Script should be rehearsable and runnable in a classroom setting.

### 8. Screenshots for the Report

Produce paired screenshots (naive vs hardened) for each attack. Store in `docs/screenshots/`:
- `A01_naive.png` — dashboard showing leaked data.
- `A01_hardened.png` — dashboard showing 403 + audit entry.
- Continue for A02–A05.
- Bonus: approval UI, audit chain verifier output.

### 9. Final Report Document

`docs/FINAL_REPORT.md` (compiled to PDF):

Sections:
1. Abstract
2. Introduction (reuse from proposal)
3. Threat model (from `THREAT_MODEL.md`)
4. Architecture (from `CLAUDE.md` + diagrams)
5. Implementation (per-stage narrative with screenshots)
6. Evaluation
   - Attack suite results
   - Latency
   - DLP precision/recall
   - Audit integrity
   - Legitimate-use regressions
7. Discussion
   - What the gateway does well
   - Limitations (explicitly: LLM-layer attacks, supply chain, DoS)
   - Comparison with alternative designs (client-side enforcement, per-MCP enforcement)
8. Future Work (expand from proposal § 4.2)
9. References (from proposal, updated)
10. Appendices: full policy listings, pipeline diagrams, reproducibility instructions.

### 10. Reproducibility Artifact

`REPRODUCE.md` at repo root:

```
git clone <repo>
cp .env.example .env   # no changes needed for local demo
make up                # boots all services
make evaluate          # runs full eval, produces evaluation/out/report.pdf
make demo              # opens dashboards and pre-populates demo state
```

Anyone with Docker should be able to reproduce every figure in the report.

## Acceptance Criteria

- [ ] `make evaluate` runs end-to-end on a clean clone and produces `evaluation/out/report.pdf`.
- [ ] `docs/FINAL_REPORT.md` renders cleanly to PDF via pandoc.
- [ ] `docs/PROPOSAL_MAPPING.md` ties every proposal objective to a measurable result.
- [ ] Demo script runs in under 20 minutes without errors.
- [ ] All screenshots produced and referenced in the report.
- [ ] `tempo-audit-verify` passes on the final audit log from the evaluation run.
- [ ] Latency p95 under 50ms per request on reference hardware; if not, document why and what would fix it.
- [ ] DLP precision ≥ 0.9, recall ≥ 0.85 on the corpus.

## Deliverable Commit

Tag: `v1.0-final`
Message: `chore: final evaluation, demo script, and report`

## What This Stage Is NOT

- Not adding new features.
- Not refactoring for aesthetics.
- Not writing a paper — the report is a course deliverable, not a publication. Keep it focused.

## Notes for Claude Code

- When generating the report, avoid regenerating content that could be inconsistent run-to-run. Every number in the report should be pulled from `evaluation/out/*.json` — no hand-typed figures.
- pandoc + a simple LaTeX template is enough for the PDF. Don't over-engineer the typography.
- The screenshots matter more than the prose for defense. Invest in clear, cropped, annotated images.
- Rehearse the demo at least twice before presenting. Docker boot order issues will bite on the day.
- Include a one-page executive summary at the top of the report for readers who won't go deeper.

## After Stage 7

The project is complete. Future extensions (section 4.2 of the proposal) belong in a separate branch or follow-on project:
- Enterprise IdP integration (Okta, Auth0).
- Anomaly detection with ML.
- Extended DLP with classifiers beyond regex.
- Policy simulation tooling.
- Multi-tenant gateway deployment.

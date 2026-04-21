# TEMPO MCP Security — Instruction Set for Claude Code

This bundle contains all the markdown instruction documents for the TEMPO Secure MCP Gateway project. Drop them into your repo as follows:

```
your-repo/
├── CLAUDE.md                          ← from this bundle
├── THREAT_MODEL.md                    ← from this bundle
└── docs/
    └── stages/
        ├── CURRENT_STAGE.md           ← from this bundle (contents: STAGE_0)
        ├── STAGE_0.md                 ← from this bundle
        ├── STAGE_1.md                 ← from this bundle
        ├── STAGE_2.md                 ← from this bundle
        ├── STAGE_3.md                 ← from this bundle
        ├── STAGE_4.md                 ← from this bundle
        ├── STAGE_5.md                 ← from this bundle
        ├── STAGE_6.md                 ← from this bundle
        └── STAGE_7.md                 ← from this bundle
```

## How to use this with Claude Code

1. Initialize an empty git repo.
2. Drop `CLAUDE.md`, `THREAT_MODEL.md`, and the `docs/stages/` directory into place.
3. Commit them: `git add . && git commit -m "docs: initial project instructions"`.
4. Open Claude Code in the repo. It will automatically read `CLAUDE.md` on every session.
5. Start each session with: *"Read `docs/stages/CURRENT_STAGE.md` and continue the current stage."*
6. When a stage completes, update `CURRENT_STAGE.md` to the next stage and tag the commit.

## Why this structure works for Claude Code

- **CLAUDE.md as the persistent system prompt.** Claude Code reads this file automatically. It contains the architectural rules that must hold across all sessions — the two-stack convention, the append-only test rule, technology constraints.

- **Per-stage files keep context windows manageable.** Instead of one giant design doc, each stage is self-contained with its own tasks, acceptance criteria, and scope boundaries. Claude Code only needs to load the current stage.

- **CURRENT_STAGE.md as a pointer.** A single line of text that tells the next session where to pick up. Trivial to update, unambiguous.

- **Threat model as a fixed target.** Because the attack list is frozen and versioned, every defensive stage has a measurable goal. Claude Code can verify its own work against the acceptance criteria.

## Adapting this to your workflow

- **Shorter timeline?** Skip Stage 6's external MCP integration and Stage 7's extensive report sections. The core contribution (Stages 3–5) is defensible on its own.
- **Longer timeline?** Extend Stage 6 with more external MCPs (Atlassian Rovo, custom enterprise services) and push anomaly detection from "future work" into an actual implementation.
- **Solo vs team?** The structure is written for solo work. For a team, assign each stage to a person; the acceptance criteria make handoffs clean.

## A note on scope

Every stage file has a "What This Stage Is NOT" section. These are not suggestions. Scope creep is the single biggest risk to a capstone project. If Claude Code proposes features listed under "is not," push back.

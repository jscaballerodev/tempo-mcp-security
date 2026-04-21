# Getting Started with Claude Code on the TEMPO Project

This guide walks you from empty directory → running Stage 0 session with Claude Code.

## Prerequisites

- Claude Code installed (`claude` CLI available)
- Docker + Docker Compose
- Python 3.11+, Node 20+
- Git
- Your existing Ollama-based NL→SQL prototype in any folder — we'll port it in Stage 1, not now

## Step 1: Initialize the repo

```bash
mkdir tempo-mcp-security
cd tempo-mcp-security
git init
```

## Step 2: Drop in the instruction documents

Copy from the earlier bundle:

```
tempo-mcp-security/
├── CLAUDE.md
├── THREAT_MODEL.md
└── docs/
    ├── PROPOSAL.pdf          # your academic proposal
    └── stages/
        ├── CURRENT_STAGE.md  # contents: STAGE_0
        ├── STAGE_0.md
        ├── STAGE_1.md
        ├── STAGE_2.md
        ├── STAGE_3.md
        ├── STAGE_4.md
        ├── STAGE_5.md
        ├── STAGE_6.md
        └── STAGE_7.md
```

## Step 3: Drop in the skills

From this bundle, copy the `.claude/` directory into your repo root:

```
tempo-mcp-security/
├── .claude/
│   └── skills/
│       ├── mcp-gateway-dev/
│       │   └── SKILL.md
│       ├── rego-policy-authoring/
│       │   └── SKILL.md
│       ├── attack-test-authoring/
│       │   └── SKILL.md
│       └── ollama-nl2sql/
│           └── SKILL.md
├── CLAUDE.md
├── THREAT_MODEL.md
└── docs/...
```

`.claude/skills/` is a **project-scoped** skills directory. Claude Code discovers skills here automatically when you start a session in this repo. You don't need to configure anything else.

## Step 4: Commit everything before writing code

```bash
git add .
git commit -m "docs: initial project instructions and skills"
```

This baseline commit is important. Every stage-complete tag will diff against this starting point.

## Step 5: Stash your existing Ollama code somewhere accessible

Your existing Gradio-based NL→SQL prototype should **not** be dropped into the repo yet. Keep it in its current location. You'll port it in Stage 1, guided by the `ollama-nl2sql` skill.

If you want, copy it into a `_prior_art/` directory inside the repo and add `_prior_art/` to `.gitignore`. This keeps it reachable without polluting the project tree:

```bash
mkdir _prior_art
cp -r /path/to/your/gradio-project _prior_art/
echo "_prior_art/" >> .gitignore
git add .gitignore
git commit -m "chore: gitignore prior art staging area"
```

## Step 6: Start the first Claude Code session

Open Claude Code in the repo:

```bash
cd tempo-mcp-security
claude
```

Your first message to Claude Code should be short and explicit. Something like:

> Read `CLAUDE.md`, `THREAT_MODEL.md`, and `docs/stages/CURRENT_STAGE.md`. Then read the stage file it points to. Before you start writing code, tell me in plain English what you understand this stage to require, and which skills you plan to use. Do not start implementation until I confirm.

This first message does four things:

1. Forces Claude Code to load the persistent context.
2. Confirms it found and understands the current stage.
3. Asks it to surface which skills it plans to use — this catches the case where skills aren't being discovered.
4. Introduces a human gate before any code is written, which prevents wrong-direction starts.

When it responds correctly, something like *"I understand Stage 0 requires repo scaffolding, docker-compose with health checks, and the attack test harness skeleton. I plan to use the attack-test-authoring skill when scaffolding attacks/conftest.py"*, say *"Good. Proceed with Stage 0."*

If it doesn't mention the skills or misinterprets the stage, correct it explicitly before letting it write code.

## Step 7: Working through a stage

Each stage has explicit acceptance criteria. The healthy pattern:

1. **Plan message.** "Outline what you'll do for this stage. Break it into commits."
2. **Approve the plan.** Push back on anything that crosses into "What This Stage Is NOT."
3. **Work in increments.** Let Claude Code implement one task from the stage file, run the acceptance check for that task, move to the next.
4. **Verify acceptance criteria.** At the end of the stage, go through the checklist in the stage file item by item. *"Run criterion 3 from STAGE_3.md and show me the output."*
5. **Tag the commit.** Once all criteria are green, `git tag stage-3-complete`.
6. **Advance the pointer.** Update `docs/stages/CURRENT_STAGE.md` to the next stage and commit.

## Step 8: Starting a new session mid-project

When you come back to the project the next day, the pattern is simpler:

> Read `docs/stages/CURRENT_STAGE.md` and continue. First, summarize what's already done (check git log) and what's left to finish this stage.

Claude Code will pick up the thread. The stage files + CURRENT_STAGE.md pointer make this reliable.

## Troubleshooting: Claude Code isn't using the skills

Symptoms: Claude Code writes code that violates a skill's conventions (wrong return type, writes to naive stack, uses regex instead of sqlglot).

Diagnose:

1. In the session, ask: *"List the skills you have access to in this project."* If your four skills aren't listed, the discovery is broken.
2. Check the directory structure: skills must be at `.claude/skills/<skill-name>/SKILL.md` exactly. The `SKILL.md` filename is fixed.
3. Check the frontmatter: each `SKILL.md` needs a YAML frontmatter with `name:` and `description:` fields at the top.
4. If listing works but a skill wasn't consulted on a relevant task, the skill's `description` may not be descriptive enough. Good descriptions are trigger-focused: *"Use when ..."* / *"Trigger for ..."*. Vague descriptions like *"Helps with gateways"* don't activate reliably.

## Troubleshooting: Stages being skipped or rushed

If Claude Code wants to start Stage 2 work while Stage 1 isn't done, stop and say: *"`docs/stages/CURRENT_STAGE.md` says STAGE_1. Only work on tasks from STAGE_1.md. Do not begin STAGE_2.md until I advance the pointer."*

The stage discipline is the single most important thing for a long project. Protect it.

## A note on Ollama specifically

Once you reach Stage 1, the `ollama-nl2sql` skill has the full porting guide for your existing Gradio code. The short version:

1. Extract the pure prompt→SQL function from the Gradio code.
2. Wrap it in FastAPI following the skill's contract.
3. Delete the Gradio UI code. The SvelteKit dashboard is now the only UI.
4. Run the Stage 1 integration smoke test to confirm the port works.

Don't let Claude Code skip step 3. Keeping Gradio around "just in case" tempts future drift.

## A note on the proposal PDF

Drop your academic proposal PDF at `docs/PROPOSAL.pdf`. `CLAUDE.md` references it. When Claude Code is making an architectural decision, it should be able to cite proposal sections (§ 2.1, § 2.5, etc.) as justification. This matters for your final report's proposal-mapping document in Stage 7.

## First-session checklist

- [ ] Repo initialized
- [ ] Instruction docs in place
- [ ] `.claude/skills/` directory with all four skills
- [ ] Proposal PDF at `docs/PROPOSAL.pdf`
- [ ] Initial commit made
- [ ] Ollama prototype stashed in `_prior_art/` (gitignored)
- [ ] Claude Code session started
- [ ] First message sent and response verified before any code is written

When all boxes are checked, you're ready for Stage 0.

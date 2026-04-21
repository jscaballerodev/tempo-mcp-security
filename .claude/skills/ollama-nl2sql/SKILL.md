---
name: ollama-nl2sql
description: Use when building, modifying, or debugging the NL→SQL microservice at stack-*/nl2sql/ that converts natural-language prompts into SQL using a local Ollama model. Covers the service contract (FastAPI endpoint shape), deterministic generation settings, schema-aware prompting, how to port existing Gradio-based code into the FastAPI service, and the critical constraint that the service output is TREATED AS UNTRUSTED by the gateway. Trigger for any work on nl2sql/ or when the dashboard needs to call the NL→SQL layer.
---

# Ollama NL→SQL Microservice

The NL→SQL service is a **separate microservice**, not part of the gateway. It converts user prompts to SQL using Ollama. Its output is untrusted input to the gateway, which re-parses, re-validates, and re-authorizes everything.

## Service contract

FastAPI app at `stack-hardened/nl2sql/` (and an identical copy or shared service used by the naive stack).

**Endpoint:** `POST /generate`

**Request:**
```json
{
  "prompt": "how much did we spend on marketing last quarter?",
  "schema_hint": "<optional schema snippet>",
  "role_hint": "marketing_analyst"
}
```

**Response:**
```json
{
  "sql": "SELECT SUM(spend) FROM app.marketing_campaigns WHERE start_date >= '2026-01-01'",
  "model": "llama3.1:8b-instruct-q5_K_M",
  "temperature": 0.0,
  "latency_ms": 842,
  "prompt_hash": "sha256:..."
}
```

**Errors:** HTTP 400 if Ollama produces no parseable SQL; 500 on Ollama unavailable.

## Deterministic generation

The service's job is to be as reproducible as possible. Default parameters:

```python
OLLAMA_OPTIONS = {
    "temperature": 0.0,
    "top_p": 1.0,
    "seed": 42,           # if Ollama supports; ignored otherwise
    "num_predict": 512,
    "stop": [";\n", "```"],
}
```

Don't stream the response. Return only after the full SQL is produced so the service can validate it parses before returning.

## System prompt template

Store at `stack-hardened/nl2sql/prompts/system.md`:

```
You are a SQL generator for the TEMPO data platform. Produce a single PostgreSQL
SELECT statement that answers the user's question, using only tables documented
in the schema hint. Do not produce multiple statements, comments, DDL, or DML.
Do not wrap the SQL in markdown or explain it. Output only the SQL.

Schema:
{schema_hint}

User question:
{prompt}
```

The service appends the user's question as a separate user message. System-user separation matters because it reduces prompt-injection surface (though the gateway is the real defense).

## Schema hint construction

`stack-hardened/nl2sql/schema.py`:

- Read `app.table_sensitivity` once at startup (and refresh on SIGHUP).
- Build a schema hint that includes **only tables the user's role may access**. An `hr_admin` sees HR tables; a `marketing_analyst` does not see them in the hint.
- This is a **usability** feature, not a security control. The gateway is what enforces access. But hiding out-of-scope tables reduces the chance the model proposes a forbidden query in the first place.

```python
def build_schema_hint(role: str, catalog: dict) -> str:
    allowed_tiers = ROLE_TIER_CAPS[role]  # duplicated from Rego for hints only
    visible_tables = [
        t for t, tier in catalog.items() if tier in allowed_tiers
    ]
    return render_schema(visible_tables, catalog)
```

Keep in mind: this mapping duplicates policy logic. Drift between the hint-side caps and the Rego caps is a usability bug (confused users) but not a security bug (Rego is the source of truth).

## Porting existing Gradio code

If your existing prototype is a Gradio app, extract in this order:

1. **Identify the pure function** in the Gradio code that takes a prompt and returns SQL. This becomes `nl2sql/generator.py`.
2. **Strip UI code.** Gradio decorators, `gr.Blocks`, event handlers — all of it goes.
3. **Replace Gradio state with FastAPI state.** If the original used `gr.State`, you probably don't need it; requests are stateless.
4. **Wrap in FastAPI.** `nl2sql/main.py` has the endpoint; `nl2sql/generator.py` has the logic.
5. **Add response validation.** Before returning, parse the SQL with `sqlglot.parse_one(sql, dialect="postgres")`. If it fails, retry once, then return 400.
6. **Delete the Gradio code.** Don't keep it "just in case." The dashboard (SvelteKit) is the only UI from now on. Leaving Gradio around tempts future confusion about which is canonical.

A minimal scaffold:

```python
# nl2sql/main.py
from fastapi import FastAPI, HTTPException
from nl2sql.generator import generate_sql
from nl2sql.schema import build_schema_hint
from nl2sql.models import GenerateRequest, GenerateResponse

app = FastAPI(title="TEMPO NL2SQL")

@app.post("/generate", response_model=GenerateResponse)
async def generate(req: GenerateRequest) -> GenerateResponse:
    hint = build_schema_hint(req.role_hint, CATALOG)
    try:
        result = await generate_sql(req.prompt, hint)
    except OllamaUnavailable:
        raise HTTPException(503, "nl2sql backend unavailable")
    if not result.parsed_ok:
        raise HTTPException(400, "could not produce valid SQL")
    return result
```

## The service is shared between naive and hardened

Both stacks call the same NL→SQL service. The security difference is entirely in what happens *after* the SQL is produced:

- **Naive stack:** dashboard → nl2sql → naive MCP → Postgres. No gatekeeping.
- **Hardened stack:** dashboard → nl2sql → gateway → hardened MCP → Postgres. Gateway re-parses SQL, enforces policies, audits.

This is a *feature* of the architecture. It demonstrates that the NL→SQL layer's security properties are irrelevant — the gateway assumes SQL is adversarial regardless of source. Make this point explicitly in the final report.

## Ollama model selection

Default: `llama3.1:8b-instruct-q5_K_M`. Reasons:
- Small enough for local dev without a GPU (runs on CPU, slower but works).
- Good enough at SQL generation for demo-quality queries.
- Open weights, reproducible across environments.

For better quality during evaluation: `llama3.1:70b-instruct-q4_K_M` if hardware allows. For attack 4 (prompt injection), test against multiple models — some resist injection better than others, and the finding is "at least one capable model fell for it."

Pin the model tag in `.env.example`:
```
NL2SQL_MODEL=llama3.1:8b-instruct-q5_K_M
```

## Don't put an LLM inside the gateway

The gateway is deterministic. Never import Ollama or any LLM client under `stack-hardened/gateway/`. If you find yourself reaching for an LLM in a decision path, stop — the gateway's defenses are all rule-based by design. This is a documented architectural constraint in `CLAUDE.md`.

## Reference documents

- `CLAUDE.md` § "Technology Constraints"
- `docs/stages/STAGE_1.md` — initial NL→SQL microservice setup
- Ollama docs: https://github.com/ollama/ollama/blob/main/docs/api.md

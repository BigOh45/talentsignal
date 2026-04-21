# TalentSignal — Full Build Prompt
# Copy this entire message to Claude / GitHub Copilot

---

## CONTEXT

I am building **TalentSignal** — an AI-powered resume triage system for enterprise HR.

The system scores CVs against job descriptions using a multi-stage LLM funnel, produces evidence cards (not black-box scores), and integrates with Workday for verified candidate data.

This is a **local development setup**. No cloud storage. CVs and SKILL.md files live in the local repo. Only LLM calls go to Azure AI Foundry.

---

## TECH STACK

```
Language:        Python 3.11+
LLM:             Azure AI Foundry (Claude Haiku 3.5 + Sonnet 4)
Database:        PostgreSQL (Docker)
Cache:           Redis (Docker)
CV Parsing:      pdfplumber, python-docx, pytesseract, Pillow, pdf2image
Validation:      Pydantic v2
API:             FastAPI
Config:          python-dotenv + YAML
Skill matching:  rapidfuzz
```

---

## PROJECT STRUCTURE

Build exactly this structure:

```
talentsignal/
├── cvs/
│   └── .gitkeep
├── skills/
│   ├── data-engineering/
│   │   └── SKILL.md
│   ├── ai-engineering/
│   │   └── SKILL.md
│   └── generic/
│       └── SKILL.md
├── src/
│   ├── __init__.py
│   ├── config.py         ← env vars + settings
│   ├── db.py             ← PostgreSQL connection + table creation
│   ├── loader.py         ← SKILL.md loading from local folder + Redis cache
│   ├── parser.py         ← CV text extraction (PDF/DOCX/OCR/Vision)
│   ├── stages.py         ← Stage 1 hard rules + Stage 2 skill matching
│   ├── scorer.py         ← Stage 3 quick LLM + Stage 4 full evidence card
│   ├── worker.py         ← orchestrates full pipeline for a job_id
│   └── api.py            ← FastAPI endpoints
├── tests/
│   ├── test_setup.py     ← verify all connections work
│   └── test_pipeline.py  ← end to end test with a sample CV
├── .env.example
├── .gitignore
├── docker-compose.yml
└── requirements.txt
```

---

## ENVIRONMENT VARIABLES

```bash
# .env

# Azure AI Foundry
AZURE_AI_ENDPOINT=https://your-project.services.ai.azure.com/models
AZURE_AI_KEY=your-key-here

# Models
HAIKU_MODEL=claude-haiku-3-5
SONNET_MODEL=claude-sonnet-4-0

# Local paths
CVS_DIR=./cvs
SKILLS_DIR=./skills

# PostgreSQL
DATABASE_URL=postgresql://tsuser:tspass@localhost:5432/talentsignal

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
```

---

## DOCKER COMPOSE

```yaml
version: "3.9"

services:
  postgres:
    image: postgres:15
    container_name: ts_postgres
    environment:
      POSTGRES_USER: tsuser
      POSTGRES_PASSWORD: tspass
      POSTGRES_DB: talentsignal
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    container_name: ts_redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

---

## DATABASE SCHEMA

```sql
CREATE TABLE IF NOT EXISTS jobs (
    job_id          TEXT PRIMARY KEY,
    title           TEXT NOT NULL,
    jd_text         TEXT,
    skill_md_key    TEXT NOT NULL,
    status          TEXT DEFAULT 'OPEN',
    created_at      TIMESTAMP DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS applications (
    app_id          TEXT PRIMARY KEY,
    job_id          TEXT REFERENCES jobs(job_id),
    cv_filename     TEXT,
    cv_parsed       JSONB,
    score           FLOAT,
    score_detail    JSONB,
    why_yes         JSONB,
    why_no          JSONB,
    ai_detection    JSONB,
    recommendation  TEXT DEFAULT 'PENDING',
    status          TEXT DEFAULT 'PENDING',
    applied_at      TIMESTAMP DEFAULT NOW(),
    scored_at       TIMESTAMP
);
```

---

## SKILL.MD FORMAT

Each SKILL.md has two sections separated by `---`:

1. **YAML frontmatter** — machine-readable config for Stage 1 and Stage 2
2. **Prose body** — LLM system prompt for Stage 3 and Stage 4

Example `skills/data-engineering/SKILL.md`:

```yaml
---
name: resume-scorer-data-engineering
category: data-engineering
version: 1.0

hard_filters:
  min_yoe: 3
  work_auth_required: false
  locations_allowed: [UK, Remote, India, US]

skill_taxonomy:
  python:
    must_have: true
    aliases: [python3, pandas, numpy, scripting]
  spark:
    must_have: true
    aliases: [pyspark, databricks, distributed computing, apache spark]
  sql:
    must_have: true
    aliases: [postgresql, mysql, hive, presto, bigquery, redshift]
  kafka:
    must_have: false
    aliases: [kinesis, pulsar, event streaming, rabbitmq]
  airflow:
    must_have: false
    aliases: [prefect, dagster, luigi, workflow orchestration]
  dbt:
    must_have: false
    aliases: [data build tool, dbt core, dbt cloud]

must_have_skills: [python, spark, sql]
min_overlap: 0.6
---

## Role
You are a senior technical recruiter scoring CVs for
Data Engineering roles at XYZ organisation.
Score based on evidence only. Be specific —
reference actual content from the CV, never generic phrases.
Return structured JSON using the score_cv tool.

## Scoring Dimensions

skills_match (30pts):
  Required skills from JD found in CV.
  Adjacent skills get 70% credit not zero.

experience_relevance (25pts):
  Domain alignment and seniority match.
  Penalise if YoE less than 70% of JD requirement.

impact_evidence (20pts):
  Quantified outcomes in work history.
  Specific metrics beats vague responsibilities.

tech_stack_depth (15pts):
  Modern tooling and cloud native stack.

education (10pts):
  Degree relevance plus certifications.

## Hard Disqualifiers
- FAIL if YoE less than 50% of JD minimum
- FAIL if zero relevant tech stack overlap

## Evidence Rules
Reference actual content from CV.
Never say "strong background" without citing evidence.
```

---

## MODULE SPECIFICATIONS

### src/config.py

Load all env vars using pydantic-settings or python-dotenv.
Expose as a single `Settings` object imported everywhere.

---

### src/db.py

- `get_connection()` — returns psycopg2 connection
- `create_tables()` — creates jobs + applications tables
- `get_job(job_id)` — returns job row as dict
- `get_pending_cvs(job_id)` — returns list of PENDING applications
- `save_application(app_id, job_id, cv_filename)` — inserts PENDING row
- `save_result(app_id, result_dict)` — updates score, recommendation, status

---

### src/loader.py

`SkillConfig` class:
- Parses raw SKILL.md text
- Splits YAML frontmatter from prose body on `---` separator
- Exposes: `hard_filters`, `skill_taxonomy`, `must_have`, `min_overlap`, `prose`, `raw_text`, `version`, `name`

`load_skill_md(skill_md_key: str) -> SkillConfig`:
- Checks Redis cache first (key: `skill_md:{skill_md_key}`)
- On miss: loads from `{SKILLS_DIR}/{skill_md_key}/SKILL.md`
- Falls back to `{SKILLS_DIR}/generic/SKILL.md` if not found
- Caches in Redis with TTL 3600 seconds
- Returns SkillConfig object

---

### src/parser.py

`extract_raw_text(file_bytes: bytes, filename: str) -> tuple[str, str]`:
- Returns `(text, method)` where method is one of: pdf, docx, ocr, plaintext
- PDF: use pdfplumber. If extracted text < 100 chars → fall back to OCR
- DOCX: use python-docx
- Scanned PDF: use pytesseract via pdf2image

`parse_cv_azure(raw_text: str, client) -> dict`:
- Calls Claude Haiku via Azure AI Foundry
- Uses tool calling to force structured output
- Tool name: `parse_cv`
- Returns structured dict with these fields:

```python
{
  "full_name": str,
  "email": str,
  "location": str,
  "work_auth_stated": bool,
  "total_yoe": float,
  "current_title": str,
  "skills": list[str],
  "work_history": list[{
    "company": str,
    "title": str,
    "start_year": int,
    "end_year": int,
    "duration_months": int,
    "achievements": list[str]
  }],
  "education": list[{
    "degree": str,
    "field": str,
    "institution": str,
    "year": int
  }],
  "certifications": list[str],
  "parse_confidence": float,   # 0.0 to 1.0
  "parse_issues": list[str],
  "_source": str               # pdf/docx/ocr/vision
}
```

`parse_cv_image_azure(file_bytes: bytes, media_type: str, client) -> dict`:
- For image CVs and designer CVs
- Sends image directly to Claude Sonnet Vision
- Same tool calling approach, same output schema

`parse_cv(file_bytes: bytes, filename: str, client) -> dict`:
- Master parser — routes to correct method based on file extension
- NEVER silently drops a CV
- On any exception returns dict with `_needs_review: True`
- Low parse_confidence (< 0.6) sets `_needs_review: True`

---

### src/stages.py

`hard_filter(cv: dict, hard_filters: dict) -> tuple[bool, list[str]]`:
- Returns `(passed, failure_reasons)`
- Checks: min_yoe (50% floor), work_auth_required, locations_allowed
- Uses VERIFIED data where available, CV data as fallback
- Failures are plain English strings for recruiter display

`semantic_skill_score(cv: dict, skill_config: SkillConfig) -> tuple[float, list[str]]`:
- Returns `(score_0_to_10, missing_skills_list)`
- For each must_have skill: check if skill OR any alias appears in cv["skills"]
- Use rapidfuzz.fuzz.partial_ratio >= 80 for fuzzy matching
- Adjacent skill found: award 70% credit, flag as "adjacent" not "missing"
- Score = (matched_must_have / total_must_have) * 10
- missing_skills is list of plain English gap descriptions

---

### src/scorer.py

#### Stage 3 — Quick LLM Score

`quick_llm_score(cv: dict, jd_text: str, skill_config: SkillConfig, client) -> dict`:
- Model: Haiku
- Short prompt ~300 tokens — seniority fit + career trajectory only
- Returns:

```python
{
  "score": int,            # 1-10
  "seniority_fit": str,    # UNDER | MATCH | OVER
  "one_line_reason": str
}
```

#### Stage 4 — Full Evidence Card

Tool definition for `score_cv`:

```python
{
  "composite_score": int,           # 0-100
  "dimensions": {
    "skills_match":          {"score": int, "reason": str},
    "experience_relevance":  {"score": int, "reason": str},
    "impact_evidence":       {"score": int, "reason": str},
    "tech_stack_depth":      {"score": int, "reason": str},
    "education":             {"score": int, "reason": str}
  },
  "why_yes": list[str],             # top 3 specific reasons candidate fits
  "why_no": list[str],              # top 3 specific concerns or gaps
  "ai_written_assessment": {
    "likelihood": str,              # LOW | MEDIUM | HIGH
    "signals": list[str],
    "verdict": str
  },
  "recommendation": str,            # SHORTLIST | HUMAN_REVIEW | DEPRIORITISE
  "summary": str                    # one sentence for recruiter
}
```

`full_evidence_card(cv_text: str, jd_text: str, skill_config: SkillConfig, client) -> dict`:
- Model: Sonnet 4
- System prompt = skill_config.prose (the SKILL.md prose body)
- User message = f"JD:\n{jd_text}\n\nCV:\n{cv_text}"
- Uses score_cv tool calling
- Returns structured evidence card dict

---

### src/worker.py

`score_job(job_id: str)`:
- Load job context once: `get_job(job_id)` → get `jd_text` + `skill_md_key`
- Load SKILL.md once: `load_skill_md(skill_md_key)`
- Fetch all PENDING CVs: `get_pending_cvs(job_id)`
- For each CV:
  1. Load CV file from `{CVS_DIR}/{job_id}/{cv_filename}`
  2. Parse CV → `parse_cv()`
  3. If `_needs_review` → save as `PARSE_REVIEW`, skip scoring
  4. Stage 1 → `hard_filter()` → if fail save as `DEPRIORITISED`
  5. Stage 2 → `semantic_skill_score()` → if < 3 save as `DEPRIORITISED`
  6. Stage 3 → `quick_llm_score()` → if < 4 save as `DEPRIORITISED`
  7. Stage 4 → `full_evidence_card()` → save full result
  8. Recommendation:
     - score >= 75 → SHORTLIST
     - score 45-74 → HUMAN_REVIEW
     - score < 45  → DEPRIORITISED
- Print progress per CV
- Print summary at end: total / shortlisted / review / deprioritised / failed

---

### src/api.py

FastAPI app with these endpoints:

```
POST /jobs
  body: {job_id, title, jd_text, skill_md_key}
  → inserts job row

POST /jobs/{job_id}/applications
  body: multipart form — job_id + cv_file upload
  → saves CV file to cvs/{job_id}/
  → inserts PENDING application row
  → returns app_id

POST /jobs/{job_id}/score
  → triggers score_job(job_id) as background task
  → returns {"status": "scoring started", "job_id": job_id}

GET /jobs/{job_id}/results
  → returns all scored applications for job
  → ordered by score descending
  → includes full evidence card per candidate

GET /jobs/{job_id}/results/{app_id}
  → returns single evidence card

GET /health
  → checks DB + Redis + Azure AI connections
  → returns {"status": "ok"} or error details
```

---

### tests/test_setup.py

Test all four connections independently:
- Redis: set and get a key
- PostgreSQL: SELECT 1
- SKILL.md loader: load data-engineering, verify hard_filters not empty
- Azure AI Foundry: send "Say OK", verify response contains "ok"

Print ✅ for each passing test.
Print ❌ with error message for each failing test.
Print final summary.

---

### tests/test_pipeline.py

End to end test:
1. Create a test job in DB with skill_md_key = "data-engineering"
2. Use a hardcoded CV text string (no real file needed)
3. Run parse_cv_azure on the text
4. Run hard_filter on parsed result
5. Run semantic_skill_score on parsed result
6. Run quick_llm_score
7. Run full_evidence_card
8. Print full result as formatted JSON
9. Assert composite_score is between 0 and 100
10. Assert recommendation is one of SHORTLIST/HUMAN_REVIEW/DEPRIORITISED

---

## IMPORTANT RULES FOR IMPLEMENTATION

1. **Azure AI Foundry tool calling format** — use `azure-ai-inference` SDK:

```python
from azure.ai.inference import ChatCompletionsClient
from azure.ai.inference.models import (
    SystemMessage,
    UserMessage,
    ChatCompletionsToolDefinition,
    FunctionDefinition
)
from azure.core.credentials import AzureKeyCredential

client = ChatCompletionsClient(
    endpoint=os.getenv("AZURE_AI_ENDPOINT"),
    credential=AzureKeyCredential(os.getenv("AZURE_AI_KEY"))
)

# Tool call
response = client.complete(
    model=os.getenv("HAIKU_MODEL"),
    messages=[UserMessage(content=prompt)],
    tools=[tool_definition],
    tool_choice="parse_cv"
)

# Extract result
tool_call = response.choices[0].message.tool_calls[0]
result = json.loads(tool_call.function.arguments)
```

2. **Never silently drop a CV** — always save a record even if parse fails

3. **SKILL.md loading** — split on `---` to separate YAML from prose.
   YAML goes to stages.py for rules.
   Prose goes to scorer.py as system prompt.

4. **skill_md_key mapping** — comes from jobs table column `skill_md_key`.
   Load SKILL.md using this key:
   `skills/{skill_md_key}/SKILL.md`

5. **Confidence threshold** — if parse_confidence < 0.6, set status = PARSE_REVIEW.
   Never score a low-confidence parse.

6. **All reasons in plain English** — why_yes, why_no, failure_reasons
   must be readable by a non-technical recruiter.

7. **No hardcoded model names** — always use env vars HAIKU_MODEL and SONNET_MODEL.

8. **Local file paths** — CVs at `{CVS_DIR}/{job_id}/{cv_filename}`.
   SKILL.md at `{SKILLS_DIR}/{skill_md_key}/SKILL.md`.

9. **Pydantic v2** for all request/response models in api.py.

10. **requirements.txt** must be complete and pinned to specific versions.

---

## DELIVERABLES

Build all files listed in the project structure.
Every file must be complete and runnable.
No placeholders like `# TODO implement this`.
No stub functions that just `pass`.

After building, provide:
1. Exact commands to set up and run locally:
   ```bash
   docker-compose up -d
   python src/db.py
   python tests/test_setup.py
   python tests/test_pipeline.py
   uvicorn src.api:app --reload
   ```

2. Example curl commands to test the API:
   ```bash
   # Create job
   curl -X POST http://localhost:8000/jobs ...
   
   # Upload CV
   curl -X POST http://localhost:8000/jobs/JOB-001/applications ...
   
   # Trigger scoring
   curl -X POST http://localhost:8000/jobs/JOB-001/score
   
   # Get results
   curl http://localhost:8000/jobs/JOB-001/results
   ```

3. Sample SKILL.md files for all three categories:
   - data-engineering
   - ai-engineering
   - generic

---

## WHAT THIS IS NOT

- Not a real-time system — scoring is async/triggered
- Not using vector databases or embeddings
- Not using LangChain — direct Azure AI Foundry SDK only
- Not using cloud storage — all files are local
- Not using MCP tools — tool calling here means Azure AI function calling
- Not a frontend — API only for now

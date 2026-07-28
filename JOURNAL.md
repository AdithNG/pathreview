# Contribution Journal — Module 3

## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/148

**Issue title:** Skill extractor fails to detect JavaScript and TypeScript

**Tier:** [x] Tier 1  [ ] Tier 2  [ ] Tier 3

**Problem summary:**
The ingestion pipeline's skill extractor (`ingestion/parsers/skill_extractor.py`) is supposed to detect technologies from resume and portfolio text, but the JavaScript/TypeScript family is essentially invisible to it. The class defines a `JS_TS_KEYWORDS` constant that is never actually used — JS/TS detection relies only on the optional `filename` argument (which is `None` for free text) and an import regex that requires whitespace after the keyword, so real-world code like `require('fs')` never matches. The same class of bug affects Docker detection: `_detect_tools` only looks for the literal word "docker", so a Dockerfile (`FROM`/`RUN`/`EXPOSE`) or a docker-compose YAML is never recognized. A successful fix makes the four failing tests in `tests/unit/test_skill_extractor.py` pass by detecting JS/TS from language keywords and syntax in the text itself, and Docker from Dockerfile/compose structure — without changing the Python, database, or framework detection that already works. This matters because skill detection feeds the RAG feedback pipeline, so a portfolio full of JavaScript work currently gets reviewed as if those skills don't exist.

**Branch name:** `fix/148-skill-extractor-js-ts-detection`

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [x] Issue added to cohort ledger

### Issue-fit checklist notes (scope reasoning)

- **Understanding:** I can reproduce the bug directly — `SkillExtractor().extract_skills('Wrote index.js using const arrow functions')` returns `[]`. I traced each of the four failing tests to a specific root cause (unused `JS_TS_KEYWORDS` set, filename-only detection, `\b(import|require)\s+` regex missing `require(`, and literal-word-only Docker matching).
- **Tier fit:** Tier 1 and my first contribution to a large codebase — the change lives in one implementation file plus its test file.
- **Codebase readiness:** I read the full implementation and test file. The parser `SkillExtractor` is imported only by its own tests (verified by grep), so the change is isolated and the tests define the acceptance contract. Note: there is a separate `SkillExtractor` agent tool in `agent/tools/` that this issue does *not* touch.
- **Scope and time:** Estimated 3–6 hours (keyword/regex logic plus tests) — realistic for the Week 8–9 window. Three other students have claimed the issue, which is on the low end for tier-1 issues in this cohort; claims are non-exclusive.
- **Blockers:** None. I verified that no open PR modifies `ingestion/parsers/skill_extractor.py` (PR #162 touches the tech detector, chunker, faithfulness checker, and PII scrubber — different files).
- **Extra finding while reading:** `tests/unit/test_skill_extractor.py:138` has a pre-existing bug of its own (`skill_names = [s.name for s in skill_names]` — a `NameError` from referencing the variable being defined). It's out of scope for #148 but worth flagging.

### Environment setup notes

Setup on Windows hit four stacked issues before `make setup` succeeded, all worth documenting:

1. Missing `.env` — the app's built-in default `DATABASE_URL` points at port 5432; copying `.env.example` to `.env` (which uses 5433) is required.
2. Port conflicts from other projects: a native Windows PostgreSQL service on 5432, plus leftover Docker containers from another project squatting 5433 and 6379 (they auto-start with Docker Desktop and had to be stopped).
3. The pathreview db container was first created while its port was taken, so it needed `docker compose up -d --force-recreate db` to bind correctly.
4. `scripts/seed_db.py` prints ✓/✗ characters that crash on the default Windows cp1252 console, masking real errors — running with `PYTHONUTF8=1 make setup` fixes it.

Verified working: frontend at localhost:5173 (HTTP 200), API docs at localhost:8000/docs, and a successful login with a seeded test account returning a JWT.

---

## Week 8 — Reproduction & solution planning

**Reproduction commit link:** _(this commit — link added below once pushed)_

**Reproduction summary:**
I reproduced the issue two ways in my local environment: by running
`pytest tests/unit/test_skill_extractor.py`, which fails 5 of 18 tests (the 4 named in the
issue plus one unrelated pre-existing test bug), and by calling `extract_skills()` directly
on the samples from the issue body. JavaScript text returns an empty list, TypeScript text
returns only `['React']`, and Dockerfile/compose text returns no Docker detection at all —
matching the reported behavior exactly.

**PLAN.md link:** _(added below once pushed)_

**Walkthrough video (recommended):** _(not yet recorded)_

**Blockers or open questions:**
- `test_database_technology_detection` also fails, but it is **not** one of the four tests
  named in issue #148. It fails first on its own bug (`skill_names = [s.name for s in skill_names]`
  at line 138 — a `NameError`), and I confirmed that even after fixing that line the assertion
  still fails, because `psycopg2` is not in the `DATABASES` map (only `postgresql` is). That
  means it needs an implementation change of its own. I plan to leave it out of my PR and
  mention it in the PR description rather than expand scope, but I may ask the maintainer
  whether they'd prefer it included.
- I need to decide how strict the new JavaScript keyword matching should be. Detection that is
  too loose creates false positives (see the `import psycopg2` finding below); too strict and
  the issue's own sample text won't be detected.

### Reproduction steps and observed output

```
$ PYTHONUTF8=1 .venv/Scripts/python -m pytest tests/unit/test_skill_extractor.py -v
FAILED tests/unit/test_skill_extractor.py::TestSkillExtractor::test_text_with_typescript_files
FAILED tests/unit/test_skill_extractor.py::TestSkillExtractor::test_database_technology_detection
FAILED tests/unit/test_skill_extractor.py::TestSkillExtractor::test_devops_tool_detection
FAILED tests/unit/test_skill_extractor.py::TestSkillExtractor::test_javascript_detection
FAILED tests/unit/test_skill_extractor.py::TestSkillExtractor::test_docker_compose_detection
5 failed, 13 passed in 1.30s
```

Calling the extractor directly (`SkillExtractor().extract_skills(text)`):

| Input | Expected | Observed |
|---|---|---|
| `Wrote index.js using const arrow functions and async/await callbacks` | JavaScript | `[]` |
| `Built app.tsx and types.ts with strict TypeScript interfaces` | TypeScript | `['React']` |
| `const fs = require('fs'); ... console.log(data)` | JavaScript | `[]` |
| `export interface User { id: string; }` (TS) | TypeScript | `['Python']` |
| `FROM python:3.9 / RUN pip install / EXPOSE 8000` (Dockerfile) | Docker | `['Python']` |
| `version: '3.8' / services: / build: .` (compose) | Docker | `[]` |

### Root causes confirmed during reproduction

All five located in `ingestion/parsers/skill_extractor.py`:

1. **`JS_TS_KEYWORDS` is dead code** (lines 30–41). I grepped the whole repo: the set is
   defined and never referenced. `PYTHON_KEYWORDS` is unused in the same way, but Python
   still gets detected through other signals, which is why only JS/TS visibly breaks.
2. **`require(` never matches.** `_detect_languages` uses `re.search(r"\b(import|require)\s+", text)`,
   which demands whitespace after the keyword, so `require('fs')` fails to match.
3. **TypeScript vs JavaScript is decided only by filename** (line 186:
   `lang = "TypeScript" if ".ts" in str(filename or "").lower() else "JavaScript"`). For free
   text with no filename — the normal case for resume/portfolio content — TypeScript can never
   be detected, even when the text says "TypeScript" and names `.ts` files.
4. **Docker is matched as a literal word only.** `_detect_tools` does `if tool in text_lower`,
   so a real Dockerfile (`FROM`/`RUN`/`EXPOSE`) or a compose file (`services:`/`build:`) is
   never recognized as Docker.
5. **False positive found while reproducing:** plain Python `import psycopg2` is currently
   reported as **JavaScript**, because the shared `\b(import|require)\s+` regex matches Python
   import statements too. So the current behavior is both missing real JS and inventing fake JS.

A sixth, related quirk: the Python type-annotation regex `:\s*(int|str|float|bool|list|dict)`
matches TypeScript's `: string` (because `str` is a prefix of `string`), which is why the
TypeScript sample above is misreported as Python.

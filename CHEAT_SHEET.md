# GermanCards — Session Cheat Sheet

## What files to attach per session

| Session | Attach | Expected output |
|---------|--------|-----------------|
| 1 | MASTER_SPEC.md | Scaffold, Docker, DB models, Alembic, frontend skeleton |
| 2 | MASTER_SPEC.md | Auth, Pydantic schemas, normalization, input parser |
| 3 | MASTER_SPEC.md | LLM adapters, generation pipeline, canonical lookup |
| 4 | MASTER_SPEC.md | FSRS reviews, collection, decks, export |
| 5 | MASTER_SPEC.md | Complete frontend (all pages) |
| 6 | MASTER_SPEC.md | Security, tests, Docker prod config, README |

---

## How to start each session in Claude Code

```
claude "Read the attached MASTER_SPEC.md fully.
Then read SESSION_PROMPTS.md and follow Session N exactly.
Do not build anything outside the scope of Session N."
```

Or in the Claude Code chat, paste:
1. The contents of MASTER_SPEC.md
2. Then the specific session block from SESSION_PROMPTS.md

---

## When Claude Code goes wrong

### It invents a different tech stack
→ Paste Section 1 from MASTER_SPEC.md and say: "Use this stack exactly."

### It simplifies the DB schema
→ Paste Section 4 from MASTER_SPEC.md and say: "Implement this schema exactly, no changes."

### It skips the dual-LLM audit step
→ Paste Section 9 from MASTER_SPEC.md and say: "The pipeline has 6 steps. Implement all of them."

### It treats sentence inputs as canonical cards
→ Paste Section 7, paragraph "Sentence input behavior" and say: "Sentences are never canonical. Re-read this."

### It uses localStorage for tokens
→ Say: "Store the access token in React context memory only. httpOnly cookie for refresh token. See Section 14."

### It generates but doesn't validate LLM output
→ Paste Section 9, Steps 2-3 and say: "Pydantic validation + rule checks are mandatory before the audit step."

---

## Test commands to run after each session

After Session 1:
```bash
docker compose up -d
cd backend && alembic upgrade head
python scripts/create_admin.py --email admin@test.com --password test123
uvicorn app.main:app --reload
# Check: http://localhost:8000/health returns {"status": "ok"}
```

After Session 2:
```bash
cd backend && python -m pytest tests/test_normalize.py tests/test_input_parser.py -v
# All tests must pass before proceeding
```

After Session 3:
```bash
curl -X POST http://localhost:8000/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@test.com","password":"test123"}'
# Should return access_token
```

After Session 4:
```bash
# With token from above:
curl -X POST http://localhost:8000/v1/input/resolve \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"input_mode":"raw_text","type_final":"word","target_term":"laufen",
       "target_term_lookup":"laufen","context_sentence":null,
       "pos_choice":"verb","gender_hint":"none",
       "locks":{"type_locked":false,"term_locked":false,"pos_locked":true,"gender_locked":false}}'
# Should trigger generation pipeline and return card
```

After Session 5:
```bash
cd frontend && npm run dev
# Walk through: login → add "laufen" → study session → check export
```

After Session 6:
```bash
cd backend && python -m pytest tests/ -v
# All tests pass. Then run full Docker build.
docker compose up --build
```

---

## Seed data commands (after Session 4+)

To seed the top words into the canonical library:

```bash
curl -X POST http://localhost:8000/v1/admin/jobs/batch-generate \
  -H "Authorization: Bearer ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "words": [
      {"word": "laufen", "pos": "verb"},
      {"word": "Haus", "pos": "noun", "gender": "das"},
      {"word": "schnell", "pos": "adj"},
      {"word": "in Frage kommen", "pos": null}
    ],
    "deck_name": "Test Seed"
  }'
```

---

## Cost estimates per session

| Session | LLM calls | Est. cost |
|---------|-----------|-----------|
| 1-4 | 0 (backend only) | $0 |
| 5 | 0 (frontend only) | $0 |
| 6 test run | ~5-10 test cards generated | ~$0.01 |
| Seeding 100 words | 200 LLM calls (gen + audit) | ~$0.10-0.20 |
| Seeding 1000 words | 2000 LLM calls | ~$1-2 |

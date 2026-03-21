# GermanCards — Master Specification (V1)

> This document is the single source of truth for the app.
> When in doubt, follow this spec exactly. Do not invent, assume, or simplify.
> Every session prompt will reference this file. Read it fully before writing any code.

---

## 0. Project Identity

- **Name:** GermanCards
- **Purpose:** German vocabulary flashcard app with a canonical shared card library, FSRS scheduling, and dual-LLM generation + audit pipeline.
- **V1 scope:** Web app only. Single user (private). No Anki integration. No audio/images yet (architecture must support them later).
- **API-first:** All business logic lives in the backend. Frontend is a thin client. Future clients (Telegram bot, mobile) reuse the same API without changes.

---

## 1. Tech Stack (non-negotiable)

### Backend
- **Language:** Python 3.12
- **Framework:** FastAPI (async)
- **ORM:** SQLAlchemy 2.0 async (`asyncpg` driver)
- **Migrations:** Alembic
- **Validation:** Pydantic v2 (used everywhere, especially for LLM output)
- **Auth:** JWT (access token 15min, refresh token 30 days via httpOnly cookie). Passwords: bcrypt.
- **Rate limiting:** `slowapi`
- **LLM SDKs:** `openai` (>=1.0) and `anthropic` (>=0.25)
- **FSRS:** `fsrs` package (pip install fsrs)
- **Background tasks:** FastAPI `BackgroundTasks` (no Celery in V1)
- **Config:** `pydantic-settings` reading from `.env` file

### Frontend
- **Framework:** React 18 + Vite
- **Styling:** Tailwind CSS
- **HTTP client:** `axios` with a central client instance
- **State:** React `useState` / `useReducer` (no Redux in V1)
- **Routing:** React Router v6

### Database
- **PostgreSQL 15** (local via Docker Compose, production on Railway)
- **Redis:** included in Docker Compose for future use (rate limit storage in V1)

### Storage (V2 media — columns exist, not populated in V1)
- Cloudflare R2 (S3-compatible). Backend has a stub `MediaService` that does nothing in V1 but has the right interface.

### Deployment
- **Platform:** Railway.app
- **Local dev:** Docker Compose (postgres + redis) + `uvicorn` hot-reload + Vite dev server

---

## 2. Repository Structure

```
germanCards/
├── backend/
│   ├── app/
│   │   ├── main.py                  # FastAPI app, router registration, middleware
│   │   ├── config.py                # pydantic-settings config
│   │   ├── database.py              # async engine, session factory, Base
│   │   ├── models/                  # SQLAlchemy ORM models (one file per table group)
│   │   │   ├── canonical.py         # canonical_cards, canonical_aliases
│   │   │   ├── user.py              # users
│   │   │   ├── collection.py        # user_cards, user_card_examples
│   │   │   ├── scheduling.py        # card_schedules, reviews
│   │   │   ├── decks.py             # decks, deck_cards, user_decks
│   │   │   └── admin.py             # admin_jobs
│   │   ├── schemas/                 # Pydantic schemas (request/response shapes)
│   │   │   ├── auth.py
│   │   │   ├── cards.py
│   │   │   ├── input_parser.py      # NormalizedRequest and related
│   │   │   ├── reviews.py
│   │   │   ├── decks.py
│   │   │   └── card_content.py      # NounContent, VerbContent, AdjContent
│   │   ├── routers/                 # FastAPI routers
│   │   │   ├── auth.py
│   │   │   ├── input.py             # POST /v1/input/resolve
│   │   │   ├── cards.py
│   │   │   ├── reviews.py
│   │   │   ├── decks.py
│   │   │   ├── export.py
│   │   │   └── admin.py
│   │   ├── services/
│   │   │   ├── normalize.py         # German text normalization
│   │   │   ├── input_parser.py      # Implements the full input parsing spec
│   │   │   ├── canonical.py         # Canonical lookup + alias resolution
│   │   │   ├── generation.py        # LLM generation pipeline
│   │   │   ├── audit.py             # LLM audit step
│   │   │   ├── llm/
│   │   │   │   ├── base.py          # Abstract LLMAdapter interface
│   │   │   │   ├── openai_adapter.py
│   │   │   │   └── anthropic_adapter.py
│   │   │   ├── scheduler.py         # FSRS wrapper
│   │   │   ├── collection.py        # User card management
│   │   │   ├── media.py             # Stub MediaService (V2)
│   │   │   └── export.py            # CSV export
│   │   ├── middleware/
│   │   │   ├── auth.py              # JWT verification dependency
│   │   │   └── security.py          # Security headers
│   │   └── utils/
│   │       ├── german.py            # Irregular verb list, known genders, etc.
│   │       └── exceptions.py        # Custom exception classes
│   ├── alembic/
│   │   └── versions/                # Migration files
│   ├── tests/
│   │   ├── test_normalize.py
│   │   ├── test_input_parser.py
│   │   └── test_generation_pipeline.py
│   ├── scripts/
│   │   └── create_admin.py          # python scripts/create_admin.py --email x --password y
│   ├── Dockerfile
│   ├── requirements.txt
│   └── .env.example
├── frontend/
│   ├── src/
│   │   ├── api/                     # axios API wrappers
│   │   ├── components/              # Reusable UI components
│   │   ├── pages/                   # Route-level page components
│   │   ├── hooks/                   # Custom React hooks
│   │   └── App.jsx
│   ├── index.html
│   ├── vite.config.js
│   └── package.json
├── docker-compose.yml
└── README.md
```

---

## 3. Environment Variables (.env.example)

```
# App
SECRET_KEY=changeme-generate-with-secrets-token-hex-32
ALLOW_REGISTRATION=false
ENVIRONMENT=development

# Database
DATABASE_URL=postgresql+asyncpg://germanCards:pass@localhost:5432/germanCards

# Redis
REDIS_URL=redis://localhost:6379

# LLM Providers
OPENAI_API_KEY=
ANTHROPIC_API_KEY=

# LLM Config
GENERATOR_PROVIDER=openai        # openai | anthropic
GENERATOR_MODEL=gpt-4o-mini
AUDITOR_PROVIDER=anthropic       # must differ from generator
AUDITOR_MODEL=claude-haiku-4-5-20251001
AUDIT_PASS_THRESHOLD=0.85

# Rate limits
MAX_GENERATIONS_PER_USER_PER_DAY=50

# Media (V2 — leave empty in V1)
R2_ACCOUNT_ID=
R2_ACCESS_KEY_ID=
R2_SECRET_ACCESS_KEY=
R2_BUCKET_NAME=
```

---

## 4. Database Schema (implement exactly)

```sql
-- ─── CANONICAL LIBRARY ───────────────────────────────────────────────────────

CREATE TABLE canonical_cards (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  lookup_key       TEXT UNIQUE NOT NULL,
  -- e.g. "laufen|word|verb", "in frage kommen|phrase"
  word_display     TEXT NOT NULL,
  -- Original casing: "laufen", "in Frage kommen"
  type             TEXT NOT NULL CHECK (type IN ('word', 'phrase')),
  pos              TEXT CHECK (pos IN ('noun', 'verb', 'adj')),
  gender           TEXT CHECK (gender IN ('der', 'die', 'das')),
  schema_version   INT NOT NULL DEFAULT 1,
  content          JSONB NOT NULL,
  audit_status     TEXT NOT NULL DEFAULT 'pending'
                   CHECK (audit_status IN ('pending', 'approved', 'needs_human_review', 'rejected')),
  audit_score      FLOAT,
  audit_notes      TEXT,
  generator_model  TEXT,
  auditor_model    TEXT,
  audio_key        TEXT,   -- null in V1, R2 key in V2
  image_key        TEXT,   -- null in V1, R2 key in V2
  source           TEXT NOT NULL DEFAULT 'llm'
                   CHECK (source IN ('llm', 'manual', 'import')),
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE canonical_aliases (
  alias_key        TEXT PRIMARY KEY,
  canonical_id     UUID NOT NULL REFERENCES canonical_cards(id) ON DELETE CASCADE,
  alias_type       TEXT NOT NULL CHECK (alias_type IN ('umlaut', 'case', 'spacing', 'manual'))
);

-- ─── USERS ───────────────────────────────────────────────────────────────────

CREATE TABLE users (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email            TEXT UNIQUE NOT NULL,
  hashed_pw        TEXT NOT NULL,
  is_admin         BOOLEAN NOT NULL DEFAULT FALSE,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  settings         JSONB NOT NULL DEFAULT '{}'
);

-- ─── USER COLLECTION ─────────────────────────────────────────────────────────

CREATE TABLE user_cards (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id          UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  canonical_id     UUID NOT NULL REFERENCES canonical_cards(id),
  content_override JSONB,  -- null = use canonical content as-is
  added_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(user_id, canonical_id)
);

CREATE TABLE user_card_examples (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id          UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  canonical_id     UUID NOT NULL REFERENCES canonical_cards(id),
  example_de       TEXT NOT NULL,
  example_en       TEXT,
  added_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ─── FSRS SCHEDULING ─────────────────────────────────────────────────────────

CREATE TABLE card_schedules (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_card_id     UUID NOT NULL REFERENCES user_cards(id) ON DELETE CASCADE UNIQUE,
  stability        FLOAT NOT NULL DEFAULT 0,
  difficulty       FLOAT NOT NULL DEFAULT 5,
  due_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_review_at   TIMESTAMPTZ,
  reps             INT NOT NULL DEFAULT 0,
  lapses           INT NOT NULL DEFAULT 0,
  state            TEXT NOT NULL DEFAULT 'new'
                   CHECK (state IN ('new', 'learning', 'review', 'relearning'))
);

-- Immutable log — never update rows, only insert
CREATE TABLE reviews (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_card_id     UUID NOT NULL REFERENCES user_cards(id) ON DELETE CASCADE,
  rating           SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 4),
  reviewed_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  elapsed_ms       INT
);

-- ─── DECKS ───────────────────────────────────────────────────────────────────

CREATE TABLE decks (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name             TEXT NOT NULL,
  description      TEXT,
  source           TEXT,
  is_public        BOOLEAN NOT NULL DEFAULT TRUE,
  created_by       UUID REFERENCES users(id),
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE deck_cards (
  deck_id          UUID REFERENCES decks(id) ON DELETE CASCADE,
  canonical_id     UUID REFERENCES canonical_cards(id) ON DELETE CASCADE,
  position         INT,
  PRIMARY KEY (deck_id, canonical_id)
);

CREATE TABLE user_decks (
  user_id          UUID REFERENCES users(id) ON DELETE CASCADE,
  deck_id          UUID REFERENCES decks(id) ON DELETE CASCADE,
  added_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (user_id, deck_id)
);

-- ─── ADMIN / PIPELINE ────────────────────────────────────────────────────────

CREATE TABLE admin_jobs (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  job_type         TEXT NOT NULL,
  status           TEXT NOT NULL DEFAULT 'pending'
                   CHECK (status IN ('pending', 'running', 'completed', 'failed')),
  input_data       JSONB,
  result_data      JSONB,
  error            TEXT,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  completed_at     TIMESTAMPTZ
);

-- ─── INDEXES ─────────────────────────────────────────────────────────────────

CREATE INDEX idx_canonical_lookup    ON canonical_cards(lookup_key);
CREATE INDEX idx_canonical_audit     ON canonical_cards(audit_status);
CREATE INDEX idx_aliases_key         ON canonical_aliases(alias_key);
CREATE INDEX idx_user_cards_user     ON user_cards(user_id);
CREATE INDEX idx_schedules_due       ON card_schedules(due_at)
  WHERE due_at <= now();
CREATE INDEX idx_reviews_user_card   ON reviews(user_card_id, reviewed_at);
```

---

## 5. JSONB Content Schemas (schema_version=1)

These are the structures stored in `canonical_cards.content`. Pydantic models in `schemas/card_content.py` must match exactly.

### Noun
```json
{
  "article": "der",
  "plural": "die Berge",
  "meanings": ["mountain", "peak"],
  "usage_context": "Natural elevation; very common in compound words.",
  "compounds": ["das Gebirge", "der Bergsteiger", "das Bergdorf"],
  "synonyms": [],
  "antonyms": [],
  "example_sentences": [
    {"de": "Der Berg ist über 3000 Meter hoch.", "en": "The mountain is over 3000 metres tall."}
  ]
}
```

### Verb
```json
{
  "meanings": ["to run", "to walk", "to function"],
  "conjugations": {
    "present_3sg": "läuft",
    "perfekt": "ist gelaufen",
    "praeteritum": "lief"
  },
  "object_usage": "intransitive; 'durch + Akk' (durch den Park laufen)",
  "collocations": ["schnell laufen", "laufen lassen", "Gefahr laufen"],
  "synonyms": [],
  "antonyms": [],
  "example_sentences": [
    {"de": "Er läuft jeden Morgen durch den Park.", "en": "He runs through the park every morning."}
  ]
}
```

### Adjective
```json
{
  "meanings": ["fast", "quick", "rapid"],
  "declension_hint": "schnell → schneller Berg (strong masc. nom.)",
  "typical_combinations": ["schnelles Auto", "schnelle Entscheidung", "schnell handeln"],
  "synonyms": [],
  "antonyms": ["langsam"],
  "example_sentences": [
    {"de": "Er ist ein sehr schneller Läufer.", "en": "He is a very fast runner."}
  ]
}
```

### Phrase
```json
{
  "meanings": ["to come into question", "to be an option"],
  "usage_notes": "Often used in negative: 'Das kommt nicht in Frage.'",
  "register": "neutral",
  "collocations": ["nicht in Frage kommen", "kaum in Frage kommen"],
  "example_sentences": [
    {"de": "Das kommt für mich nicht in Frage.", "en": "That is not an option for me."}
  ]
}
```

Rules:
- `synonyms` and `antonyms`: empty array `[]` unless truly strong and unambiguous. Max 2 each.
- `example_sentences`: min 1, max 2.
- All string values: no markdown, no HTML, no code fences.
- `meanings`: max 3 items, each max 50 chars.

---

## 6. German Normalization (implement in `services/normalize.py`)

```python
import re
import unicodedata

UMLAUT_MAP = str.maketrans({
    'ä': 'ae', 'ö': 'oe', 'ü': 'ue',
    'Ä': 'ae', 'Ö': 'oe', 'Ü': 'ue',
    'ß': 'ss',
})

def to_lookup_key(text: str) -> str:
    """Maximally normalized key for cache lookup."""
    text = text.strip().lower()
    text = re.sub(r'\s+', ' ', text)
    text = text.translate(UMLAUT_MAP)
    text = unicodedata.normalize('NFC', text)
    return text

def to_display_form(text: str) -> str:
    """Minimal normalization for display — preserve umlauts, collapse spaces."""
    text = text.strip()
    text = re.sub(r'\s+', ' ', text)
    text = unicodedata.normalize('NFC', text)
    return text

def generate_aliases(display_form: str) -> list[str]:
    """
    Returns all lookup key variants for a display form.
    All should be stored in canonical_aliases pointing to same canonical card.
    """
    variants = set()
    variants.add(to_lookup_key(display_form))          # full umlaut expansion
    direct_lower = re.sub(r'\s+', ' ', display_form.strip().lower())
    direct_lower = unicodedata.normalize('NFC', direct_lower)
    variants.add(direct_lower)                          # raw lowercase (ä not expanded)
    return list(v for v in variants if v)
```

---

## 7. Input Parsing Spec (implement in `services/input_parser.py`)

The parser converts raw user input into a `NormalizedRequest` object.

### Separators (in priority order)
1. ` — ` (em dash with spaces)
2. ` - ` (hyphen with spaces — note spaces required to avoid splitting "e-mail")
3. `|`
4. `:` ONLY if left side is <= 25 chars (to avoid splitting normal sentences)

### Mode B split valid if
- `len(term_hint_raw)` between 1 and 60
- `len(context_sentence_raw)` >= 6
- `context_sentence_raw` contains at least one whitespace

### Type inference (when not locked by user)
- Has `.`, `?`, `!` at end → `word_or_phrase` (extract core term) — sentences never go to canonical library
- Has 4+ words with no punctuation → `phrase`
- Has whitespace → `phrase`
- No whitespace → `word`

### Sentence input behavior
Sentence-type inputs are NOT stored as sentence cards in the canonical library.
Instead: extract the most likely target term, generate/lookup a word or phrase card for it,
and store the full sentence in `user_card_examples`. The frontend should ask the user
"We'll create a card for [term]. Does that look right?" before proceeding.

### Output: NormalizedRequest
```python
class NormalizedRequest(BaseModel):
    input_mode: Literal["term_with_context", "raw_text"]
    type_final: Literal["word", "phrase"]
    target_term: str                    # always present (extracted from sentence if needed)
    target_term_lookup: str             # to_lookup_key(target_term)
    context_sentence: str | None        # stored in user_card_examples if present
    pos_choice: Literal["noun", "verb", "adj", "not_sure"]
    gender_hint: Literal["der", "die", "das", "none"]
    locks: Locks

class Locks(BaseModel):
    type_locked: bool
    term_locked: bool
    pos_locked: bool
    gender_locked: bool
```

---

## 8. Canonical Lookup Flow

```
lookup(normalized_request) -> CanonicalLookupResult

1. Build candidate keys:
   key = target_term_lookup + "|" + type_final
   if type_final == "word": key += "|" + pos_choice  (if pos locked)
   if pos == "noun" and gender locked: key += "|gender:" + gender_hint

2. Check canonical_aliases for alias_key = key → get canonical_id
   OR check canonical_cards.lookup_key = key directly

3. If single match → return HIT(canonical_card)

4. If multiple matches (e.g. essen|word|verb AND essen|word|noun, pos not locked)
   → return NEEDS_USER_CHOICE(candidates=[...])

5. If no match → return MISS → trigger generation pipeline

Result type:
{
  "result": "HIT" | "MISS" | "NEEDS_USER_CHOICE",
  "canonical_card": {...} | null,
  "candidates": [...] | null    # only for NEEDS_USER_CHOICE
}
```

---

## 9. Generation Pipeline (implement in `services/generation.py` + `services/audit.py`)

```
Input: NormalizedRequest + pos (resolved) + gender (if noun)

Step 1 — GENERATOR LLM
  provider: GENERATOR_PROVIDER (from config)
  model: GENERATOR_MODEL
  Use structured output / tool call forcing JSON matching content schema.
  System prompt must include:
    - Exact JSON schema expected
    - Rules: no markdown, max meanings, min 1 example sentence
    - Example sentences MUST contain the target word
    - synonyms/antonyms only if truly strong (otherwise empty array)

Step 2 — PYDANTIC VALIDATION
  Parse raw JSON into NounContent / VerbContent / AdjContent / PhraseContent
  ValidationError → retry Step 1 once with error injected in prompt
  Still failing → go to Step 5c

Step 3 — RULE CHECKS (deterministic, fast)
  - article must be exactly "der"/"die"/"das" for nouns
  - perfekt must contain "hat" or "ist" for verbs
  - praeteritum must not equal present_3sg (catches hallucinations)
  - at least one example sentence must contain the target word (substring match on lowercased conjugated forms)
  - no field contains "<", ">", "```", "*", "#"

Step 4 — AUDITOR LLM
  provider: AUDITOR_PROVIDER (must differ from generator)
  model: AUDITOR_MODEL
  System prompt: "You are a German linguistics expert auditing a flashcard for accuracy.
  Check: article correctness, plural plausibility, Partizip II accuracy, Präteritum accuracy,
  example sentence grammaticality and naturalness, meaning completeness and accuracy.
  Return ONLY valid JSON: {score: float 0-1, issues: [string], approved: bool}"
  User message: full card JSON + target word + POS

Step 5 — DECISION
  score >= AUDIT_PASS_THRESHOLD (0.85) → save canonical_card(status='approved')
  score 0.6-0.85 → retry generation once with audit issues injected → re-audit
                    if approved → save; else → save(status='needs_human_review')
  score < 0.6 → save(status='needs_human_review')
  
  In all cases: create user_card linking user to canonical_card.
  Return canonical_card to API (even if needs_human_review — user can edit).

Step 6 — ALIAS REGISTRATION
  After saving canonical_card, call generate_aliases(word_display)
  Insert all aliases into canonical_aliases (ignore conflicts).
```

---

## 10. API Endpoints (all under `/v1/`)

All responses use envelope: `{"data": ..., "error": null, "meta": {}}`.
All endpoints require `Authorization: Bearer <token>` except `/v1/auth/*`.

```
POST   /v1/auth/register        body: {email, password} — only if ALLOW_REGISTRATION=true
POST   /v1/auth/login           body: {email, password} → {access_token}
POST   /v1/auth/refresh         (reads httpOnly refresh cookie) → {access_token}
POST   /v1/auth/logout          clears refresh cookie

POST   /v1/input/resolve        body: NormalizedRequest input (from frontend parser)
                                → ResolveResult (HIT | MISS+generation | NEEDS_USER_CHOICE)
POST   /v1/input/confirm-choice body: {canonical_id, context_sentence?}
                                → adds card to user collection, returns user_card

GET    /v1/cards                ?page&limit&pos&search → paginated user_cards with content
GET    /v1/cards/:id            single user_card with merged content + user examples
PUT    /v1/cards/:id            body: {content_override: partial JSONB}
DELETE /v1/cards/:id            removes from user collection (does not delete canonical)

GET    /v1/reviews/due          ?limit=20 → due user_cards sorted by due_at
POST   /v1/reviews              body: {user_card_id, rating: 1|2|3|4, elapsed_ms?}
                                → updated schedule, next_due
GET    /v1/reviews/stats        → {due_today, reviewed_today, streak, retention_7d}

GET    /v1/decks                → public decks + user's decks
GET    /v1/decks/:id            → deck with card list
POST   /v1/decks/:id/subscribe  → adds all deck cards to user collection
DELETE /v1/decks/:id/subscribe  → removes deck cards from user collection

GET    /v1/export/csv           → CSV download (Anki-compatible)

# Admin only (requires is_admin=true)
GET    /v1/admin/queue          → canonical_cards where audit_status='needs_human_review'
PUT    /v1/admin/cards/:id      body: {content, audit_status} → approve/edit canonical card
POST   /v1/admin/jobs/batch-generate  body: {words: [{word, pos, gender?}], deck_name?}
GET    /v1/admin/jobs/:id       → job status
```

---

## 11. FSRS Integration

Use the `fsrs` Python package. Wrap it in `services/scheduler.py`.

```python
from fsrs import FSRS, Card, Rating, State

scheduler = FSRS()

def process_review(schedule_row, rating_int: int) -> dict:
    """
    rating_int: 1=Again, 2=Hard, 3=Good, 4=Easy
    Returns updated fields to write back to card_schedules.
    """
    card = Card(
        stability=schedule_row.stability,
        difficulty=schedule_row.difficulty,
        due=schedule_row.due_at,
        last_review=schedule_row.last_review_at,
        reps=schedule_row.reps,
        lapses=schedule_row.lapses,
        state=State[schedule_row.state.capitalize()],
    )
    rating = {1: Rating.Again, 2: Rating.Hard, 3: Rating.Good, 4: Rating.Easy}[rating_int]
    new_card, review_log = scheduler.review_card(card, rating)
    return {
        "stability": new_card.stability,
        "difficulty": new_card.difficulty,
        "due_at": new_card.due,
        "last_review_at": review_log.review,
        "reps": new_card.reps,
        "lapses": new_card.lapses,
        "state": new_card.state.name.lower(),
    }
```

---

## 12. LLM Adapter Interface

```python
# services/llm/base.py
from abc import ABC, abstractmethod
from pydantic import BaseModel

class LLMMessage(BaseModel):
    role: str   # "system" | "user" | "assistant"
    content: str

class LLMAdapter(ABC):
    @abstractmethod
    async def generate_structured(
        self,
        messages: list[LLMMessage],
        response_schema: type[BaseModel],
        temperature: float = 0.3,
    ) -> BaseModel:
        """Call LLM and return parsed Pydantic model. Raise on failure."""
        ...

    @abstractmethod
    async def generate_text(
        self,
        messages: list[LLMMessage],
        temperature: float = 0.3,
        max_tokens: int = 500,
    ) -> str:
        """Call LLM and return raw text. Used for audit."""
        ...
```

Both `OpenAIAdapter` and `AnthropicAdapter` implement this interface.
- `generate_structured` uses OpenAI structured outputs (`response_format`) or Anthropic tool calling.
- Factory function in `services/llm/__init__.py`: `get_adapter(provider: str) -> LLMAdapter`.

---

## 13. Frontend Pages (V1)

```
/login                  LoginPage — email + password form
/                       DashboardPage — due count, streak, quick actions
/study                  StudyPage — flashcard flip + rating UI
/cards                  CardListPage — browse, search, filter by POS
/cards/:id              CardDetailPage — full card view + edit
/add                    AddCardPage — input form + clarification flow
/decks                  DecksPage — browse + subscribe to decks
/admin/queue            AdminQueuePage — review needs_human_review cards (admin only)
```

### AddCardPage flow
1. Text input field (free text)
2. Optional UI dropdowns: Type (word/phrase/not sure), POS (noun/verb/adj/not sure), Gender (der/die/das/none)
3. On submit → `POST /v1/input/resolve`
4. If HIT → show card preview → "Add to my cards" button
5. If NEEDS_USER_CHOICE → show 2-3 buttons for disambiguation → user picks → `POST /v1/input/confirm-choice`
6. If MISS (generation) → show loading spinner → poll or wait → show generated card with "Saved" confirmation
7. If `needs_review=true` on returned card → show inline edit UI

### StudyPage (flashcard UI)
- Shows word (front) with POS and gender badge
- Click/tap anywhere to flip
- Back shows full card content
- 4 rating buttons: Again (1) / Hard (2) / Good (3) / Easy (4)
- Keyboard shortcuts: Space = flip, 1/2/3/4 = rate
- Progress bar: X of N cards reviewed this session
- Shows user context sentence first (if present), then canonical examples

---

## 14. Security Requirements

- All LLM-bound input sanitized: strip tags, limit to 120 chars, reject if contains "ignore previous", "system prompt", or 3+ consecutive special chars.
- Rate limit: `POST /v1/input/resolve` → 20 req/min per user, 50 generations/day per user.
- JWT secret from env var — never hardcoded.
- ALLOW_REGISTRATION defaults to `false` — admin created only via script.
- Security headers middleware: X-Content-Type-Options, X-Frame-Options, CSP.
- All DB queries via SQLAlchemy ORM — no raw string interpolation.
- Refresh token stored as httpOnly, SameSite=Strict cookie — not in localStorage.

---

## 15. CSV Export Format (Anki-compatible)

```
#separator:tab
#html:false
#notetype:German Vocabulary
#deck:German Vocabulary
Word\tPOS\tArticle\tPlural\tMeanings\tConjugations\tExample_DE\tExample_EN\tCollocations\tCompounds
```

Route: `GET /v1/export/csv`
Filename: `germanCards-export-YYYY-MM-DD.txt`

---

## 16. What is NOT in V1 (do not build)

- Audio / TTS
- Images
- Anki APKG export
- AnkiConnect sync
- Multi-user registration (admin only)
- Telegram bot
- Mobile apps
- Gamification / leaderboards
- Public deck sharing by users
- Statistics charts / graphs (basic counts only)
- Offline PWA mode
- Sentence-type canonical cards (sentences → extract term → word/phrase card)

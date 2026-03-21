# GermanCards — Claude Code Session Prompts

How to use this file:
- Run sessions IN ORDER. Each session assumes the previous is complete.
- At the start of EVERY session, attach MASTER_SPEC.md to the conversation.
- Copy the session prompt verbatim. Do not paraphrase it.
- After each session: review the code, run the tests, fix issues before moving on.
- If Claude Code gets stuck or makes a wrong decision, paste the relevant section
  from MASTER_SPEC.md directly into the chat as a correction.

═══════════════════════════════════════════════════════════════════════════════
SESSION 1 — Project Scaffold + Database
═══════════════════════════════════════════════════════════════════════════════

ATTACH: MASTER_SPEC.md

---

Read MASTER_SPEC.md fully before writing any code.

Your task in this session is ONLY: project scaffold, Docker setup, database layer.
Do NOT build any API routes, services, or frontend yet.

Build the following:

1. Create the full directory structure from Section 2 of the spec.
   Create empty placeholder files (with a one-line comment) for everything
   that will be built later. This gives us the full skeleton immediately.

2. `docker-compose.yml`
   - PostgreSQL 15 service (db) with volume, env vars from .env
   - Redis service
   - No app container yet (we run uvicorn locally in dev)

3. `backend/requirements.txt` with these exact packages:
   fastapi uvicorn[standard] sqlalchemy[asyncio] asyncpg alembic pydantic pydantic-settings
   python-jose[cryptography] bcrypt passlib slowapi openai anthropic fsrs python-multipart
   httpx pytest pytest-asyncio

4. `backend/.env.example` — copy exactly from Section 3 of the spec.

5. `backend/app/config.py`
   - Use pydantic-settings BaseSettings
   - Load all vars from Section 3
   - Single `get_settings()` function with lru_cache

6. `backend/app/database.py`
   - Async SQLAlchemy engine using DATABASE_URL from settings
   - AsyncSession factory
   - Base declarative class
   - `get_db()` async dependency for FastAPI

7. `backend/app/models/` — implement ALL SQLAlchemy ORM models from the SQL schema in Section 4.
   - canonical.py: CanonicalCard, CanonicalAlias
   - user.py: User
   - collection.py: UserCard, UserCardExample
   - scheduling.py: CardSchedule, Review
   - decks.py: Deck, DeckCard, UserDeck
   - admin.py: AdminJob
   All models must include all columns and constraints from the schema.
   Use `mapped_column` and `Mapped` typing (SQLAlchemy 2.0 style).

8. `backend/alembic/` — init Alembic, create initial migration that creates all tables.
   The migration must be auto-generated from the models (not hand-written).

9. `backend/scripts/create_admin.py`
   CLI script: `python scripts/create_admin.py --email admin@example.com --password secret`
   Creates a user with is_admin=True. Uses bcrypt. Checks if email already exists.

10. `backend/app/main.py` — minimal FastAPI app:
    - Include CORS middleware (allow localhost:5173 in dev)
    - Include security headers middleware (stub for now)
    - Register all routers (even if routers are empty stubs)
    - Health check: GET /health → {"status": "ok"}

11. `frontend/` — scaffold with Vite + React + Tailwind:
    Run: `npm create vite@latest frontend -- --template react`
    Then install: tailwindcss postcss autoprefixer react-router-dom axios
    Configure Tailwind. Create placeholder files for all pages listed in Section 13.
    App.jsx should set up React Router with routes for all pages (just render
    placeholder divs for now).

At the end, provide instructions for:
- Running `docker compose up -d` to start Postgres + Redis
- Running `alembic upgrade head` to create tables
- Running `python scripts/create_admin.py` to create admin user
- Running `uvicorn app.main:app --reload` to start backend
- Running `npm run dev` to start frontend

Do not proceed to build any API logic. That is Session 2.

═══════════════════════════════════════════════════════════════════════════════
SESSION 2 — Auth + Core Schemas + Normalization + Input Parser
═══════════════════════════════════════════════════════════════════════════════

ATTACH: MASTER_SPEC.md

---

Read MASTER_SPEC.md fully before writing any code.
The project scaffold from Session 1 is complete. All models, DB, and directory
structure exist. Do NOT modify the DB models or migrations.

Your task: Auth system, Pydantic schemas, German normalization, and input parser.

1. `backend/app/schemas/card_content.py`
   Pydantic v2 models for EXACTLY the content structures in Section 5:
   - ExampleSentence(de: str, en: str | None)
   - NounContent, VerbContent, AdjContent, PhraseContent
   Each must have field validators enforcing the rules from Section 5:
   - meanings: max 3 items, each max 50 chars (validator trims silently)
   - example_sentences: min 1, max 2
   - synonyms/antonyms: max 2 each
   - No markdown/HTML: validator strips <, >, ```, *, # from all str fields
   Also create a union type: CardContent = NounContent | VerbContent | AdjContent | PhraseContent

2. `backend/app/schemas/auth.py` — LoginRequest, TokenResponse, UserResponse

3. `backend/app/schemas/input_parser.py`
   - Locks, NormalizedRequest (from Section 7)
   - ResolveResult with discriminated union:
     - HIT: result="HIT", canonical_card, user_card_id (if already in collection)
     - MISS: result="MISS", canonical_card (after generation), just_generated=True
     - NEEDS_USER_CHOICE: result="NEEDS_USER_CHOICE", candidates=[{canonical_id, lookup_key, pos, gender, preview}]

4. `backend/app/schemas/cards.py` — UserCardResponse (merged canonical + override + user examples)

5. `backend/app/schemas/reviews.py` — ReviewRequest, ReviewResponse, StatsResponse

6. `backend/services/normalize.py`
   Implement EXACTLY as specified in Section 6. Include:
   - to_lookup_key(text) → str
   - to_display_form(text) → str
   - generate_aliases(display_form) → list[str]
   
   Write tests in `backend/tests/test_normalize.py` covering:
   - "Straße" and "Strasse" and "strasse" all produce same lookup key
   - "ä" and "ae" produce same lookup key
   - "Essen" and "essen" produce same lookup key
   - Spaces are collapsed
   - generate_aliases("Straße") returns both "strasse" and "straße"

7. `backend/services/input_parser.py`
   Implement EXACTLY as specified in Section 7. Rules:
   - Separator detection in priority order from spec
   - Colon only if left side <= 25 chars
   - Mode B split criteria from spec
   - Type inference rules from spec
   - Sentence inputs: type_final is always "word" or "phrase" (never sentence)
     For sentence input: extract the most content-rich noun/verb as target_term
     Use a simple heuristic: last capitalized word (likely noun) or first verb form
     Store original sentence as context_sentence
   - Returns NormalizedRequest
   
   Write tests in `backend/tests/test_input_parser.py` covering ALL 4 examples
   from the original spec document (see examples section) plus:
   - "Straße: das ist eine lange Straße." → Mode B, target="Straße"
   - "schnell" → word, no context
   - "in Frage kommen" → phrase
   - "Das ist eine Straße." → sentence → extract "Straße", context_sentence set

8. `backend/app/middleware/auth.py`
   - JWT creation: create_access_token(user_id, email, is_admin)
   - JWT creation: create_refresh_token(user_id) — stored as httpOnly cookie
   - FastAPI dependency: get_current_user(token) → User ORM object
   - FastAPI dependency: require_admin(user) → raises 403 if not is_admin

9. `backend/app/routers/auth.py`
   Implement:
   - POST /v1/auth/login — verify email+password, return access token, set refresh cookie
   - POST /v1/auth/refresh — read refresh cookie, return new access token
   - POST /v1/auth/logout — clear refresh cookie
   - POST /v1/auth/register — only if ALLOW_REGISTRATION=true in settings, else 403

Run all tests at the end. All must pass.

═══════════════════════════════════════════════════════════════════════════════
SESSION 3 — LLM Adapters + Generation Pipeline + Canonical Lookup
═══════════════════════════════════════════════════════════════════════════════

ATTACH: MASTER_SPEC.md

---

Read MASTER_SPEC.md fully before writing any code.
Sessions 1 and 2 are complete. Auth works, schemas exist, normalization tested.

Your task: LLM adapters, generation + audit pipeline, canonical lookup service.

1. `backend/app/services/llm/base.py`
   Implement LLMAdapter ABC EXACTLY as in Section 12.

2. `backend/app/services/llm/openai_adapter.py`
   Implement OpenAIAdapter(LLMAdapter):
   - generate_structured: use openai client with response_format={"type": "json_object"}
     Parse result into provided Pydantic model. Raise ValueError on parse failure.
   - generate_text: standard completion, return text.

3. `backend/app/services/llm/anthropic_adapter.py`
   Implement AnthropicAdapter(LLMAdapter):
   - generate_structured: use Anthropic tool calling to force JSON output matching schema.
     Define the schema as a tool with the Pydantic model's JSON schema as input_schema.
   - generate_text: standard message, return text.

4. `backend/app/services/llm/__init__.py`
   Factory: `get_adapter(provider: str) -> LLMAdapter`
   provider: "openai" → OpenAIAdapter; "anthropic" → AnthropicAdapter

5. `backend/app/services/generation.py`
   Implement the full pipeline from Section 9 (Steps 1-6).
   
   System prompts (embed directly in this file as constants):
   
   GENERATOR_SYSTEM_NOUN = """
   You are a German linguistics expert creating flashcard content.
   Generate a flashcard for the given German noun.
   Return ONLY valid JSON matching this exact schema — no markdown, no explanation:
   {
     "article": "der|die|das",
     "plural": "die [Pluralform]",
     "meanings": ["meaning1", "meaning2"],  // max 3, each max 50 chars
     "usage_context": "short context string, max 100 chars",
     "compounds": ["compound1", "compound2"],  // max 3
     "synonyms": [],  // only truly strong synonyms, max 2, else empty
     "antonyms": [],  // only truly strong antonyms, max 2, else empty
     "example_sentences": [
       {"de": "German sentence containing the word.", "en": "English translation."}
     ]  // min 1, max 2. Sentence MUST contain the target word.
   }
   Rules: no markdown, no HTML, no code fences in any field value.
   """
   
   (Create similar constants for VERB, ADJ, PHRASE)
   
   AUDIT_SYSTEM = """
   You are a German linguistics expert auditing a flashcard for factual accuracy.
   You will receive a flashcard JSON and the target word + POS.
   Check: article correctness, plural plausibility, Partizip II accuracy,
   Präteritum accuracy, example sentence grammaticality and naturalness,
   meaning accuracy and completeness, no hallucinated compounds.
   Return ONLY valid JSON, no markdown:
   {"score": 0.0-1.0, "issues": ["issue1", "issue2"], "approved": true|false}
   approved=true only if score >= 0.85 and no critical errors.
   """
   
   Input sanitization (before any LLM call):
   - Strip leading/trailing whitespace
   - Reject if len > 120
   - Reject if contains any of: "ignore previous", "system prompt", "forget",
     "disregard", "jailbreak", or 3+ consecutive special characters
   - Raise ValueError("Input rejected: potential injection") on rejection

6. `backend/app/services/audit.py`
   Implements Step 4 of the pipeline (separated for testability):
   - audit_card(content_dict, target_word, pos) → AuditResult(score, issues, approved)
   - Uses AUDITOR_PROVIDER adapter
   - Returns AuditResult with parsed score and issues
   - If LLM returns unparseable response: return AuditResult(score=0.0, issues=["Parse error"], approved=False)

7. `backend/app/services/canonical.py`
   Implement canonical lookup from Section 8:
   - async lookup(db, normalized_request) → LookupResult
   - Build candidate keys based on locks
   - Check canonical_aliases first, then canonical_cards.lookup_key
   - Return HIT / MISS / NEEDS_USER_CHOICE

8. Wire everything into `backend/app/routers/input.py`:
   - POST /v1/input/resolve
     1. Parse + validate body as NormalizedRequest
     2. Call canonical.lookup()
     3. If HIT: check if user already has this card, return ResolveResult(HIT)
     4. If MISS: call generation pipeline (as FastAPI BackgroundTask? No — run inline
        with timeout. Generation is synchronous for V1. Return result when done.)
     5. If NEEDS_USER_CHOICE: return candidates
   - POST /v1/input/confirm-choice
     body: {canonical_id: str, context_sentence: str | None}
     Creates user_card + card_schedule(state=new) + user_card_example if context given.
     Returns UserCardResponse.

   Rate limiting: apply @limiter.limit("20/minute") to /resolve.

Write one integration test (mock LLM calls): test the full resolve flow for a cache miss.

═══════════════════════════════════════════════════════════════════════════════
SESSION 4 — User Collection, FSRS Reviews, Decks, Export
═══════════════════════════════════════════════════════════════════════════════

ATTACH: MASTER_SPEC.md

---

Read MASTER_SPEC.md fully before writing any code.
Sessions 1-3 complete. Generation pipeline works. Input resolver works.

Your task: user card collection management, FSRS review system, decks, CSV export.

1. `backend/app/services/scheduler.py`
   Implement process_review() EXACTLY as in Section 11.
   Also implement:
   - get_due_cards(db, user_id, limit=20) → list[UserCard with joined CardSchedule]
     Order by: state='new' last (show new cards only when review pile is empty),
     otherwise order by due_at ASC.
   - init_schedule(db, user_card_id) → creates CardSchedule with defaults

2. `backend/app/services/collection.py`
   - add_card(db, user_id, canonical_id, context_sentence=None) → UserCard
     Creates user_card + card_schedule(state=new) + user_card_example if context.
     If user_card already exists: return existing (idempotent).
   - get_card_with_content(db, user_id, user_card_id) → UserCardResponse
     Merge: canonical content + content_override (if set) + user_card_examples.
     Override merge: content_override fields replace canonical fields (shallow merge).
   - remove_card(db, user_id, user_card_id) → None (delete user_card, cascade handles rest)
   - list_cards(db, user_id, page, limit, pos_filter, search) → paginated UserCardResponse list
     search: ILIKE on canonical_cards.word_display

3. `backend/app/routers/cards.py`
   Implement all /v1/cards/* endpoints from Section 10.
   PUT /v1/cards/:id accepts partial content_override JSONB — merge, not replace.

4. `backend/app/routers/reviews.py`
   Implement all /v1/reviews/* endpoints from Section 10.
   POST /v1/reviews:
   - Insert immutable review row
   - Call scheduler.process_review()
   - Update card_schedules row
   - Return {next_due, new_state, cards_remaining_today}
   GET /v1/reviews/stats:
   - due_today: count where due_at <= end of today
   - reviewed_today: count reviews in last 24h
   - streak: consecutive days with at least 1 review (compute from review log)
   - retention_7d: % of reviews in last 7 days where rating >= 3

5. `backend/app/services/export.py` + `backend/app/routers/export.py`
   GET /v1/export/csv
   - Fetch all user_cards with canonical content + user_card_examples
   - Build TSV with columns and header from Section 15
   - Return as StreamingResponse with content-type text/tab-separated-values
   - Filename: germanCards-export-YYYY-MM-DD.txt

6. `backend/app/routers/decks.py`
   Implement deck endpoints from Section 10:
   - GET /v1/decks: return public decks + user's subscribed decks
   - GET /v1/decks/:id: deck info + card list (canonical_id, word_display, pos, audit_status)
   - POST /v1/decks/:id/subscribe: for each deck_card where canonical audit_status='approved',
     call collection.add_card(). Skip unapproved cards. Return {added, skipped}.
   - DELETE /v1/decks/:id/subscribe: remove all user_cards for this deck's canonical_ids.

7. `backend/app/routers/admin.py`
   - GET /v1/admin/queue: canonical_cards where audit_status='needs_human_review', paginated
   - PUT /v1/admin/cards/:id: update content + audit_status. If approved, also regenerate aliases.
   - POST /v1/admin/jobs/batch-generate:
     Accepts {words: [{word, pos, gender?}], deck_name?: str}
     Creates AdminJob record, runs processing as BackgroundTask:
       For each word: build NormalizedRequest → canonical.lookup → if MISS → run pipeline
       Track successes/failures in job result_data.
       If deck_name provided: create Deck, add all approved canonical_ids to deck_cards.
     Returns {job_id} immediately.
   - GET /v1/admin/jobs/:id: return job status and result_data.

═══════════════════════════════════════════════════════════════════════════════
SESSION 5 — Frontend Implementation
═══════════════════════════════════════════════════════════════════════════════

ATTACH: MASTER_SPEC.md

---

Read MASTER_SPEC.md fully before writing any code.
Backend is complete. All API endpoints work. Now build the frontend.

The frontend is a React 18 SPA with Vite, Tailwind CSS, React Router v6, and axios.
Design language: clean, minimal, dark-mode friendly. Focus on readability and keyboard use.
No component library — use Tailwind utility classes only.

1. `frontend/src/api/client.js`
   Axios instance:
   - baseURL from env var VITE_API_URL (default http://localhost:8000)
   - Request interceptor: attach Authorization: Bearer <token> from memory
     (store access token in memory — NOT localStorage. Use React context.)
   - Response interceptor: on 401, call /v1/auth/refresh, retry original request once.
     If refresh also fails: redirect to /login.

2. `frontend/src/api/` — one file per resource:
   auth.js, cards.js, reviews.js, decks.js, export.js, input.js
   Each exports async functions wrapping the axios calls.

3. `frontend/src/context/AuthContext.jsx`
   - Provides: user, isAdmin, accessToken (in memory), login(), logout()
   - On app load: attempt /v1/auth/refresh to restore session from cookie.
   - login() calls /v1/auth/login, stores token in memory, sets user.
   - logout() calls /v1/auth/logout, clears memory state.

4. `frontend/src/pages/LoginPage.jsx`
   - Email + password form
   - On submit: call auth.login(), redirect to /
   - Show error on failed login

5. `frontend/src/pages/DashboardPage.jsx`
   - Stats row: cards due today, reviewed today, current streak
   - "Study Now" button → /study (disabled if due_today=0)
   - "Add Card" button → /add
   - "Browse Cards" link → /cards
   - "My Decks" link → /decks

6. `frontend/src/pages/StudyPage.jsx`
   This is the most important page. Requirements:
   - Fetch due cards on mount (GET /v1/reviews/due?limit=20)
   - Show progress: "3 / 20 cards" bar at top
   - FRONT of card: word_display large, POS badge, gender badge if noun
   - Click/tap anywhere on card → flip animation → show BACK
   - BACK shows (in order):
     1. Meanings (numbered list)
     2. If noun: article + plural
     3. If verb: conjugations table (present_3sg | perfekt | praeteritum)
     4. If adj: declension hint
     5. Collocations / compounds (small text)
     6. User context sentence first (if exists), then canonical example sentences
     7. Synonyms/antonyms (only if not empty arrays)
   - After flip: show 4 rating buttons: Again / Hard / Good / Easy
   - Rating buttons show estimated next review time (Again=<1d, Hard=few days, etc.)
     (hardcode estimates for V1 — not computed dynamically)
   - On rating click: POST /v1/reviews, advance to next card
   - Keyboard: Space = flip, 1=Again, 2=Hard, 3=Good, 4=Easy
   - When queue empty: show completion screen with session stats

7. `frontend/src/pages/AddCardPage.jsx`
   - Large text input (autofocused)
   - Below input: optional dropdowns (Type, POS, Gender) — collapsed by default, "Options" toggle
   - Submit button: "Look up"
   - States:
     - loading: spinner
     - HIT (card exists): show card preview card component, "Add to my cards" button
       If already in user collection: "Already in your cards" with link to card
     - NEEDS_USER_CHOICE: show disambiguation buttons (e.g., "essen as verb" / "essen as noun")
       On click: POST /v1/input/confirm-choice
     - MISS (generating): "Generating card..." with progress indication
       Poll result or wait for response. Show generated card with "Saved ✓" message.
     - needs_review=true: show generated card with inline edit form pre-filled
       "Card needs your review" banner. Fields are editable. Save button.
   - Sentence input detection: if type inferred as sentence, show banner:
     "We'll create a card for '[extracted_term]' and save your sentence as an example."

8. `frontend/src/pages/CardListPage.jsx`
   - Search input (debounced 300ms → GET /v1/cards?search=...)
   - Filter tabs: All / Nouns / Verbs / Adjectives / Phrases
   - Card list: word, POS badge, gender, first meaning, due date badge
   - Click → /cards/:id
   - "needs_review" cards have orange warning badge

9. `frontend/src/pages/CardDetailPage.jsx`
   - Full card display (same layout as StudyPage back face)
   - Edit mode: click "Edit" button → all fields become editable inline
   - Save → PUT /v1/cards/:id with {content_override: changedFields}
   - Delete card button (with confirmation dialog)

10. `frontend/src/pages/DecksPage.jsx`
    - List of public decks with name, card count, description
    - Subscribed decks highlighted
    - "Subscribe" / "Unsubscribe" button per deck
    - On subscribe: show toast "Added X cards to your collection"

11. `frontend/src/pages/AdminQueuePage.jsx` (only shown if isAdmin)
    - List of needs_human_review cards
    - Each row: word, POS, audit_score, audit_notes, "Edit & Approve" button
    - Edit modal: edit all content fields inline, Approve / Reject buttons

12. `frontend/src/components/`
    - CardDisplay.jsx: reusable card face renderer (used in Study + Detail)
    - POSBadge.jsx: colored badge for noun/verb/adj/phrase
    - Toast.jsx: simple notification component
    - ProtectedRoute.jsx: redirects to /login if not authenticated

Design notes:
- Background: gray-950 or gray-900 for dark feel
- Card faces: white bg with strong shadow
- Flip animation: CSS transform rotateY (use Tailwind + inline style)
- Rating buttons: color-coded (red=Again, orange=Hard, green=Good, blue=Easy)
- Font: system font stack, large for card words (text-4xl or text-5xl)

═══════════════════════════════════════════════════════════════════════════════
SESSION 6 — Hardening, Tests, Deployment Config
═══════════════════════════════════════════════════════════════════════════════

ATTACH: MASTER_SPEC.md

---

Read MASTER_SPEC.md fully before writing any code.
Sessions 1-5 complete. Full app is built. Now harden it for real use.

1. Security headers middleware (`backend/app/middleware/security.py`):
   Add these headers to all responses:
   - X-Content-Type-Options: nosniff
   - X-Frame-Options: DENY
   - X-XSS-Protection: 1; mode=block
   - Referrer-Policy: strict-origin-when-cross-origin
   - Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'

2. Input sanitization audit:
   Review every router that accepts user text input.
   Ensure `sanitize_input()` from generation.py is called on all text before it touches
   the DB or any LLM. For card content edits (PUT /v1/cards/:id), sanitize each string
   field of content_override.

3. Rate limiting review:
   - POST /v1/input/resolve → 20/minute per user
   - POST /v1/auth/login → 10/minute per IP (apply to router)
   - GET /v1/reviews/due → 60/minute per user
   - POST /v1/reviews → 120/minute per user
   Confirm all limits are applied via @limiter.limit decorators.

4. Add missing indexes (if any added during implementation) via new Alembic migration.

5. Write tests:
   - test_normalize.py: all cases listed in Session 2 task 6
   - test_input_parser.py: all cases listed in Session 2 task 7
   - test_generation_pipeline.py:
     - Mock LLM adapter. Test that ValidationError on generator output triggers retry.
     - Test that audit score < 0.85 sets audit_status='needs_human_review'.
     - Test that injection attempt in input raises ValueError.
   - test_auth.py: login success, login fail, refresh, logout

6. `backend/Dockerfile`:
   ```
   FROM python:3.12-slim
   WORKDIR /app
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt
   COPY . .
   CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
   ```

7. `frontend/Dockerfile`:
   ```
   FROM node:20-alpine AS builder
   WORKDIR /app
   COPY package*.json .
   RUN npm ci
   COPY . .
   RUN npm run build

   FROM nginx:alpine
   COPY --from=builder /app/dist /usr/share/nginx/html
   COPY nginx.conf /etc/nginx/nginx.conf
   ```
   Create a minimal `nginx.conf` that serves the SPA (all routes → index.html).

8. Update `docker-compose.yml` to include the app containers for production-like local testing.
   Add `db`, `redis`, `backend`, `frontend` services.

9. `README.md` with:
   - Prerequisites (Docker, Python 3.12, Node 20)
   - Local dev setup (step by step)
   - How to create admin user
   - How to run tests
   - How to seed words via batch-generate API
   - Environment variable reference

10. Final checklist — verify each item:
    [ ] `alembic upgrade head` creates all tables with correct constraints
    [ ] `python scripts/create_admin.py` creates admin user
    [ ] POST /v1/auth/login returns JWT and sets cookie
    [ ] POST /v1/input/resolve with "laufen" (cache miss) calls generation pipeline
    [ ] POST /v1/input/resolve with same word twice hits cache, no LLM call
    [ ] POST /v1/reviews with rating=3 updates card_schedule and inserts review
    [ ] GET /v1/export/csv returns valid TSV with all user cards
    [ ] Accessing /v1/admin/queue with non-admin JWT returns 403
    [ ] Frontend login → dashboard → add card → study flow works end to end
```

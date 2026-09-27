# PharmQMS — Architecture Notes

## High-Level Flow

### 1. Public complaint (customer)
```
Customer opens /submit → fills form → POST /api/complaints/public
                                       ↓
                             MongoDB.complaints.insertOne
                                       ↓
                     is_public=true, seen=false, status="Pending Triage"
                                       ↓
                       (later) admin sees red bell → mark-read
```

### 2. Staff complaint (QA analyst)
```
Analyst opens /complaints/new
   → Paste text OR upload PDF/DOCX/TXT/EML
   → POST /api/ai/extract | /api/ai/extract-file
       ↓
   LangGraph pipeline in ai_agent.py:
     preprocess → extract (Claude Sonnet 4.5) → assess → finalize
       ↓
   Form auto-populates (13 fields)
   → Analyst reviews / edits
   → POST /api/complaints → MongoDB → Redirect to /dashboard
```

### 3. Dashboard
```
GET /api/complaints/stats  →  5 metric cards
GET /api/complaints        →  Sortable table
GET /api/notifications     →  Bell badge (poll 20s)
```

---

## Backend Choices

- **FastAPI** — async-first, Pydantic validation matches our strict JSON contract with the LLM.
- **motor (async MongoDB)** — non-blocking DB I/O; avoids thread-pool contention when the LLM call is slow.
- **bcrypt** — password hashing (cost 12 by default). Passwords never logged or echoed.
- **PyJWT** — HS256 tokens, 24h expiry, `sub=email` + custom `role`.
- **emergentintegrations** — Emergent Universal LLM Key wrapper. Selected model: `claude-sonnet-4-5-20250929`.
- **LangGraph** — deterministic 4-node graph gives us clear failure points and future-proofs the pipeline (easy to add "duplicate detection" or "risk classification" nodes).
- **Document parsing** — `pypdf` (text), `python-docx` (paragraphs), stdlib `email` (EML). No OCR (per assignment; "production-grade OCR not required").

### Load order matters
`server.py` **loads `.env` before importing** `ai_agent.py`. The LLM key inside `ai_agent.py` is read lazily via `_get_key()` so environment mutation from `load_dotenv` is guaranteed to be visible.

### Startup seed
On boot, `seed_admin()` upserts the admin user and **re-syncs the password hash** from `.env` so the demo credentials always work after a redeploy.

---

## Frontend Choices

- **React 19 + React Router 7** — modern routing with `<Navigate>` for auth gating.
- **Redux Toolkit** — user requirement. Two slices: `auth` (token, email) and `complaints` (list, stats).
- **Axios interceptors** — inject Bearer token; on 401 clear token + redirect to `/login`.
- **Shadcn UI** — all form primitives (Input, Select, Textarea, Table, Badge, Card, Button). Custom design tokens in `index.css`.
- **Framer Motion** — chat message entrance stagger only (not everywhere) — keeps interactions crisp.
- **Sonner** — top-right toasts.
- **Inter font** — loaded from Google Fonts with strict typesetting rules (uppercase 0.15em tracking for QA labels, tight tracking on headers).

### Polling notifications
`NotificationBell` polls `/api/notifications` every 20 s using `setInterval`. When
`unread_count` **increases** (and it's not the first poll), a Sonner toast fires so
even distracted analysts get an in-app nudge. Panel opens on click and closes on
outside-click.

---

## Security

- Password never leaves the backend in plaintext.
- JWT stored in `localStorage` — acceptable for internal QMS tools; upgrade to httpOnly cookies if the app leaves the intranet.
- Public endpoint (`/api/complaints/public`) has **no auth**; validation is done via Pydantic with strict length and email checks. Rate-limit + CAPTCHA are on the roadmap.
- CORS is `*` in dev — tighten to your production origins by setting `CORS_ORIGINS` before shipping externally.

---

## AI Prompt Design

The system prompt in `ai_agent.py` is intentionally strict:

- Explicitly enumerates the 13 keys.
- Forces ISO `YYYY-MM-DD` dates.
- Enumerates valid `complaint_type`, `initial_severity`, `priority` values.
- Bans markdown fences and prose (`"Return ONLY the JSON object"`).
- Encourages `null` over hallucination.

The response is then defensively parsed via `_parse_json()` which:
1. Strips ` ```json ` fences.
2. Falls back to regex `{...}` extraction if the LLM adds preamble.
3. Returns `{}` on unrecoverable output (never raises).

---

## MongoDB — no ObjectId leak

All models use a plain `id: str` field seeded with `uuid.uuid4()`. Every `find` uses
projection `{"_id": 0}`. This means no BSON `ObjectId` ever crosses the API boundary
and Pydantic serialisation is guaranteed to succeed.

---

## Adding a new AI capability

Say you want **duplicate detection**. Steps:

1. Add a MongoDB text index on `description` (one-time).
2. Add a new LangGraph node `duplicate_check` after `extract`:
   ```python
   async def _duplicate_check(state):
       matches = await db.complaints.find(
           {"$text": {"$search": state["extracted"]["description"]}}
       ).limit(3).to_list(3)
       state["duplicates"] = matches
       return state
   ```
3. Return `duplicates` from `run_intake_agent`.
4. In `AIIntakeAssistant.jsx`, render a "Possible duplicates" card above the chat when the array is non-empty.

Because the graph is a state machine, adding a node doesn't touch any other node —
this is the payoff of using LangGraph over a linear function chain.

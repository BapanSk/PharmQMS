# PharmQMS — AI-Powered Customer Complaint Management System

**Domain:** Pharmaceutical manufacturing (API & FDF) Quality Assurance
**Stack:** React 19 + Redux Toolkit · FastAPI · MongoDB · LangGraph · Claude Sonnet 4.5

An enterprise-grade Quality Management System (QMS) complaint intake module. QA analysts
log complaints from any source — public web form, email, PDF, DOCX — and an AI agent
automatically extracts the structured pharma fields (product, batch, severity, priority,
etc.), saving the team hours of manual data entry.

---

## Live Environments

| Environment | URL |
|-------------|-----|
| **Production** | `https://website-builder-2480.emergent.host` |
| **Preview (dev)** | `https://website-builder-2480.preview.emergentagent.com` |

**Public complaint URL (share this with customers):** `<base>/submit`
**Staff console:** `<base>/login`

## Demo Credentials

```
Email:    admin@pharmqms.com
Password: Admin@123
```
(auto-seeded on backend startup from `ADMIN_EMAIL` / `ADMIN_PASSWORD` in `.env`)

---

## Features

### For customers (no login)
- Public `/submit` page: name, email, phone, company, product, batch/lot, type, description
- Instant reference ID on success (copyable)

### For QA staff (JWT login)
- **Dashboard** — 5 metric cards (Total, Pending Triage, Under Review, Closed, Critical) + sortable complaint table
- **Log Complaint** — split-panel:
  - **Left:** 4-section form (Origin & Customer · Product & Batch · Complaint Details · Assessment & Priority) — 13 fields
  - **Right:** AI Complaint Intake Assistant
    - Drag & drop or click-to-browse: PDF · DOCX · TXT · EML (max 10MB)
    - OR paste complaint email/text
    - LangGraph pipeline auto-populates all form fields
    - Chat with the AI about the current complaint
- **Notification bell** — red badge for unread public submissions, polling every 20s, mark-all-read
- **Sonner toasts** for every action
- **data-testid** attribute on every interactive element (for automated QA)

### AI Agent (LangGraph)
Deterministic 4-node state machine in `backend/ai_agent.py`:

```
preprocess → extract → assess → finalize
```

- `preprocess` — normalizes and truncates the input to keep token cost bounded
- `extract` — Claude Sonnet 4.5 returns strict JSON with 13 pharma fields
- `assess` — heuristic completeness scoring; produces the assistant message
- `finalize` — hands the result back to the API

---

## Architecture

```
┌────────────────────────┐          ┌────────────────────────┐
│  React (Redux + RRD)   │  HTTPS   │  FastAPI (uvicorn)     │
│  /submit  /login       │◀────────▶│  /api/*                │
│  /dashboard /new       │  (JWT)   │  (all routes prefixed) │
└────────────────────────┘          └──────────┬─────────────┘
                                               │
                              ┌────────────────┼────────────────────┐
                              ▼                ▼                    ▼
                       ┌─────────────┐  ┌─────────────┐    ┌───────────────────┐
                       │   MongoDB   │  │  LangGraph  │    │ Emergent LLM Key  │
                       │  motor-async│  │  agent      │───▶│ (Claude Sonnet 4.5)│
                       └─────────────┘  └─────────────┘    └───────────────────┘
```

**Why MongoDB (vs Postgres/MySQL in the original spec):** the managed platform ships
with MongoDB. Schema is document-based and mirrors the pharma complaint fields exactly.

**Why Claude Sonnet 4.5 (vs Groq gemma2-9b-it in the original spec):** Groq keys
weren't provided; the Emergent Universal LLM Key covers OpenAI/Anthropic/Gemini and
Claude Sonnet 4.5 delivers superior structured-JSON extraction for pharma domain.
Swap is one-line if a Groq key becomes available.

---

## Project Layout

```
/app
├── backend/
│   ├── server.py            # FastAPI app, routes, MongoDB, startup seed
│   ├── auth.py              # JWT (PyJWT) + bcrypt password hashing
│   ├── ai_agent.py          # LangGraph pipeline + emergentintegrations LLM
│   ├── document_parser.py   # pypdf / python-docx / stdlib email parsing
│   ├── requirements.txt
│   └── .env                 # MONGO_URL, DB_NAME, EMERGENT_LLM_KEY, JWT_SECRET
├── frontend/
│   ├── package.json         # yarn, React 19, RRD 7, Redux Toolkit, framer-motion
│   ├── .env                 # REACT_APP_BACKEND_URL
│   └── src/
│       ├── App.js
│       ├── index.js
│       ├── App.css
│       ├── index.css        # Inter font, HSL tokens, ai-pulse animation
│       ├── lib/api.js       # axios client with Bearer interceptor
│       ├── store/index.js   # Redux slices: auth + complaints
│       ├── components/
│       │   ├── Navbar.jsx
│       │   ├── NotificationBell.jsx
│       │   ├── AIIntakeAssistant.jsx
│       │   ├── Badges.jsx
│       │   └── ui/          # Shadcn primitives
│       └── pages/
│           ├── Login.jsx
│           ├── Dashboard.jsx
│           ├── LogComplaint.jsx
│           └── PublicSubmit.jsx
└── memory/
    ├── PRD.md
    ├── test_credentials.md
    └── ARCHITECTURE.md      (this file lives at /app/README.md — see also docs/)
```

---

## Environment Variables

### `backend/.env`
| Variable | Purpose |
|----------|---------|
| `MONGO_URL` | MongoDB connection string (default local `mongodb://localhost:27017`) |
| `DB_NAME` | Mongo database name |
| `CORS_ORIGINS` | Comma-separated allow list (`*` in dev) |
| `EMERGENT_LLM_KEY` | Emergent Universal LLM Key (Claude / OpenAI / Gemini) |
| `JWT_SECRET` | HS256 signing secret for JWT |
| `ADMIN_EMAIL` | Seeded admin login |
| `ADMIN_PASSWORD` | Seeded admin password (rehashed on every startup) |
| `NOTIFICATIONS_EMAIL` | Recipient for future email alerts (email currently NOT wired) |

### `frontend/.env`
| Variable | Purpose |
|----------|---------|
| `REACT_APP_BACKEND_URL` | Public base URL of the FastAPI backend |

> Never hardcode any of these values — they're read from `process.env` / `os.environ`
> everywhere in the codebase.

---

## Local Development

The container in this platform already has both services running under
supervisor. If you're running elsewhere:

### Backend
```bash
cd backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
pip install emergentintegrations --extra-index-url https://d33sy5i8bnduwe.cloudfront.net/simple/
cp .env.example .env   # fill in the values above
uvicorn server:app --host 0.0.0.0 --port 8001 --reload
```

### Frontend
```bash
cd frontend
yarn install
# frontend/.env should point REACT_APP_BACKEND_URL at http://localhost:8001
yarn start
```

Open http://localhost:3000/login and sign in with the demo credentials.

---

## API Reference

Full API spec is in [`docs/API.md`](docs/API.md).

Quick summary:

| Method | Route | Auth | Purpose |
|--------|-------|------|---------|
| POST | `/api/auth/login` | – | Get JWT |
| GET | `/api/auth/me` | JWT | Current user |
| POST | `/api/complaints/public` | – | **Public complaint intake** |
| GET | `/api/complaints` | JWT | List complaints |
| GET | `/api/complaints/stats` | JWT | Dashboard counters |
| GET | `/api/complaints/{id}` | JWT | Single record |
| POST | `/api/complaints` | JWT | Create (from Log Complaint form) |
| PATCH | `/api/complaints/{id}` | JWT | Update status / any field |
| POST | `/api/ai/extract` | JWT | Extract fields from pasted text |
| POST | `/api/ai/extract-file` | JWT | Extract fields from an uploaded document |
| POST | `/api/ai/chat` | JWT | Chat about the current complaint |
| GET | `/api/notifications` | JWT | Unread public complaints |
| POST | `/api/notifications/mark-read` | JWT | Mark all as seen |

---

## Data Model (MongoDB)

### `users`
```json
{
  "id": "uuid",
  "email": "admin@pharmqms.com",
  "password_hash": "$2b$…",
  "role": "admin",
  "created_at": "ISO-8601"
}
```

### `complaints`
```json
{
  "id": "uuid",
  "complaint_source": "Email | Phone | Portal | Regulatory | Public Form | …",
  "customer_name": "string | null",
  "product_name": "string | null",
  "product_strength": "string | null",
  "batch_lot_number": "string | null",
  "manufacturing_date": "YYYY-MM-DD | null",
  "expiry_date": "YYYY-MM-DD | null",
  "quantity_affected": "string | null",
  "complaint_type": "Impurity | Packaging Defect | Efficacy | Contamination | Labeling | Adverse Event | Other",
  "complaint_date": "YYYY-MM-DD | null",
  "description": "string | null",
  "initial_severity": "Minor | Major | Critical | null",
  "priority": "Low | Medium | High | Urgent | null",
  "status": "Pending Triage | Under Review | Closed",
  "created_by": "email | null",
  "is_public": "bool",
  "seen": "bool",
  "reporter_email": "string | null",
  "reporter_phone": "string | null",
  "reporter_company": "string | null",
  "created_at": "ISO-8601",
  "updated_at": "ISO-8601"
}
```

---

## AI Extraction — Example

**Input** (pasted text):
> Dear QA Team, we received a complaint from Apollo Hospitals about Paracetamol 500 mg
> tablets, Batch B2405-A72, manufactured 2024-05-10 and expiring 2026-05-09. Two blisters
> showed white powder discoloration. Approximately 240 tablets affected. Reported on
> 2026-01-14 via email. Patient safety not impacted.

**Output** (`POST /api/ai/extract`):
```json
{
  "extracted": {
    "complaint_source": "Email",
    "customer_name": "Apollo Hospitals",
    "product_name": "Paracetamol",
    "product_strength": "500 mg",
    "batch_lot_number": "B2405-A72",
    "manufacturing_date": "2024-05-10",
    "expiry_date": "2026-05-09",
    "quantity_affected": "240 tablets",
    "complaint_type": "Packaging Defect",
    "complaint_date": "2026-01-14",
    "description": "Two blisters …",
    "initial_severity": "Minor",
    "priority": "Medium"
  },
  "missing_fields": [],
  "assistant_message": "Extraction complete. All 13 fields captured. Please review and save."
}
```

---

## Roadmap (deferred features)

- **Bonus AI Suite** — Duplicate Detection, CAPA Recommendation, Root-Cause Analysis, AI Risk Classification, Auto-Summary
- **Complaint Detail View** with status workflow (Pending Triage → Under Review → Closed)
- **CSV Export** with filters (product / severity / priority / date)
- **Real Email Alerts** via Resend or SendGrid (recipient already in `.env`)
- **Attachment Uploads** on the public form (photo/PDF of defective product)
- **Rate limiting + CAPTCHA** on the public endpoint
- **Groq gemma2-9b-it** swap (one-line change once a Groq key is provided)

---

## License

Internal / proprietary. Adapt as needed for your organisation.

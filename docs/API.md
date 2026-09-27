# PharmQMS — API Reference

Base URL: `${REACT_APP_BACKEND_URL}` (e.g. `https://website-builder-2480.emergent.host`)
All routes are prefixed with `/api`.
Auth: JWT Bearer token (from `POST /api/auth/login`) in the `Authorization` header.

---

## Auth

### `POST /api/auth/login`
Request:
```json
{ "email": "admin@pharmqms.com", "password": "Admin@123" }
```
Response 200:
```json
{
  "token": "eyJhbGciOi…",
  "email": "admin@pharmqms.com",
  "role": "admin"
}
```
Errors: `401` invalid credentials.

### `GET /api/auth/me`  *(JWT)*
```json
{ "email": "admin@pharmqms.com", "role": "admin" }
```

---

## Public Complaint Intake

### `POST /api/complaints/public`  *(no auth)*
Public URL-friendly endpoint. Anyone can hit this from the `/submit` page.

Request:
```json
{
  "reporter_name": "Jane Doe",
  "reporter_email": "jane@apollo.com",
  "reporter_phone": "+91 98765 43210",
  "reporter_company": "Apollo Hospitals",
  "product_name": "Paracetamol 500 mg",
  "batch_lot_number": "B2405-A72",
  "complaint_type": "Packaging Defect",
  "description": "Blister pack torn on delivery; 3 tablets missing."
}
```
Required: `reporter_name`, `reporter_email`, `product_name`, `description` (min 10 chars).

Response 200:
```json
{
  "success": true,
  "reference_id": "32ef17e7-05ac-419c-9849-838e91cf240f",
  "message": "Your complaint has been received. Our QA team will review it and reach out to you."
}
```

---

## Complaints

### `GET /api/complaints`  *(JWT)*
Returns all complaints sorted by newest first. Response is an array of `Complaint` objects.

### `GET /api/complaints/stats`  *(JWT)*
```json
{ "total": 5, "pending": 3, "under_review": 1, "closed": 1, "critical": 2 }
```

### `GET /api/complaints/{id}`  *(JWT)*
Single record. `404` if not found.

### `POST /api/complaints`  *(JWT)*
Body: any subset of the 13 pharma fields (see [Complaint schema](../README.md#data-model-mongodb)).
Auto-assigns `id`, `status="Pending Triage"`, `created_by=<jwt email>`.

### `PATCH /api/complaints/{id}`  *(JWT)*
Body: any subset of `Complaint` fields (except `id`). Updates `updated_at` automatically.
Common use — status transition:
```json
{ "status": "Under Review" }
```

---

## AI

### `POST /api/ai/extract`  *(JWT)*
Extract pharma fields from pasted text via the LangGraph pipeline.
```json
{ "text": "Dear QA Team, we received a complaint from Apollo Hospitals about …" }
```
Response — see [example](../README.md#ai-extraction--example).

### `POST /api/ai/extract-file`  *(JWT — multipart/form-data)*
Upload a `.pdf`, `.docx`, `.txt`, or `.eml` file (max 10 MB) under form field `file`.
Same response as `/ai/extract` plus a `raw_text` preview (first 4000 chars).

```bash
curl -X POST "$BASE/api/ai/extract-file" \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@complaint.pdf"
```

### `POST /api/ai/chat`  *(JWT)*
Free-form Q&A about a complaint. `context` is optional (usually the last extracted `raw_text`).
```json
{ "question": "What batch was reported?", "context": "…full complaint text…" }
```
Response:
```json
{ "answer": "Batch B2405-A72 of Paracetamol 500 mg tablets." }
```

---

## Notifications

### `GET /api/notifications`  *(JWT)*
```json
{
  "unread_count": 3,
  "items": [
    { "id": "…", "customer_name": "Jane Doe", "product_name": "Paracetamol 500 mg", "…": "…" }
  ]
}
```
Items are the 20 most recent unread public complaints (`is_public=true` & `seen != true`).

### `POST /api/notifications/mark-read`  *(JWT)*
Marks all unread public complaints as seen.
```json
{ "marked": 3 }
```

---

## Error Contract

FastAPI returns `{ "detail": "<message>" }` with the appropriate HTTP code:

| Code | Meaning |
|------|---------|
| 400 | Bad request / validation failure (e.g. empty text) |
| 401 | Missing or invalid token |
| 404 | Resource not found |
| 413 | File too large (>10 MB) |
| 500 | LLM or server error (see backend logs) |

---

## cURL Recipes

```bash
BASE="https://website-builder-2480.emergent.host"

# 1. Login
TOKEN=$(curl -s -X POST "$BASE/api/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@pharmqms.com","password":"Admin@123"}' \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['token'])")

# 2. List stats
curl -s "$BASE/api/complaints/stats" -H "Authorization: Bearer $TOKEN"

# 3. AI extract
curl -s -X POST "$BASE/api/ai/extract" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"text":"Apollo Hospitals report Paracetamol 500 mg B2405-A72 discoloration"}'

# 4. Public complaint (no token)
curl -s -X POST "$BASE/api/complaints/public" \
  -H "Content-Type: application/json" \
  -d '{"reporter_name":"Jane","reporter_email":"j@x.com","product_name":"Ibuprofen","description":"issue observed today at hospital"}'

# 5. Notifications
curl -s "$BASE/api/notifications" -H "Authorization: Bearer $TOKEN"
```

# PharmQMS — User Guide

## 1. As a customer (public complaint)

1. Open the shared URL: **`<base>/submit`**
2. Fill in the form:
   - **Reporter Details** — your name and email are required; phone and company help us follow up.
   - **Product Information** — product name is required; batch/lot and type help triage faster.
   - **Complaint Details** — describe what happened in your own words (min 10 characters).
3. Click **Submit Complaint**.
4. You'll see a green confirmation with a **Reference ID** (e.g. `32ef17e7-…`). Copy it — quote it in any follow-up email.

That's it. Our QA team is notified inside the console the moment you submit.

---

## 2. As a QA analyst / admin

### 2.1 Sign in
- Visit **`<base>/login`**
- Demo credentials are pre-filled: `admin@pharmqms.com` / `Admin@123`
- Click **Sign in** → land on the Dashboard.

### 2.2 Read the Dashboard
- Five metric cards at the top summarise the queue:
  - **Total Complaints** — everything ever logged
  - **Pending Triage** — the "todo" pile
  - **Under Review** — actively being investigated
  - **Closed** — resolved
  - **Critical** — severity=Critical (regardless of status)
- The table below lists every complaint newest-first. `data-testid` on each row makes it QA-automation-ready.

### 2.3 See a new public complaint arrive
- A red badge appears on the **bell icon** in the top-right of the navbar with the unread count.
- If you're on the page when it arrives, a **toast** pops "New public complaint received".
- Click the bell → panel shows the last 20 unread submissions with reporter, product, batch and a snippet.
- Click **Mark all read** to clear the badge, or click any item to jump to the Dashboard.

### 2.4 Log a complaint from an email / PDF
- Click **Log Complaint** (top-right of Dashboard or in the navbar).
- Two panels appear side by side.

**Right — AI Complaint Intake Assistant:**
- Drag a `.pdf`, `.docx`, `.txt` or `.eml` file onto the dropzone (or click to browse).
- **OR** paste the complaint email/text into the textarea and click **Analyze Text**.
- The progress bar fills. When it's done:
  - The form on the left auto-populates.
  - The Assistant chat area reports how many of the 13 fields were captured.
- You can then **ask the AI questions** in the chat box, e.g.
  - "What batch is this?"
  - "Draft a professional acknowledgement email to the customer."
  - "What's the risk if we don't recall this batch?"

**Left — Log Customer Complaint form:**
- Four numbered sections; every field is editable — the AI is a suggestion, you have the final say.
- **Reset Form** clears everything back to `"Awaiting AI extraction…"` placeholders.
- **Save Complaint** persists the record and drops you back on the Dashboard where it appears with a `Pending Triage` badge.

### 2.5 Sign out
- Click **Logout** in the navbar. You'll be redirected to `/login` and the JWT is cleared from browser storage.

---

## 3. Distributing the public URL

Ways to get your `<base>/submit` link in front of customers:

- Add a "**Report a Complaint**" button on your corporate website.
- Include the URL (or a QR code of it) on product packaging, invoices, and delivery notes.
- Add it to auto-reply signatures on your customer-service inbox.
- Share it with distributors, hospitals, regulators and clinical trial partners.

The public form has **no login and no company branding requirement** — you can drop the URL anywhere.

---

## 4. Troubleshooting

| Symptom | Fix |
|---------|-----|
| "Invalid credentials" on login | Re-check the seeded creds in `backend/.env` (`ADMIN_EMAIL` / `ADMIN_PASSWORD`) — they re-sync on every backend restart. |
| AI extract returns an error | Check `EMERGENT_LLM_KEY` in `backend/.env` and your Emergent balance. Backend logs: `tail -f /var/log/supervisor/backend.err.log`. |
| "File too large" on upload | Max 10 MB. Compress the PDF or paste the text into the textarea instead. |
| Bell never lights up | The frontend polls every 20 s — hard-refresh or wait a moment. Verify the public POST returned a 200. |
| Dashboard table empty after Save | Check the Network tab: `POST /api/complaints` should be 200 and the response added to the top of the list. |

# PharmQMS – AI-powered Customer Complaint Management System

## Original Problem Statement
Build an AI-powered Customer Complaint Management System for the pharmaceutical
manufacturing industry (API & FDF QA module). Two-panel reference UI: LEFT — "Log
Customer Complaint" form (Origin & Customer, Product & Batch, Complaint Details,
Assessment & Priority) with Reset/Save; RIGHT — "AI Complaint Intake Assistant"
that ingests PDF/DOCX/TXT/EML or pasted text, extracts structured fields via an
LLM, and provides a chat interface.

## Tech Stack (as delivered v1)
- **Frontend:** React 19 + Redux Toolkit + React Router 7 + Tailwind + Shadcn UI + Framer Motion + Sonner
- **Backend:** FastAPI + Motor (MongoDB async)
- **Database:** MongoDB (swap from Postgres approved by user)
- **AI Orchestration:** LangGraph state machine (preprocess → extract → assess → finalize)
- **LLM:** Claude Sonnet 4.5 via Emergent Universal LLM Key (Groq fallback per user choice)
- **Doc parsing:** pypdf, python-docx, stdlib email
- **Auth:** JWT (bcrypt) with single seeded admin

## Users / Personas
- **QA Analyst / Complaint Handler** (single-role v1) — logs, reviews, triages complaints

## Core Requirements
1. Auth (admin login, JWT-guarded API)
2. Log Complaint form (13 fields, 4 sections) with Reset / Save
3. AI Intake Assistant — file upload OR paste text → auto-populate form + chat
4. Complaints dashboard with metrics + table

## Implemented (2026-02)
- JWT login, admin seed on startup
- Log Complaint form with all 4 sections and 13 fields (shadcn Input/Select/Textarea)
- AI Intake Assistant: PDF/DOCX/TXT/EML upload, paste text, progress bar, chat
- LangGraph pipeline (`ai_agent.py`) using Claude Sonnet 4.5
- Dashboard with 5 metric cards + complaints table
- Sonner toasts, Framer Motion chat animations
- data-testid coverage on every interactive element

## APIs (all under /api)
- POST /auth/login, GET /auth/me
- GET /complaints, GET /complaints/stats, GET /complaints/{id}
- POST /complaints, PATCH /complaints/{id}
- POST /ai/extract (text), POST /ai/extract-file (upload), POST /ai/chat

## Backlog (P0/P1/P2)
- **P1 – Bonus AI features** (per assignment): Complaint Completeness Checker, Duplicate Detection, Root Cause Recommendation, CAPA Recommendation, Complaint Summary, AI Risk Classification
- **P1 – Complaint detail view** with status transitions (Pending Triage → Under Review → Closed)
- **P2 – Advanced dashboard filters** (severity, priority, product, date range) + CSV export
- **P2 – User management** (multiple QA analysts, roles)
- **P2 – Audit trail** (21 CFR Part 11 style change log)
- **P2 – Groq gemma2-9b-it swap** once a Groq API key is provided

## Test Credentials
- admin@pharmqms.com / Admin@123 (seeded on startup)

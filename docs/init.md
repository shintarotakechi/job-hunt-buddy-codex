# Requirements Definition — Personal Job Search Automation (Single-User, No Login, Google Integration Last)

Stack: Next.js 15 (frontend) / ASP.NET Web API (backend) / Hangfire (jobs) / Playwright.NET (crawling) / Semantic Kernel (generation) / Cloudflare Workers + Wrangler (edge/proxy) / PostgreSQL (storage)  
Monorepo: single GitHub repo hosting frontend + backend + shared packages

This document is optimized for implementation by an LLM (small cards with clear scope). It focuses on **what to build**, **why**, **in what order**, and **how to validate**, without including code.

---

## 0. Purpose & Scope (Personal Tool)
- Goal: Continuously find fitting U.S. jobs, auto-generate tailored cover letters, send applications, track status, and schedule interviews—**for a single owner** (you), without any login or multi-tenant SaaS features.
- Out of scope (initial): Public sign-up, billing, multi-user tenancy, admin roles, third-party data providers, mobile apps.
- Privacy: Local-first mindset. Secrets are env-based; repo remains public but without secrets committed.

---

## 1. Architecture (work without Google first; add Google last)
- Edge: Cloudflare Workers
  - Reverse proxy `/api/*` to ASP.NET origin; add rate-limits and request correlation IDs; serve SSR/static.
- Frontend: Next.js 15 (App Router, RSC)
  - Local “owner mode”: no login. A small “Config” panel writes preferences to backend (DB). Dashboard shows applications and statuses; buttons to retry/regenerate.
- Backend: ASP.NET Web API (+ OpenAPI)
  - Ports & Adapters for mail/sheets/calendar/auth so that we can start with **Local/Mock providers** and later swap in Google providers.
- Jobs: Hangfire (PostgreSQL storage)
  - Recurring and delayed jobs for crawl → score → generate → send → sync replies → update sheets/calendar.
- Crawler: Playwright.NET
  - Structured adapters per source (search URL builder, pagination, selectors, normalization, dedupe).
- Storage: PostgreSQL
  - Single-owner schema (no user table). Entities: Preferences, ResumeProfile, JobPosting, Application, EmailThread, SheetRow, CalendarEvent, CrawlLog, JobLog, FeatureFlags, SecretsMeta.
- Providers (first = Local/Mock; last = Google):
  - MailPort: LocalSMTP (send), IMAP (receive) → later Gmail.
  - SheetPort: Local DB rows + CSV exporter → later Google Sheets.
  - CalendarPort: ICS file generator → later Google Calendar.
  - AuthPort: ‘none’ (no login) → later Google Sign-In if ever needed.

---

## 2. Monorepo Layout (logical only)
- `/apps/frontend` — Next.js 15
- `/apps/api` — ASP.NET Web API + Hangfire server
- `/packages/domain` — Domain models, validation, state machine
- `/packages/ports` — Interfaces (MailPort, SheetPort, CalendarPort, AuthPort)
- `/packages/adapters-local` — LocalSMTP, IMAP, CSVSheet, ICSCalendar
- `/packages/adapters-google` — Gmail, Sheets, Calendar (added last)
- `/packages/jobs` — Hangfire jobs (crawl/score/generate/send/sync)
- `/packages/edge` — Workers handlers, Wrangler config
- `/infra` — Runbooks, env and secrets naming, deploy notes
- `/docs` — This requirements, ERD text, data contracts

---

## 3. Domain Model (single owner)
- Preference: job_categories[], location_radius, remote_allowed, keywords+, blacklist_keywords[], salary_floor, seniority_range.
- ResumeProfile: extracted keywords/summary from your resume; multiple versions supported.
- JobPosting: source, external_id, title, company, location, description_digest, url, fetched_at, dedupe_key.
- Application: job_posting_id, fit_score, cover_letter_version, status (collected | generated | emailed | awaiting | interview | offer | rejected | closed), last_transition_at.
- EmailThread: application_id, provider_thread_id (or msg IDs), labels[], last_snippet, last_synced_at.
- SheetRow: application_id, row_key, fields(jsonb); CSV export record.
- CalendarEvent: application_id, provider_event_id (or ics file path), when, status.
- State machine: collected → generated → emailed → awaiting → interview/offer/rejected → closed. Events: generate_cover, send_email, reply_received, schedule_interview, outcome_offer, outcome_reject, timeout_followup.

---

## 4. Phasing (Google last)
- Phase 0: Infra and Ports (Local/Mock only).
- Phase 1: Crawl → Score → Generate → Send (LocalSMTP) E2E.
- Phase 2: Receive sync (IMAP), state updates, CSV export (SheetRow), ICS generation.
- Phase 3: Observability, idempotency, error handling, rate control.
- Phase 4: Optional Google adapters (Gmail/Sheets/Calendar) and feature flag switch.

---

## 5. Global Config Flags (env; no secrets in repo)
- FEATURE_GOOGLE: false → true (Phase 4).
- MAIL_TX_PROVIDER: 'local_smtp' | 'gmail'.
- MAIL_RX_PROVIDER: 'imap' | 'gmail'.
- SHEET_PROVIDER: 'local_csv' | 'google'.
- CAL_PROVIDER: 'ics' | 'google'.
- AUTH_METHOD: 'none' (always none for personal mode).
- SAFETY: rate limits, polite crawling delay, user-agent string, robots compliance policy toggle.

---

## 6. Definition of Done (overall)
- From clean deploy, with no login:
  - Preferences set via UI → crawler fetches ≥3 jobs in 24h.
  - Top matches: cover letters generated; at least one application sent via LocalSMTP.
  - Dashboard reflects statuses; CSV file shows rows; ICS files open in a calendar client.
  - On reply (IMAP), state transitions update; interview generates ICS.
  - Failures retried; duplicates not sent; actions fully logged.

---

## 7. Implementation Cards (tiny, ordered, each with validation and dependencies)

[INF-001] Secrets & naming
- Build: Env keys list (DB_CONN, API_ORIGIN, SMTP_HOST/PORT/USER/PASS, IMAP_HOST/USER/PASS, CSV_OUT_DIR, ICS_OUT_DIR, FEATURE_GOOGLE=false).
- Test: Start API with dummy env; missing key yields clear error.
- Depends: none.

[INF-002] Environments & Wrangler
- Build: `wrangler.toml` with dev/stg/prd sections; `API_ORIGIN` mapping; request ID middleware on Workers.
- Test: `/api/health` via Workers returns 200 with correlation ID.
- Depends: INF-001.

[DB-001] Base schema (single-owner)
- Build: Migrations for Preferences, ResumeProfile, JobPosting, Application, EmailThread, SheetRow, CalendarEvent, CrawlLog.
- Test: Apply migrations; insert sample rows; constraints (unique dedupe_key) enforced.
- Depends: none.

[API-001] ASP.NET baseline + OpenAPI
- Build: `/health`, `/version`, `/openapi.json`.
- Test: 200 responses; schema browsable.
- Depends: DB-001.

[EDGE-001] Workers proxy to API
- Build: Proxy `/api/*` to origin with timeout/backoff; add rate-limit per path.
- Test: `/api/health` works through edge; rate-limit triggers at threshold.
- Depends: API-001, INF-002.

[FE-001] Next.js 15 baseline (no auth)
- Build: SSR page with “Config” and “Dashboard” routes; call API for mock data.
- Test: Page renders server-side; mock data visible.
- Depends: EDGE-001.

[CFG-001] Preferences API + UI
- Build: CRUD endpoints and simple form (categories, radius, remote, keywords, salary floor).
- Test: Save, reload, and read back identical values.
- Depends: FE-001, API-001, DB-001.

[JOB-BASE] Hangfire setup
- Build: Server, Dashboard (protected via shared secret header), queues, retry policy, DLQ concept.
- Test: Enqueue dummy job; success, failure, retry paths visible in dashboard.
- Depends: API-001, DB-001.

[CRAWL-001] Playwright foundation
- Build: Base crawler class (UA, polite delays, wait-for-selectors, timeout, snapshot).
- Test: Fetch a known static page; log DOM capture and screenshot.
- Depends: JOB-BASE.

[CRAWL-002] Normalization & dedupe
- Build: Mapping contract for a JobPosting; dedupe_key rules (source+external_id or URL hash).
- Test: Insert duplicates; only one JobPosting persists.
- Depends: DB-001, CRAWL-001.

[CRAWL-003] Source A adapter
- Build: Query builder, pagination, item extraction, normalization, save to DB.
- Test: With Preferences, collect ≥10 items; log success/fail rate; dedupe < 10%.
- Depends: CRAWL-002.

[CRAWL-004] Source B adapter
- Build: Same as A for a second site.
- Test: Combined unique count increases; overlap ratio measured.
- Depends: CRAWL-002.

[MATCH-001] Fit scoring v1
- Build: Keyword match with weights (ResumeProfile × JobPosting); threshold + top-K selection.
- Test: For a target role, top results look reasonable in manual spot-check.
- Depends: DB-001, CRAWL-002.

[RESUME-001] ResumeProfile extractor
- Build: Upload endpoint (PDF/DOCX) → text extract → key skills summary stored.
- Test: Known resume yields expected key phrases; re-upload updates summary.
- Depends: API-001, DB-001.

[GEN-001] Cover letter template (Semantic Kernel)
- Build: Input contract (JobPosting + ResumeProfile + optional company signals) → output contract (sections, length, tone).
- Test: 3 job types produce distinct, non-repetitive letters without hallucinated claims.
- Depends: MATCH-001, RESUME-001.

[MAIL-TX-LCL-001] Local SMTP sender (MailPort)
- Build: Send email with subject, body, attachments (resume). Persist a send hash per application to ensure idempotency.
- Test: Deliver to a test mailbox; verify headers/body; ensure repeated attempts do not duplicate (hash guard).
- Depends: INF-001, GEN-001, STATE-001.

[STATE-001] Application state machine
- Build: Enforce legal transitions; hooks to call Ports on transitions (send, sheet update, calendar).
- Test: Unit test every transition and block illegal ones.
- Depends: DB-001.

[SEND-001] Application send job
- Build: Pull top-K matches above threshold; generate cover if missing; lock application; send via MailPort; transition to 'emailed'.
- Test: Dry-run (no send) vs real send; in real send, status becomes 'emailed' once; logs contain correlation IDs.
- Depends: GEN-001, MAIL-TX-LCL-001, STATE-001, JOB-BASE.

[UI-DASH-001] Dashboard v1
- Build: Table with Role, Company, FitScore, Status, LastUpdate, links to posting and email thread IDs.
- Test: After SEND-001, rows appear and update on refresh.
- Depends: FE-001, STATE-001.

[SHEET-LCL-001] Local sheet adapter (DB rows)
- Build: Upsert a 'SheetRow' per application; maintain a consistent column mapping.
- Test: State change updates the row (idempotent upsert).
- Depends: STATE-001, DB-001.

[SHEET-CSV-001] CSV exporter
- Build: Periodic job writes CSV snapshot to `CSV_OUT_DIR`; stable headers.
- Test: Edit an application; next export reflects change; no duplicate rows.
- Depends: SHEET-LCL-001, JOB-BASE.

[CAL-ICS-001] ICS generator
- Build: Create `.ics` files for interviews; include title, description, location/meeting link, reminders.
- Test: Import ICS into a calendar client; time zone correct; update regenerates file.
- Depends: STATE-001.

[MAIL-RX-IMAP-001] IMAP puller (receive sync)
- Build: Connect to inbox/folder; incremental fetch; basic intent detection (interview/offer/reject).
- Test: Seed mailbox with sample replies; correct transitions to 'interview', 'offer', 'rejected'.
- Depends: INF-001, STATE-001, DB-001.

[AUTO-LOOP-001] Orchestrated loop
- Build: Recurring jobs for crawl (A,B), scoring, generate, send, receive sync, sheet export, ics updates; concurrency guard (single-owner).
- Test: 24-hour burn: logs show cycles; no unbounded retries; idempotency holds.
- Depends: CRAWL-003/004, MATCH-001, GEN-001, SEND-001, MAIL-RX-IMAP-001, SHEET-CSV-001, CAL-ICS-001.

[SAFE-IDEMP-001] Idempotency & duplicate prevention
- Build: Business IDs and send-hashes; upsert semantics; lock per application when acting.
- Test: Concurrent re-runs do not duplicate emails or rows; race tests pass.
- Depends: SEND-001, SHEET-LCL-001.

[OBS-001] Observability & alerts (local)
- Build: Structured logs with correlation IDs; metrics for success rates, retries, 4xx/5xx; alert on high failure rate.
- Test: Force failures; ensure alert triggers and quiets once recovered.
- Depends: JOB-BASE.

[UX-CFG-001] Config UX polish (personal mode)
- Build: One-page config with live validation and helpful defaults; tooltips for crawl safety and email etiquette.
- Test: Change config; next cycles obey new settings without restart.
- Depends: CFG-001, FE-001.

— Up to here: Phases 0–3 complete (no Google; real email via SMTP, real replies via IMAP, local CSV/ICS) —

[GOOG-BOOT-001] Google Cloud project (optional last)
- Build: Project + OAuth consent (test user only).
- Test: Consent screen shows; authorization code returned.
- Depends: none (for Google branch).

[GOOG-MAIL-TX-001] Gmail sender adapter
- Build: Draft→Send; thread_id capture; label strategy (Jobs/*).
- Test: Real send; correct labels; thread tracking.
- Depends: GOOG-BOOT-001, Ports.

[GOOG-MAIL-RX-001] Gmail receive adapter
- Build: Incremental sync; map threads to Applications; intent detection.
- Test: Sample replies drive correct transitions.
- Depends: GOOG-MAIL-TX-001, STATE-001.

[GOOG-SHEET-001] Google Sheets adapter
- Build: Column mapping; idempotent upsert; error handling on rate limits.
- Test: Local SheetRow mirrors to Google; no duplicates.
- Depends: GOOG-BOOT-001, Ports.

[GOOG-CAL-001] Google Calendar adapter
- Build: Event create/update; reminders; attendees optional.
- Test: Events visible and editable in Google Calendar.
- Depends: GOOG-BOOT-001, Ports.

[MIG-001] Switch & migration
- Build: Feature flags to move from local providers to Google; one-time sync (DB/CSV → Sheets; ICS → Calendar).
- Test: Staging switch shows identical dashboard; no double-writes; backout plan validated.
- Depends: All GOOG-*.

---

## 8. E2E Scenarios (observable checkpoints)

Scenario A: First run to first application (no Google)
- Set preferences and upload resume.
- Crawl collects ≥10 postings; top-K scored; letters generated; one email sent.
- Dashboard shows 'emailed'; CSV contains row; logs show correlation IDs.

Scenario B: Reply to interview (IMAP)
- Place a reply email containing “interview”.
- IMAP sync detects; Application → 'interview'; ICS generated.
- Dashboard and CSV reflect change; ICS opens correctly.

Scenario C: Timeout follow-up
- After N days without reply, a follow-up template is resent once (configurable).
- Status 'awaiting' persists; 'LastUpdate' changes; no duplicate threads.

Scenario D: Failure & recovery
- Cause SMTP failure; ensure retries; if exceeded, DLQ with manual retry button in UI.
- Upon recovery, only one successful send event recorded.

Scenario E (optional): Feature flag switch to Google
- Flip providers in staging; run migration; verify equality of rows and events; then flip in prod.

---

## 9. Non-Functional Requirements
- Idempotency: All write operations safe on retry (send hashes, upserts, locks).
- Safety: Crawl with politeness delays; respect robots and site TOS according to your policy; configurable.
- Reliability: Automatic backoff; DLQ and manual replays.
- Performance: Batching for CSV; bounded concurrency (single owner → 1–2 workers).
- Security: Secrets only via env or Workers secrets; no PII beyond minimal metadata stored.
- Time zone: America/Los_Angeles for all scheduling; DST-aware.

---

## 10. Risk & Compliance Notes (personal use)
- Crawling: Each site’s terms may restrict automation. Keep sources you personally use and configure rate limits; maintain a blocklist. Fail gracefully on anti-bot challenges.
- Email reputation: Stagger sends, vary subjects, and include proper signatures to avoid spam flags.
- Data loss: CSV/ICS outputs serve as human-readable backups; periodic DB dumps recommended.

---

## 11. Deliverables (from a single GitHub repo)
- Monorepo with the layout above.
- Running system (dev/stg/prod) deployable via one command per environment.
- No-login owner mode: config UI, dashboard, jobs.
- Local providers fully working (SMTP/IMAP + CSV + ICS).
- Optional Google adapters behind feature flags, plus migration script.
- Runbooks: setup, rotate secrets, deploy, rollback, crawl-safety checklist.

---

## 12. LLM Hand-off Guidance (for each card)
- Use the card’s “Build” as the minimal PR scope and “Test” as acceptance checks.
- Write/update a changelog entry per card; keep migrations reversible.
- Add idempotency tests early (before wiring the job into cron flows).
- Prefer configuration over code changes for behavior toggles (feature flags).


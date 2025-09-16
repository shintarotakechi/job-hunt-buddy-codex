# REQUIREMENTS v0.1 — Single-URL Job Canonicalizer (Personal, No Login, Gemini-2.5-flash, Fully Localhost-Testable)

Purpose: When you paste an aggregator job URL (e.g., an Indeed viewjob link), the system must 1) fetch and read that exact HTML, 2) have the LLM extract core facts only from what’s there, 3) discover the official company site, 4) navigate to the Careers/ATS section and find the same role (canonical JD), 5) re-extract and override fields from the canonical page, and 6) output (A) a single-line tab-delimited row (20 fields) and (B) a vertical keyword list. No guessing. No email/Sheets/Calendar yet. No login. LLM is Gemini-2.5-flash.

Stack: Next.js 15 (frontend), ASP.NET Web API (.NET 8) (backend), Playwright.NET (crawler), Gemini-2.5-flash (via Gemini API key), PostgreSQL (storage), Cloudflare Workers + Wrangler (optional dev proxy), Hangfire disabled in v0.1.

Localhost topology: API http://localhost:5050, Frontend http://localhost:3000, Workers proxy http://localhost:8787 (→ API), PostgreSQL localhost:5432.

------------------------------------------------------------------------------------------------------------------------------------

0) Definition of Done (v0.1)

A) From a clean local setup, POST /extract with a reachable Indeed URL returns:
• rowTabDelimited: exactly 20 fields separated by real tabs, unknowns as n/a, dates in MM/DD/YYYY, nextActionDate = today + 7 days (America/Los_Angeles default).
• keywordsBlock: 10–25 items, one per line, appearing in JD text (no invented tech).
• source.detected and source.canonicalUrl, plus a resolvable auditId that shows HTML and screenshot of both pages fetched.

B) Canonical JD (company site or company-controlled ATS) overrides aggregator values.

C) Salary, contacts, workMode, jobLevel only populate if explicitly present on official pages; otherwise n/a.

------------------------------------------------------------------------------------------------------------------------------------

1) From-Scratch Bootstrapping (Initial Setup Cards)

INIT-001 Repository and Workspace Skeleton
What to build:
• Create a new Git repository with a monorepo layout: root contains apps, packages, infra, docs.
• Add top-level files: LICENSE, README.md (empty skeleton), .gitignore (node_modules, bin, obj, .env*, audit artifacts), .editorconfig (UTF-8, LF, 2 spaces for JS/TS, 4 for C#), .gitattributes (text=auto).
• Create directories:
  - apps/frontend
  - apps/api
  - packages/domain
  - packages/extractors
  - packages/formatters
  - packages/edge
  - infra
  - docs
How to test:
• git status shows only expected files and folders.
• README exists; .gitignore prevents accidental binary/env commits.

INIT-002 Toolchain Verification
What to build:
• Decide and document required versions: Node 20+, pnpm 9+, .NET SDK 8+, PostgreSQL 14+, Wrangler CLI latest, Playwright browsers installed, curl, jq (optional).
How to test:
• Run version checks and record outputs in docs/ENVIRONMENT.md.
• Confirm PostgreSQL server reachable on localhost:5432.

INIT-003 Workspace Config Baselines
What to build:
• Frontend package manager: plan to use pnpm workspaces; create a workspace manifest at repo root naming apps and packages. Do not commit secrets.
• .NET solution structure decision: a solution in apps/api that references code from packages/* via project references.
• Document naming conventions for namespaces, folders, DTOs, and services in docs/NAMING.md.
How to test:
• pnpm detects workspace packages; dotnet list solution works once projects exist (later card).

INIT-004 Environment Variable Catalog
What to build:
• Define environment keys in docs/ENV_VARS.md (names only, no secrets):
  DB_CONN, TZ_DEFAULT, PLAYWRIGHT_HEADLESS, PLAYWRIGHT_NAV_TIMEOUT_MS, PLAYWRIGHT_POLITE_DELAY_MS_MIN, PLAYWRIGHT_POLITE_DELAY_MS_MAX, USER_AGENT_OVERRIDE, GEMINI_API_KEY, GEMINI_MODEL_ID, FEATURE_USE_HANGFIRE, SCREENSHOT_DIR, HTML_DIR, API_ORIGIN, ROUTE_RATE_LIMIT, FETCH_MAX_REDIRECTS, REQUEST_BUDGET_MS.
• Create empty .env.example files: apps/api/.env.example with keys and comments.
How to test:
• .env.example exists; keys match the catalog; .gitignore excludes .env.

INIT-005 Local Database Bootstrap
What to build:
• Decide on DB name “jobs”, local user, and connectivity policy.
• Add infra/DB_SETUP_NOTES.md with steps for creating DB and user, and a reminder to apply migrations later.
How to test:
• psql to localhost connects; “jobs” database exists and is empty.

INIT-006 Runbook Seeds
What to build:
• docs/RUNBOOK_LOCAL.md with the intended start sequence (API, FE, optional Workers), and smoke tests (health endpoint, curl examples).
• docs/TEST_CHECKLIST.md initial list covering paste-into-Excel verification.
How to test:
• Runbook documents exist and read coherently end-to-end.

------------------------------------------------------------------------------------------------------------------------------------

2) Data Model (entities; no SQL in this file)

AuditArtifact
• id (UUID)
• createdAt (timestamp with zone)
• sourceUrl (text)
• canonicalUrl (text or null)
• htmlPath (local file path)
• screenshotPath (local file path)
• detector (string tag for logging: aggregator or company/ATS context)

ExtractionResult (one per /extract request)
• id (UUID)
• artifactId (UUID → AuditArtifact)
• companyName (text)
• jobTitle (text)
• jobSourceUrl (text)
• hiringManager (text)
• phone (text)
• email (text)
• stage (text enum)
• nextActionDate (date)
• nextActionNote (text)
• lastUpdated (date)
• interviewDirection (text)
• interviewTags (text)
• salaryRange (integer range min–max; USD only)
• workMode (enum text)
• jobLevel (text)
• applicationFiles (text)
• version (int; default 1)
• createdAt (date)
• companyCareerUrl (text)
• dataConfidence (enum: High, Medium, Low)

Notes:
• salaryRange is populated only when the page explicitly shows a USD annual min and max. Non-USD or hourly are ignored in v0.1.
• workMode and jobLevel require literal tokens in the text; otherwise n/a.

------------------------------------------------------------------------------------------------------------------------------------

3) Output Contract (Excel row and Keywords)

Excel row (20 fields; exactly 19 tabs; no header; no embedded newlines)
1 companyName
2 jobTitle
3 jobSourceUrl
4 hiringManager (FullName — Title if explicit; else n/a)
5 phone (+1-###-###-#### if explicit; else n/a)
6 email (RFC 5322 if explicit; else n/a)
7 stage (Seen default)
8 nextActionDate (MM/DD/YYYY; today + 7 in timezone)
9 nextActionNote (default “Research company page & verify JD match”)
10 lastUpdated (MM/DD/YYYY; today)
11 interviewDirection (n/a v0.1)
12 interviewTags ({…} or n/a)
13 salaryRange (integer “min–max”, USD range only; else n/a)
14 workMode (On-site | Hybrid | Remote only if explicit; else n/a)
15 jobLevel (Senior | Staff | Principal | Lead | L5 | Manager only if explicit; else n/a)
16 applicationFiles (n/a v0.1)
17 version (“1”)
18 createdAt (MM/DD/YYYY; today)
19 companyCareerUrl (canonical JD URL or n/a)
20 dataConfidence (High | Medium | Low)

Sanitization rules:
• Remove tabs and newlines from all fields; collapse multi-spaces.
• Guarantee exactly 20 fields no matter how many are n/a.
• Dates in MM/DD/YYYY.

Keywords block:
• 10–25 items; one per line; must appear in JD or be clear synonyms; no trailing newline.

------------------------------------------------------------------------------------------------------------------------------------

4) End-to-End Flow (aggregator → canonical JD)

A Fetch aggregator page
• Use Playwright.NET to load the exact URL (e.g., Indeed viewjob).
• Save outer HTML and full-page PNG; create AuditArtifact (detector tag “aggregator/indeed”).
• If blocked or timeout, return FETCH_FAILED with guidance to pass htmlFallback.

B LLM extraction (aggregator page)
• Feed visible text into Gemini-2.5-flash; extract companyName, jobTitle, workMode, jobLevel, salary snippets, keywords. Enforce evidence; without explicit evidence, return n/a.
• Normalize salary only if explicit USD min–max range is present.

C Company domain discovery
• From the aggregator HTML and LLM companyName, derive the official domain, preferring links in the page. If multiple candidates, prefer .com/.ai domain containing brand token.
• Record chosen domain for traversal.

D Careers/ATS discovery on company domain
• Start at company root; try standard paths (/careers, /jobs, /join) and nav anchors (“Careers”, “Jobs”, “Join Us”, “Open Roles”).
• Allow ATS links from the company site (Greenhouse, Lever, Workday). Limit BFS depth to 2 clicks from root; remain within the registrable domain or ATS linked from it.

E Role matching on careers/ATS
• Find a JD page whose title tokens similarity to the aggregator title is ≥ 0.75; prefer explicit “Apply” presence and recent posting hints. If ties, choose H1 that contains all major title tokens.
• If not found within budget, canonicalUrl remains n/a.

F Canonical fetch and re-extraction
• Fetch the canonical page; save HTML/PNG; create secondary AuditArtifact (detector tag “company/ats”).
• Re-run LLM extraction on canonical HTML; any conflicting fields are overridden by canonical values.

G Strict contact discovery (canonical domain only)
• Scan the canonical JD and up to two linked pages on the same domain (Contact, Team, Recruiting) for explicit hiringManager (FullName + Title), recruiting email (mailto or visible RFC 5322), and US phone; normalize to +1-###-###-####. If not explicit, set n/a.

H Confidence scoring
• High: canonical found on company/ATS, title/company match, fields taken from canonical.
• Medium: no canonical found, aggregator extraction consistent and unambiguous.
• Low: ambiguous signals or insufficient content.

I Compose outputs and persist
• Compute dates: createdAt and lastUpdated = today; nextActionDate = today + 7 (timezone aware).
• Compose the 20-field tab line and keywords block; persist ExtractionResult linked to AuditArtifact(s).
• Return rowTabDelimited, keywordsBlock, auditId (primary artifact), and source info.

------------------------------------------------------------------------------------------------------------------------------------

5) LLM Usage — Gemini-2.5-flash (no code here)

Inputs:
• visibleTitle, visibleCompany, visibleBody (plain text stripped from HTML), any JSON-LD as text, source context (aggregator vs company), and the aggregator title to support canonical matching.

Outputs (JSON-style fields enforced by client validator):
• companyName (string or n/a)
• jobTitle (string)
• workMode (Remote | Hybrid | On-site | n/a) plus a short evidence quote when not n/a
• jobLevel (Senior | Staff | Principal | Lead | L5 | Manager | n/a) with evidence quote when not n/a
• salarySnippets (array of matched text lines that include explicit USD ranges; may be empty)
• keywords (10–25 items)
Validation:
• If no evidence quoted for workMode/jobLevel, set them to n/a.
• Parse salary from salarySnippets only if USD min–max; strip $, commas; output integers; else n/a.
• Remove tabs/newlines from all strings before formatting.

Budget:
• LLM call target ≤ 15 s; overall request budget ≤ REQUEST_BUDGET_MS (default 25 s). On overrun, return TIMEOUT with retry guidance.

------------------------------------------------------------------------------------------------------------------------------------

6) API Contract (no source code)

Endpoint:
• POST /extract
Request:
• url (required), htmlFallback (optional plain text or HTML if fetch blocked), timezone (optional; default TZ_DEFAULT).
Success (200):
• rowTabDelimited, keywordsBlock, auditId, source.detected (free-text tag), source.canonicalUrl (string or n/a).
Errors (>=400):
• { error, code } where code ∈ BAD_URL, FETCH_FAILED, UNSUPPORTED, PARSE_FAILED, TIMEOUT.
Audit retrieval:
• GET /audit/{auditId} shows basic metadata, screenshot, and raw HTML link for verification (personal mode; no auth).

------------------------------------------------------------------------------------------------------------------------------------

7) Minimal UI (Next.js 15)

Requirements:
• Single page with URL input and Submit button.
• Results area with two copyable boxes: 20-field tab line; vertical keywords.
• “Open audit” link navigates to audit viewer page.
• Error UX: if FETCH_FAILED or TIMEOUT, show textarea to paste htmlFallback and retry.
Accessibility:
• Disable submit while running; announce success/failure via ARIA live region; monospace display for both outputs.

------------------------------------------------------------------------------------------------------------------------------------

8) Workers Dev Proxy (optional but recommended)

Behavior:
• Proxy /api/* to API_ORIGIN; inject X-Request-Id (uuid); apply ROUTE_RATE_LIMIT.
• Abort after FETCH_MAX_REDIRECTS; pass upstream 429/5xx through.
Local:
• Dev server listens at http://localhost:8787; forwards to http://localhost:5050.

------------------------------------------------------------------------------------------------------------------------------------

9) Testing Strategy (what to build and how to verify — no code)

Unit
• Url normalization: strips common tracking params while preserving essential ATS query strings.
• Salary normalization: “$168,000–$240,000” → 168000–240000; hourly or non-USD ignored.
• Title similarity: token-based similarity meets ≥ 0.75 for positive pairs, < 0.75 for negatives.
• Sanitizers: remove tabs/newlines; always emit exactly 20 fields (19 tabs); MM/DD/YYYY dates.

Integration (HTML fixtures, no live web)
• Aggregator extractor on three Indeed fixtures:
  - Case A salary present; Case B salary absent; Case C explicit “Remote”.
  - LLM returns workMode/jobLevel only with evidence; salary parsed only for USD range.
• Canonical finder:
  - From aggregator fixture to company careers/ATS fixture; canonicalUrl correctly chosen; canonical overrides title/company.
• Contacts finder:
  - Positive fixture with explicit recruiter or hiring manager fields populates; negative fixture yields n/a.

E2E (live or replayed through Playwright)
• Happy path:
  - POST a reachable Indeed URL; receive row (20 fields) and keywords (10–25 lines); paste row into Excel and verify column alignment and date parsing; confirm audit assets match extracted values.
• Blocked page:
  - Returns FETCH_FAILED; supply htmlFallback and retry; receive valid outputs without guessing.
• No canonical:
  - Confidence = Medium; companyCareerUrl = n/a; row still valid with aggregator fields.

Operational
• Time budget: long or hung pages return TIMEOUT; no partial guessing.
• Proxy rate-limit triggers at configured threshold and returns clear message.

------------------------------------------------------------------------------------------------------------------------------------

10) Implementation Cards (build and test; dependency-ordered)

INIT-001 Repository and Workspace Skeleton
• Build: Create monorepo folders and top-level files enumerated above.
• Test: git status clean; .gitignore covers audit and env files.

INIT-002 Toolchain Verification
• Build: Record Node, pnpm, .NET, Postgres, Wrangler, Playwright versions in docs.
• Test: Commands succeed; versions meet minimums.

INIT-003 Workspace Config Baselines
• Build: Root workspace manifest decision documented; .NET solution plan in docs.
• Test: pnpm recognises workspace structure; solution plan consistent.

INIT-004 Environment Variable Catalog
• Build: docs/ENV_VARS.md and apps/api/.env.example with all keys.
• Test: Keys present and documented; .env ignored by git.

INIT-005 Local Database Bootstrap
• Build: DB “jobs” created; user and permissions documented.
• Test: Can connect locally; empty schema ready.

INIT-006 Runbook Seeds
• Build: docs/RUNBOOK_LOCAL.md and docs/TEST_CHECKLIST.md created with the intended flow and smoke checks.
• Test: Read-through confirms coherent single-operator run.

API-001 POST /extract scaffold
• Build: Controller accepts url, htmlFallback, timezone; validates url; starts correlation ID; enforces REQUEST_BUDGET_MS envelope.
• Test: BAD_URL for malformed input; 200 stub for valid input.

CRAWL-001 Playwright bootstrap and capture
• Build: Singleton Chromium; per-request context; UA override; save HTML and PNG under configured dirs; create AuditArtifact with “aggregator” detector tag.
• Test: Fetches a public page; files exist; artifact recorded.

LLM-001 Gemini client and JSON-style schema
• Build: Client for gemini-2.5-flash returning companyName, jobTitle, workMode, jobLevel, salarySnippets, keywords, with short evidence quotes for mode/level; client-side validation applies n/a without evidence.
• Test: Controlled text samples produce correct outputs; absent evidence results in n/a.

PARSE-AGG-001 Aggregator extraction (LLM-first)
• Build: From aggregator HTML, extract preliminary fields using LLM; parse salary only for explicit USD ranges; sanitize fields.
• Test: Three Indeed fixtures yield expected values; no salary when not explicit.

CANON-001 Company domain locator
• Build: Determine official domain from aggregator HTML links and companyName; prefer brand token matches; choose .com/.ai if multiple.
• Test: Fixtures resolve correct root domain.

CANON-002 Careers/ATS discovery
• Build: From root domain, find Careers/Jobs/Join pages or ATS links; BFS depth ≤ 2 on same registrable domain or ATS linked from it.
• Test: Company fixtures resolve to careers page; if absent, return n/a for canonical.

CANON-003 Role match finder
• Build: On careers/ATS, compute title similarity ≥ 0.75 to select the JD; prefer “Apply” presence; apply tie-breakers.
• Test: Positive fixtures choose the correct JD; negatives yield no match.

PARSE-CANON-001 Canonical extraction and override
• Build: Fetch canonical JD; capture artifacts; LLM re-extract; override aggregator fields with canonical ones where explicit.
• Test: Canonical overrides title/company; salary accepted only if explicit USD range.

CONTACT-001 Strict contact discovery
• Build: On canonical domain, scan JD and up to two linked pages for explicit FullName + Title, recruiting email, and US phone; normalize phone; else n/a.
• Test: Positive and negative fixtures behave accordingly.

CONF-001 Confidence calculator
• Build: Assign High/Medium/Low per rules; set companyCareerUrl accordingly.
• Test: Canonical found and matching → High; none → Medium; ambiguous → Low.

FORMAT-001 Row and keywords composition
• Build: Compute createdAt, lastUpdated, nextActionDate; sanitize; produce exactly 20 fields with 19 tabs; produce keywords block lines.
• Test: Paste row into Excel; columns align; dates parse; tab count verified.

API-002 Pipeline wiring
• Build: Chain A→I steps: fetch aggregator → LLM → company domain → careers → role match → canonical fetch → LLM override → contacts → confidence → compose → persist → respond.
• Test: E2E against three real URLs; outputs and artifacts correct.

UI-001 Minimal frontend
• Build: URL input, submit, results boxes with copy buttons, audit link; fallback htmlFallback textarea on FETCH_FAILED.
• Test: Manual run; copy-paste row into Excel; keywords list vertical; audit opens.

AUDIT-001 Audit viewer
• Build: GET /audit/{auditId} shows screenshot, HTML link, and metadata (sourceUrl, canonicalUrl, detector, createdAt).
• Test: Open latest audit; visuals match extracted values.

SAFE-001 Workers proxy and rate-limit (optional)
• Build: Proxy /api/* to API_ORIGIN; inject X-Request-Id; apply ROUTE_RATE_LIMIT; enforce FETCH_MAX_REDIRECTS.
• Test: Health via proxy; burst to trigger rate-limit returns clear message.

Dependencies overview:
• INIT-001 → INIT-002 → INIT-003 → INIT-004 → INIT-005 → INIT-006 precede all implementation.
• API-001 depends on INIT-*.
• CRAWL-001 depends on API-001.
• LLM-001 depends on INIT-004 (env) and API-001.
• PARSE-AGG-001 depends on CRAWL-001 and LLM-001.
• CANON-001 depends on PARSE-AGG-001.
• CANON-002 depends on CANON-001.
• CANON-003 depends on CANON-002.
• PARSE-CANON-001 depends on CANON-003 and LLM-001.
• CONTACT-001 depends on PARSE-CANON-001.
• CONF-001 depends on PARSE-CANON-001 (and CONTACT-001 if contacts were found).
• FORMAT-001 depends on PARSE-CANON-001, CONF-001.
• API-002 depends on all prior core cards.
• UI-001 depends on API-002.
• AUDIT-001 depends on CRAWL-001 and API-001.
• SAFE-001 depends on API-001 (optional).

------------------------------------------------------------------------------------------------------------------------------------

11) Guardrails (must-keep)

• Never guess contacts, salary, workMode, or jobLevel; populate only when explicit on the page. Otherwise n/a.
• Canonical JD must be on the company’s registrable domain or an ATS page linked from that domain; do not scrape arbitrary search engine results in v0.1.
• LLM outputs must include short evidence quotes for workMode and jobLevel when not n/a; lack of explicit evidence forces n/a.
• Salary must be an explicit USD annual range with both min and max; hourly or non-USD is ignored.
• Always emit exactly 20 fields in the row; sanitize tabs and newlines; MM/DD/YYYY for dates; nextActionDate = today + 7 days (America/Los_Angeles by default).

------------------------------------------------------------------------------------------------------------------------------------

12) Operator Runbook (local; no code)

Prereqs:
• Node 20+, pnpm 9+, .NET SDK 8+, PostgreSQL 14+, Wrangler (optional), Playwright browsers installed, a valid GEMINI_API_KEY.

Start sequence:
• Ensure DB “jobs” exists.
• Set apps/api/.env based on .env.example (without committing secrets).
• Start API on http://localhost:5050.
• Start frontend on http://localhost:3000.
• Optionally run Workers dev proxy on http://localhost:8787.

Smoke checks:
• GET /health returns 200.
• POST /extract with a reachable Indeed URL returns rowTabDelimited, keywordsBlock, auditId; paste row into Excel and verify columns and dates.
• Open /audit/{auditId} and verify screenshot/HTML align with extracted fields.
• Try a blocked URL; confirm FETCH_FAILED; paste page text into htmlFallback and retry successfully.

Next increments after v0.1:
• Add a second extractor (your next most common ATS).
• Add batch mode with Hangfire to queue multiple URLs.
• Add CSV export of rows for manual backups.
• Add Google adapters (Gmail, Sheets, Calendar) behind feature flags later.


# REQUIREMENTS_v0.1 — Single-URL Job Canonicalizer (Personal, No Login, Google Later)
Scope-narrowed, implementation-ready specification you can hand to an LLM/agent to build from scratch. The goal is to ingest ONE pasted job URL (e.g., Indeed), locate the canonical company career-page version of the same job, extract ONLY verified facts, and return:
(1) a single-line, tab-delimited row matching your schema; (2) a vertical list of resume keywords.
No email sending, no Google Sheets/Calendar yet. Zero hallucination rules apply.

Stack (unchanged): Next.js 15 (frontend) · ASP.NET Web API (backend) · Playwright.NET (crawler) · Semantic Kernel (LLM) · Hangfire (optional later) · Cloudflare Workers + Wrangler (edge) · PostgreSQL (storage)

-------------------------------------------------------------------------------

## 0) Non-Goals (for v0.1)
- No multi-user features, no signup/login, no billing.
- No bulk queue UI (internally allowed later), no emailing, no Google integrations.
- Do not guess contacts/salary/level/mode; return `n/a` unless explicitly present on official/canonical pages.

-------------------------------------------------------------------------------

## 1) Contracts — Input/Output & Field Rules

1.1 Endpoint (backend)
- Method: POST /extract
- Request JSON:
    {
      "url": "https://…",               // REQUIRED. Public job URL
      "htmlFallback": null,             // OPTIONAL. If page can’t be fetched, plain visible text/HTML pasted by operator
      "timezone": "America/Los_Angeles" // OPTIONAL. Default America/Los_Angeles
    }
- Response JSON (success 200):
    {
      "rowTabDelimited": "<single-line-with-real-tabs>",  // 1 line; tabs between fields; no header
      "keywordsBlock": "keywordA\nkeywordB\n...",         // 10–25 lines, \n separated
      "auditId": "uuid",                                  // retrieve raw artifacts
      "source": { "detected": "indeed|greenhouse|lever|workday|company|unknown", "canonicalUrl": "https://…|null" }
    }
- Response JSON (failure 4xx/5xx):
    { "error": "MESSAGE", "code": "FETCH_FAILED|UNSUPPORTED|PARSE_FAILED|TIMEOUT|BAD_URL" }

1.2 Excel Row Field Order (exact 20 fields)
Return exactly 20 columns separated by REAL tab characters (U+0009). No `\t` literals. No embedded newlines. Use `n/a` when unknown.
- companyName
- jobTitle
- jobSourceUrl                      // the original pasted URL
- hiringManager                     // "Full Name — Title" only if explicit on official domain; else n/a
- phone                              // +1-###-###-####, only if explicit; else n/a
- email                              // RFC 5322, only if explicit; else n/a
- stage                              // default "Seen"
- nextActionDate                     // MM/DD/YYYY, today+7 in timezone
- nextActionNote                     // default "Research company page & verify JD match"
- lastUpdated                        // MM/DD/YYYY (date-only, Excel-friendly)
- interviewDirection                 // markdown blob; v0.1 set n/a
- interviewTags                      // e.g., {AI Demo, System Design}; else n/a
- salaryRange                        // int4range "min–max" with integers; only if explicit $min–$max; else n/a
- workMode                           // On-site | Hybrid | Remote; only if explicit; else n/a
- jobLevel                           // e.g., Senior|Staff|L5; only if explicit; else n/a
- applicationFiles                   // n/a for v0.1
- version                            // "1"
- createdAt                          // MM/DD/YYYY (date-only)
- companyCareerUrl                   // discovered canonical JD URL or n/a  ← added to preserve provenance
- dataConfidence                     // High|Medium|Low based on checks; v0.1 rules below

1.3 Keywords Block
- 10–25 lines, each a single keyword/phrase (no commas unless the phrase includes one).
- Derived from the canonical JD (or aggregator if canonical not found). Do NOT invent technologies not present.

1.4 Data Confidence (for provenance)
- High: canonicalUrl is on company/ATS domain; title/company match; fields sourced from canonical.
- Medium: canonicalUrl not found; aggregator fields are consistent and unambiguous.
- Low: partial fields extracted; significant ambiguity or missing matches.

-------------------------------------------------------------------------------

## 2) Monorepo Layout (paths and responsibilities)

- /apps/frontend
  - Next.js 15 app (App Router). Routes:
    - / (URL form, results render, copy buttons)
    - /audit/[id] (simple viewer for screenshot/HTML)
- /apps/api
  - ASP.NET Web API project with controllers/services
  - Playwright bootstrap (singleton per process)
  - Semantic Kernel client and prompt templates
  - Data access (Npgsql)
- /packages/domain
  - DTOs: ExtractRequest/Response, ParsedFields, Confidence
  - Value objects: Phone, Email, MoneyRange, DateOnlyUS
  - State: not needed in v0.1 (keep for future)
- /packages/extractors
  - SourceDetector, HtmlNormalizer, SalaryParser, WorkModeClassifier, LevelClassifier
  - Extractors: IndeedExtractor, AtsExtractorGeneric, JsonLdJobPostingParser
  - CanonicalFinder: links traversal, ATS patterns
- /packages/formatters
  - ExcelRowFormatter, KeywordsFormatter, Sanitizers (strip newlines/tabs)
- /packages/edge
  - Cloudflare Workers handler (proxy /api/*, correlation, rate-limit)
- /infra
  - wrangler.toml templates, docker/docker-compose.local.yml, Makefile targets, runbooks
- /docs
  - This file, ERD.txt, Prompts.md, Test-fixtures.md

-------------------------------------------------------------------------------

## 3) Environments & Secrets (names, examples, rules)

All secrets via environment variables or Workers secrets. NEVER commit secrets.

- API_ORIGIN=https://api.example.com
- DB_CONN=Host=…;Port=5432;Database=jobs;Username=…;Password=…
- PLAYWRIGHT_HEADLESS=true
- PLAYWRIGHT_NAV_TIMEOUT_MS=30000
- PLAYWRIGHT_POLITE_DELAY_MS=800..1600   // randomized window
- USER_AGENT_OVERRIDE="Mozilla/5.0 …"
- FETCH_MAX_REDIRECTS=5
- TZ_DEFAULT=America/Los_Angeles
- LLM_MODEL="gpt-*-reasoning|claude-*"   // Semantic Kernel bound
- LLM_TEMPERATURE=0.1
- LLM_MAX_TOKENS=1200
- ROUTE_RATE_LIMIT=30/min
- SCREENSHOT_DIR=/var/app/audit/png
- HTML_DIR=/var/app/audit/html
- FEATURE_USE_HANGFIRE=false

-------------------------------------------------------------------------------

## 4) Cloudflare Workers (edge proxy) — Operational Spec

Routes:
- GET /api/health → proxy to API_ORIGIN
- POST /api/extract → proxy to API_ORIGIN with timeout 35s

Behavior:
- Inject X-Request-Id (uuidv4) if absent; echo back in responses.
- Rate-limit by IP: ROUTE_RATE_LIMIT.
- Abort on > FETCH_MAX_REDIRECTS.
- On 429/5xx from origin, pass through with same status.

wrangler.toml (excerpt; use exact keys)
    name = "job-canonizer"
    main = "src/index.ts"
    compatibility_date = "2025-09-16"
    [vars]
    API_ORIGIN = "https://api.example.com"

-------------------------------------------------------------------------------

## 5) Backend — ASP.NET Web API (services, lifetimes, logging)

Controllers:
- ExtractController
  - POST /extract (see contract §1)

Services (DI, scoped unless noted):
- IUrlNormalizer (stateless)
- IPageFetcher (Playwright singleton; launches browser once; context-per-request)
- ISourceDetector
- IJobExtractor (strategy: picks proper extractor based on source)
- ICanonicalFinder
- IContactFinder
- IJsonLdParser
- ILlmClassifier (Semantic Kernel)
- IExcelRowFormatter
- IKeywordsFormatter
- IAuditStore (persist HTML/PNG/JSON)
- IClock (timezone-aware)
- ILogger (structured logs with requestId)

Timeouts:
- Overall request budget: 25s default. If exceeded, return 504 FETCH_FAILED with hint "paste contents via htmlFallback".

-------------------------------------------------------------------------------

## 6) Playwright.NET — Browser Policy

Launch (singleton):
- Headless: PLAYWRIGHT_HEADLESS
- Chromium with:
  - args: ["--disable-blink-features=AutomationControlled"]
  - userAgent: USER_AGENT_OVERRIDE (if set)
  - viewport: 1440x900
- Navigation timeout: PLAYWRIGHT_NAV_TIMEOUT_MS (default 30s)

Per request:
- New browser context; persistent cookies off.
- page.goto(url, waitUntil: "domcontentloaded")
- waitForLoadState("networkidle") with cap 2.5s
- Random polite delay: PLAYWRIGHT_POLITE_DELAY_MS uniform window
- Save HTML innerText snapshot and outerHTML
- Save PNG screenshot (fullPage) to SCREENSHOT_DIR

Anti-bot guidance:
- Do not solve captchas. If blocked, return PARSE_FAILED with advisory to use htmlFallback.

-------------------------------------------------------------------------------

## 7) URL Normalization & Source Detection

Normalization:
- Remove utm_* and tracking params; keep path/query needed by ATS.
- Collapse multiple slashes; strip trailing slash unless meaningful (e.g., Workday query).

SourceDetector rules (host contains):
- indeed.com → source=indeed
- greenhouse.io / boards.greenhouse.io → source=greenhouse
- lever.co / jobs.lever.co → source=lever
- workdayjobs.com / myworkdayjobs.com → source=workday
- else if company TLD and path includes "/careers", "/jobs/" → source=company
- default → unknown (still attempt generic extraction)

-------------------------------------------------------------------------------

## 8) Extraction — Field Heuristics (No-Guess)

Primary data sources (in order of trust):
1) JSON-LD schema.org JobPosting on page
2) Visible DOM (labels near title/company/salary)
3) Meta tags (`og:title`, `twitter:title`) if consistent with DOM

8.1 JSON-LD JobPosting Parser
- If script[type="application/ld+json"] includes JobPosting:
  - title → jobTitle
  - hiringOrganization.name → companyName
  - jobLocationType: "TELECOMMUTE" → workMode=Remote
  - jobLocation.address → city/state; ignore if not needed
  - baseSalary.value.minValue/maxValue with currency "USD" → salaryRange
- If any required field conflicts with visible H1/company label, prefer visible DOM.

8.2 IndeedExtractor (selector candidates; robust to change)
- Title candidates (pick longest non-empty text):
  - h1[data-testid*="jobsearch-JobInfoHeader-title"]
  - h1:has-text("job") near header
- Company name candidates:
  - div[data-company-name] a, div:has(span:has-text("Company")) a
  - Link near title containing the company domain
- Salary:
  - Any text node containing "$" and "-" within job details panel; parse as annual if "year"/"yr"/"annual"
- Work mode:
  - Tokens: "Remote", "Hybrid", "On-site", "Onsite" (case-insensitive)
- Level:
  - Tokens: "Senior", "Staff", "Principal", "Lead", "L5", "L6", "IC", "Manager" (but avoid "Hiring Manager")
- Description:
  - main job content container; strip nav/ads

8.3 Generic Company/ATS Extractor
- Title:
  - First H1 within main/section/article; fallback to `meta[property="og:title"]`
- Company:
  - If domain ≠ aggregator and contains company branding in header/footer; else take hiringOrganization from JSON-LD
- Salary:
  - Regex for $min–$max with optional commas; prefer annual; ignore hourly unless explicitly "hour"
- Work mode:
  - "Remote", "Hybrid", "On-site|Onsite"; if multiple present, choose the one in the summary header; else n/a
- Level:
  - Strict: accept only if token appears alongside title or in "About the role" heading

8.4 Strict Contact Discovery (canonical domain only)
- Only search pages on the canonicalUrl’s registrable domain (company or ATS).
- On the JD page and at most 2 linked pages under same domain (Contact, Team, Recruiting):
  - Hiring manager pattern: "(Hiring|Engineering|Product|Data) Manager|Director|Lead|Head of .*" adjacent to a person name (First Last)
  - Recruiter pattern: "Recruiter|Talent Acquisition|TA" + person name
  - Emails: mailto links or visible RFC 5322 emails; NO pattern guesses
  - Phones: +1-###-###-####; normalize various US formats to that exact form
- If not found, set hiringManager/email/phone to `n/a`.

-------------------------------------------------------------------------------

## 9) Canonical Company JD Finder (Algorithm)

Given initial page P0:
1) If P0 domain is company/ATS → canonicalUrl = P0 (unless content indicates aggregator embed)
2) Otherwise, attempt:
   - Follow visible "Apply", "Apply on company site", "Careers", or company-domain anchor with rel* hints
   - If target matches known ATS patterns (Greenhouse/Lever/Workday), accept
   - If target is company domain and includes "job" or "careers" segments, accept
3) If multiple candidates:
   - Prefer ATS deep link with matching title string similarity ≥ 0.75 to P0 title
   - Else prefer same-domain page with H1 containing all title tokens
4) BFS depth ≤ 2; never leave the company registrable domain once entered
5) If none found in budget, canonicalUrl = null

-------------------------------------------------------------------------------

## 10) LLM (Semantic Kernel) — Prompts & Guardrails

Model settings:
- temperature=0.1, top_p=0.9, max_tokens=1200
- JSON deterministic format; reject if keys missing

Inputs:
- visibleTitle, visibleCompany, visibleBody (plain text)
- jsonld (if any), sourceType, canonicalUrl (if any)

Tasks:
A) Work mode classification
  - Return "Remote"|"Hybrid"|"On-site" or "n/a" ONLY if an explicit literal is present in text. Quote the sentence snippet used for evidence.
B) Job level classification
  - Return a short token like "Senior","Staff","Principal","Lead","L5","Manager" or "n/a"; ONLY if explicit.
C) Keywords extraction
  - Return 10–25 high-signal terms present verbatim or as clear synonyms (e.g., "ASP.NET Core" ~ "ASP.NET"). Do not invent.

Output JSON (example):
    { "workMode":"Remote|Hybrid|On-site|n/a",
      "jobLevel":"Senior|Staff|…|n/a",
      "keywords":["...", "...", "..."],
      "evidence":{"workMode":"<short quote>", "jobLevel":"<short quote>"} }

If evidence is absent → set field to "n/a".

-------------------------------------------------------------------------------

## 11) Formatting Rules — Excel Row & Keywords

Sanitization (before composing):
- Collapse internal whitespace to single spaces.
- Remove tabs/newlines from every field (they break the single line).
- Dates: format as MM/DD/YYYY in timezone.
- Salary: "$168,000 – $240,000" → "168000–240000"
- hiringManager: "First Last — Title" (em dash) if both are present; else `n/a`

Composer (EXACT order from §1.2):
- Join 20 fields with real tab chars (U+0009).
- Never emit headers. Always emit 19 tabs (to ensure 20 columns).
- Example (conceptual; not literal tabs here): company[TAB]title[TAB]url[…19 tabs total…]

Keywords block:
- Join array with '\n' (LF). No trailing newline at end.

-------------------------------------------------------------------------------

## 12) Database Schema (PostgreSQL DDL)

Tables (minimum viable, single-owner)

    -- Raw page artifacts for audit
    CREATE TABLE audit_artifact (
      id UUID PRIMARY KEY,
      created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
      source_url TEXT NOT NULL,
      canonical_url TEXT,
      html_path TEXT NOT NULL,
      screenshot_path TEXT NOT NULL,
      detector TEXT NOT NULL
    );

    -- Parsed facts (normalized), one row per extract
    CREATE TABLE extraction_result (
      id UUID PRIMARY KEY,
      artifact_id UUID NOT NULL REFERENCES audit_artifact(id) ON DELETE CASCADE,
      company_name TEXT NOT NULL,
      job_title TEXT NOT NULL,
      job_source_url TEXT NOT NULL,
      hiring_manager TEXT,
      phone TEXT,
      email TEXT,
      stage TEXT NOT NULL, -- Seen|Applied|...
      next_action_date DATE NOT NULL,
      next_action_note TEXT NOT NULL,
      last_updated DATE NOT NULL,
      interview_direction TEXT,
      interview_tags TEXT,
      salary_range INT8RANGE, -- use int8 to be safe, or int4range if preferred
      work_mode TEXT,
      job_level TEXT,
      application_files TEXT,
      version INT NOT NULL,
      created_at DATE NOT NULL,
      company_career_url TEXT,
      data_confidence TEXT NOT NULL  -- High|Medium|Low
    );

Indexes:
- CREATE INDEX ON audit_artifact (created_at DESC);
- CREATE INDEX ON extraction_result (company_name, job_title);
- CREATE INDEX ON extraction_result USING GIST (salary_range);

-------------------------------------------------------------------------------

## 13) Frontend — Next.js 15 UI Spec

Screen: “Single URL Canonicalizer”
- Components:
  - URL input (required), timezone select (default America/Los_Angeles), submit button
  - Result panel:
    - Code-style box with single-line tab-delimited row
    - Second box with vertical keywords
    - Copy buttons (use Clipboard API)
    - Audit link: /audit/{auditId}
- Error states:
  - If FETCH_FAILED → show “Paste visible contents” textarea; POST as htmlFallback
  - If PARSE_FAILED → show inspector with screenshot/HTML links

Accessibility:
- Disable submit while in-flight; show progress; announce errors via ARIA live region.

-------------------------------------------------------------------------------

## 14) Audit Viewer — /audit/[id]

Displays:
- Source URL, Canonical URL, Detector, CreatedAt
- Screenshot image (fit to width)
- Download/View raw HTML

Security (personal mode):
- No auth; hidden route (not indexed). Optionally require shared secret header when deployed publicly.

-------------------------------------------------------------------------------

## 15) Error Handling & Edge Cases

- Unsupported/blocked pages:
  - Return FETCH_FAILED with guidance to paste htmlFallback
- Multiple candidate canonicals:
  - Choose highest title similarity; otherwise prefer ATS domain; else set canonicalUrl=null (Medium confidence)
- Salary strings with hourly rates:
  - If "hour" or "/hr" present → ignore (set n/a) for v0.1 to avoid mis-scaling
- International currency:
  - If not USD $ explicitly → set salaryRange n/a

-------------------------------------------------------------------------------

## 16) Idempotency, Concurrency, Performance

- Single-user, per-request idempotency via send-hash not required yet.
- Limit to 1 browser context at a time initially; configurable later.
- Enforce 25s request budget; advise fallback paste if exceeded.

-------------------------------------------------------------------------------

## 17) Test Plan — Unit / Integration / E2E (concrete)

Unit
- UrlNormalizer: strips UTM, preserves ATS query
- SalaryParser: "$168,000 – $240,000" → 168000–240000; hourly ignored
- WorkModeClassifier: explicit literals only; ambiguous → n/a
- LevelClassifier: "Senior", "Staff", "L5" detected only when explicit
- Phone/Email normalizers: canonical forms, RFC 5322 compliance check

Integration
- IndeedExtractor on 3 recorded fixtures (stored HTML):
  - Title/company populated; salary present only when explicit; work mode from header chips
- JsonLd parser on 3 ATS fixtures (Greenhouse/Lever/Workday)
- CanonicalFinder resolves from aggregator to ATS/company in ≥ 4/5 cases

E2E
- POST /extract with a valid Indeed URL:
  - Produces a 20-field tab-delimited single line (verify exactly 19 tabs)
  - nextActionDate = today+7 in TZ; lastUpdated/createdAt = today
  - keywords 10–25 items; no invented tech
- Blocked page path:
  - Returns FETCH_FAILED; retry with htmlFallback produces valid outputs

Manual checklist
- Paste 5 diverse URLs (aggregator and ATS)
- Copy row to Excel: each field lands in correct column; no wrap
- Open audit viewer: screenshot and HTML match extracted values

-------------------------------------------------------------------------------

## 18) Implementation Cards (buildable mini-scopes)

[API-001] Wire POST /extract
- Build: Controller, model validation, correlationId propagation
- Test: 400 BAD_URL on malformed URL; 200 baseline

[CRAWL-001] Playwright bootstrap
- Build: Singleton browser; per-request context; HTML/PNG save
- Test: Known page fetch; files exist

[DETECT-001] SourceDetector
- Build: hostname/regex mapping (indeed|greenhouse|lever|workday|company)
- Test: Table-driven tests

[PARSE-001] JsonLdJobPostingParser
- Build: schema.org JobPosting parse with safe fallbacks
- Test: Fixtures cover title/company/salary

[PARSE-002] IndeedExtractor
- Build: Title/company/salary/mode/level via resilient selector candidates
- Test: 3 fixtures pass; where absent → n/a

[CANON-001] CanonicalFinder
- Build: Follow "Apply"/"Careers" anchors; ATS patterns; title similarity
- Test: 5 cases; ≥4 resolved

[CONTACT-001] ContactFinder (strict)
- Build: Same-domain search; name+title; mailto; +1 phone normalization
- Test: Positive/negative fixtures

[LLM-001] SK Classifier
- Build: Prompts+schema; temp=0.1; evidence gates
- Test: Ambiguity → n/a; explicit → correct

[FORMAT-001] Row/Keywords Composer
- Build: Sanitizers; 20-field join; keywords join
- Test: Excel paste sanity (19 tabs)

[UI-001] Minimal UI
- Build: Form; results; copy buttons; audit link
- Test: 3 URL runs

[AUDIT-001] Audit Viewer
- Build: GET /audit/{id}; show html/screenshot
- Test: Visual check

-------------------------------------------------------------------------------

## 19) Runbook — Local Dev & Deploy

Local
- Prereqs: Node 20+, .NET 8+, PostgreSQL 14+, pnpm, Playwright browsers installed
- Steps:
  1) Create DB and set DB_CONN
  2) Install Playwright deps: run playwright install
  3) Start API: dotnet run -p apps/api
  4) Start FE: pnpm dev --filter @apps/frontend
  5) Optional Workers: wrangler dev (proxy to API_ORIGIN)

Staging Deploy
- Set API_ORIGIN in Workers vars
- Publish Workers: wrangler deploy
- Publish API (container or VM)
- Verify /api/health, then /extract

-------------------------------------------------------------------------------

## 20) Acceptance (Definition of Done)

- From a clean deploy, pasting a valid Indeed URL returns:
  - Single-line row (20 columns, unknowns as `n/a`, dates MM/DD/YYYY with nextActionDate=today+7)
  - Keywords block (10–25 lines)
  - auditId resolvable; screenshot/HTML align with extracted values
- No invented data. Salary only when explicit $min–$max. Contacts only when explicitly present on official domain.
- Canonical finder prefers company/ATS and overrides aggregator fields when found.

-------------------------------------------------------------------------------

## 21) Roadmap (next increments, tiny)

- Add Source B extractor (your most common ATS)
- Add batch queue (Hangfire) for paste-many
- Add CSV export of rows
- Add Google adapters (Gmail/Sheets/Calendar) behind flags later

-------------------------------------------------------------------------------

## 22) Operator UX Notes (Personal Mode)
- If network blocks a page, use the “Paste visible contents” fallback. The system will parse the pasted HTML/text with the same rules and still return the two artifacts.
- Always review the audit screenshot before trusting salary/contact fields.


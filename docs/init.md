# REQUIREMENTS v0.1 — Single-URL Job Canonicalizer (Personal, No Login, Gemini-2.5-flash, Localhost-Testable)

Purpose: When you paste an aggregator job URL (e.g., Indeed), the system must 1) read that exact HTML, 2) have the LLM extract core facts (company, title, salary, mode, level, etc.) strictly from the page, 3) locate the official company career site for that company, 4) find the same role on the company site (canonical JD), 5) re-extract/verify against the canonical page, then 6) output one Excel-ready single-line, tab-delimited row (20 columns) plus a vertical keyword list. No email/Sheets/Calendar yet. No guessing—values appear only if explicitly present.

------------------------------------------------------------------------------------------------------------------------------------

0) Scope and “Done”
- In-scope: Single URL in, two artifacts out; aggregator → company canonicalization; strict evidence; audit trail of HTML and PNG; localhost operation.
- Out-of-scope v0.1: Login, bulk queues UI, messaging/email, Google integrations.
- Done when:
  A) From a fresh localhost setup, POST /extract with a reachable Indeed URL returns:
     • one single line with exactly 20 tab-separated fields, unknowns as n/a, dates MM/DD/YYYY with nextActionDate = today+7 (America/Los_Angeles by default)
     • a vertical keyword list (10–25 items, each on its own line, no trailing line)
     • source.detected and source.canonicalUrl populated; auditId opens screenshot and HTML
  B) If a canonical company JD exists, canonical values override aggregator values.
  C) Salary/contact/mode/level only appear if literally present; otherwise n/a.

------------------------------------------------------------------------------------------------------------------------------------

1) Localhost Topology, Stack, and Layout
- Stack: Next.js 15 (FE), ASP.NET Web API (.NET 8) (BE), Playwright.NET (crawler), Gemini-2.5-flash (LLM via Gemini API), PostgreSQL (storage), Cloudflare Workers (dev proxy optional), Hangfire off.
- Ports: API http://localhost:5050, FE http://localhost:3000, Workers proxy http://localhost:8787 → API, Postgres localhost:5432.
- Monorepo layout:
  • apps/frontend — minimal UI (URL input, results, audit link)
  • apps/api — controllers/services, Playwright bootstrap, Gemini client, DB access
  • packages/domain — DTOs, validation, confidence rules
  • packages/extractors — AggregatorExtractor (Indeed v1), JsonLdParser, CanonicalFinder, CompanyExtractor (ATS/company), ContactFinder, TextSanitizers
  • packages/formatters — RowComposer (20 fields), KeywordsComposer, DateFmt
  • packages/edge — Workers proxy handler and config
  • infra — env catalog, runbooks, migrations description
  • docs — this spec, prompts schema, fixtures guide

------------------------------------------------------------------------------------------------------------------------------------

2) Environment and Secrets (names only; no code snippets)
- DB_CONN, TZ_DEFAULT=America/Los_Angeles, PLAYWRIGHT_HEADLESS=true, PLAYWRIGHT_NAV_TIMEOUT_MS=30000, PLAYWRIGHT_POLITE_DELAY_MS_MIN=800, PLAYWRIGHT_POLITE_DELAY_MS_MAX=1600, USER_AGENT_OVERRIDE, GEMINI_API_KEY, GEMINI_MODEL_ID=gemini-2.5-flash, FEATURE_USE_HANGFIRE=false, SCREENSHOT_DIR=./audit/png, HTML_DIR=./audit/html, API_ORIGIN=http://localhost:5050, ROUTE_RATE_LIMIT=30/min, FETCH_MAX_REDIRECTS=5, REQUEST_BUDGET_MS=25000.

------------------------------------------------------------------------------------------------------------------------------------

3) Data Contract (entities; no SQL)
- AuditArtifact: id, createdAt, sourceUrl, canonicalUrl (nullable), htmlPath, screenshotPath, detector (free-text tag for logging; e.g., indeed/ats/company).
- ExtractionResult: id, artifactId, companyName, jobTitle, jobSourceUrl, hiringManager, phone, email, stage, nextActionDate, nextActionNote, lastUpdated, interviewDirection, interviewTags, salaryRange (min–max int), workMode, jobLevel, applicationFiles, version, createdAt, companyCareerUrl, dataConfidence.
- Note: salaryRange only when the page explicitly shows a USD range with a min and a max; otherwise n/a. workMode and jobLevel only when explicit tokens exist; otherwise n/a.

------------------------------------------------------------------------------------------------------------------------------------

4) Exact Outputs (format rules)
- Excel row (20 fields, exact order; always emit 19 tabs):
  1 companyName
  2 jobTitle
  3 jobSourceUrl
  4 hiringManager (“FullName — Title” only if both explicit; else n/a)
  5 phone (+1-###-###-#### only if explicit; else n/a)
  6 email (RFC 5322 only if explicit; else n/a)
  7 stage (default Seen)
  8 nextActionDate (MM/DD/YYYY; today+7 in timezone)
  9 nextActionNote (default “Research company page & verify JD match” unless the operator sets otherwise)
  10 lastUpdated (MM/DD/YYYY; today)
  11 interviewDirection (n/a v0.1)
  12 interviewTags ({…} or n/a)
  13 salaryRange (integer “min–max” only if explicit USD range on page; else n/a)
  14 workMode (On-site | Hybrid | Remote only if explicit; else n/a)
  15 jobLevel (Senior | Staff | Principal | Lead | L5 | Manager only if explicit; else n/a)
  16 applicationFiles (n/a v0.1)
  17 version (“1”)
  18 createdAt (MM/DD/YYYY; today)
  19 companyCareerUrl (canonical company/ATS JD URL if found; else n/a)
  20 dataConfidence (High | Medium | Low per rules below)
- Sanitization: remove tabs/newlines from every field; collapse multi-spaces; guarantee exactly 20 fields.
- Keywords block: 10–25 items; one per line; derived from JD text only; no trailing newline; no invented tech.

------------------------------------------------------------------------------------------------------------------------------------

5) End-to-End Flow (this is the core correction to align with your intent)
The system must behave like this when given, for example, https://www.indeed.com/viewjob?jk=05d232f064d525f1&tk=... :
A. Read the aggregator HTML (Indeed) exactly as rendered (Playwright fetch). Save outer HTML and full-page PNG for audit.
B. Feed the HTML (and visible text extracted) to the LLM (Gemini-2.5-flash) to extract: companyName, jobTitle, salary evidence (if any), explicit mode/level tokens, and JD body for keywords. Do not invent values; mark n/a if no literal evidence.
C. With companyName from B, locate the official company domain (preferably from the page content; otherwise from known branding cues within the HTML). Examples: “Anduril Industries” → anduril.com. Record the chosen domain.
D. Visit the company root domain. Discover the Careers area: try common paths (/careers, /jobs, /careers/jobs, /join, navigation link labeled “Careers”, “Jobs”, “Join Us”, “Open Roles”). Limit traversal to BFS depth 2 on the same registrable domain.
E. Inside Careers or ATS pages under the company’s control (e.g., Greenhouse/Lever/Workday subpaths or subdomains linked from the company site), search for the exact role:
   • Primary match on job title token similarity ≥ 0.75 compared to the aggregator title
   • Secondary match on location tokens if present
   • Tertiary hints: internal job ID, apply button presence, posting recency
F. When an exact match is found, treat that page as the canonical JD (companyCareerUrl). Fetch HTML/PNG; re-extract using LLM (same no-guess rules); for conflicting fields, canonical overrides aggregator.
G. Strict contacts discovery (canonical domain only): scan the canonical JD and up to two linked pages on the same domain (Contact/Team/Recruiting) for explicit person name + title, official recruiting email (mailto or visible RFC 5322), and US phone. Normalize phone to +1-###-###-####. If not explicit, set n/a.
H. Compute nextActionDate = today+7 (timezone), lastUpdated and createdAt = today (MM/DD/YYYY).
I. Decide dataConfidence:
   • High when canonical found on company/ATS, title/company match, fields sourced from canonical
   • Medium when canonical not found but aggregator extraction is consistent and unambiguous
   • Low when ambiguities remain or page content insufficient
J. Compose the 20-field single-line row with real tabs; compose keywords list; persist artifacts with auditId; return results.

Important clarifications
- The “source.detected” tag is only for logging/telemetry; the system always starts from the given aggregator page and then does the canonical hunt as described (your exact requirement).
- LLM is the primary extractor from both the aggregator HTML and the canonical HTML; rule-based parsing is supportive (e.g., for salary digits, phone normalization), not authoritative.

------------------------------------------------------------------------------------------------------------------------------------

6) Gemini-2.5-flash Usage (what it must output; how to validate)
- Input to LLM: visibleTitle string (if detectable), visibleCompany string, stripped visibleBody (from HTML), any JSON-LD content as text, and hints (e.g., “We are on the aggregator page” vs “We are on the company page”).
- LLM must return JSON fields:
  • companyName (string; only if explicitly stated; else n/a)
  • jobTitle (string; literal JD title)
  • workMode (Remote|Hybrid|On-site|n/a) with a short evidence quote when not n/a
  • jobLevel (Senior|Staff|Principal|Lead|L5|Manager|n/a) with evidence when not n/a
  • salary (raw textual snippet(s) that show explicit USD min–max; else empty)
  • keywords (array of 10–25 terms; must appear in page text or are clear synonyms)
- Client-side validation:
  • If evidence is absent for workMode/jobLevel, set to n/a.
  • Parse salary only if a USD range with min and max is detectable; convert to integers (strip $, commas); otherwise n/a.
  • Remove tabs/newlines from all returned strings.
- Time budget: The Gemini call must complete within 15s (overall budget 25s). If it cannot, short-circuit with FETCH_FAILED and request htmlFallback.

------------------------------------------------------------------------------------------------------------------------------------

7) Canonical Finder (what to build; how it decides)
- Input: sourceUrl (aggregator), extracted companyName/title from LLM, initial page HTML.
- Company domain discovery: prefer explicit links or domain mentions in the aggregator HTML; otherwise derive by brand string matching against link hosts on the page; if multiple candidates, prefer .com/.ai with the brand token.
- Careers discovery on company domain: try direct paths (/careers, /jobs, /join), then page nav anchors (“Careers”, “Jobs”, “Join Us”, “Open Roles”); limit traversal to 2 clicks from root.
- ATS handling: If careers link points to an ATS (Greenhouse/Lever/Workday) under company control (linked from company site), accept as canonical domain context.
- Role match: use title token similarity (≥ 0.75 threshold). If several matches exist, prefer page with an “Apply” button and recent posting markers. If still tied, pick the one whose H1 equals or contains all major tokens from the aggregator title.
- Failure mode: If no match within traversal budget, canonicalUrl = n/a and confidence cannot be High.

------------------------------------------------------------------------------------------------------------------------------------

8) Strict Contact Discovery (what counts; what not)
- Allowed sources: only pages on the canonical registrable domain (the JD page plus up to two linked pages like Contact, Team, Recruiting).
- Acceptable hiringManager: a person full name adjacent to a role title clearly relevant (e.g., “Director of Manufacturing Test”).
- Acceptable email: explicit on page or mailto, RFC 5322; no pattern guesses.
- Acceptable phone: US number normalized to +1-###-###-####; reject if non-standard or non-US in v0.1.
- If any field is not clearly explicit, set n/a.

------------------------------------------------------------------------------------------------------------------------------------

9) API Contract and Error Modes (no code)
- POST /extract with JSON body { url, htmlFallback?, timezone? }.
- Successful 200 returns { rowTabDelimited, keywordsBlock, auditId, source: { detected, canonicalUrl } }.
- Error returns { error, code } where code ∈ BAD_URL, FETCH_FAILED, UNSUPPORTED, PARSE_FAILED, TIMEOUT.
- Guidance for FETCH_FAILED: Prompt user to paste visible page content into htmlFallback and retry, keeping the same rules.

------------------------------------------------------------------------------------------------------------------------------------

10) UI Requirements (frontend)
- One page with a URL input, Submit, and two results boxes (copy for row; copy for keywords). A link “Open audit” points to /audit/{auditId}.
- Error UX: If blocked or timed out, show a textarea to paste visible HTML/text and retry with htmlFallback.
- Accessibility: Disable submit in flight; announce errors via ARIA live region; use monospace boxes for copy.

------------------------------------------------------------------------------------------------------------------------------------

11) Workers Dev Proxy (optional but recommended)
- /api/* proxied to API_ORIGIN with X-Request-Id injection (uuid) and rate limiting using ROUTE_RATE_LIMIT.
- Pass through 429/5xx; enforce FETCH_MAX_REDIRECTS.

------------------------------------------------------------------------------------------------------------------------------------

12) Testing — What to Build and How to Verify (no code)
Unit (pure utility)
- Salary normalization: “$168,000–$240,000” → 168000–240000; hourly or non-USD ignored (n/a).
- Date formatting: given timezone and “today”, produce MM/DD/YYYY; nextActionDate = +7 days.
- Sanitizers: remove tabs/newlines; guarantee 20 fields with 19 tabs.
- Title similarity: token Jaccard or cosine; ensure ≥ 0.75 in positive cases; < 0.75 in negatives.

Integration (fixtures; saved HTML)
- Aggregator extraction: three Indeed fixtures (salary present, salary absent, explicit Remote) produce correct fields or n/a; LLM returns mode/level only with evidence lines.
- Canonical discovery: aggregator fixture links lead to company careers/ATS fixture; canonicalUrl chosen; canonical overrides aggregator title/company; confidence = High.
- Contacts: positive fixture (JD links to Recruiting page listing a named recruiter with title and email) populates all; negative fixture yields n/a.

E2E (live or replayed through Playwright)
- Paste a reachable Indeed URL:
  • rowTabDelimited has exactly 20 fields; unknowns n/a; dates per MM/DD/YYYY; nextActionDate = today+7
  • keywordsBlock 10–25 items; each on its own line; no inventions
  • auditId opens screenshot and HTML that match the extracted values
- Blocked page: returns FETCH_FAILED; on re-try with htmlFallback, outputs are produced without guessing.
- No canonical found: confidence = Medium; companyCareerUrl = n/a.

Operational checks
- Time budget: long pages cause TIMEOUT; no partial guessing.
- Proxy rate-limit can be triggered and returns a clear message.

------------------------------------------------------------------------------------------------------------------------------------

13) Implementation Cards (small, dependency-ordered; each contains “what to build” and “how to test”)
ID API-001 — POST /extract scaffold
- Build: Validate payload; start correlation ID; enforce REQUEST_BUDGET_MS.
- Test: BAD_URL for malformed; 200 stub for valid.

ID CRAWL-001 — Playwright bootstrap and capture
- Build: Singleton browser; per-request context; UA override; save HTML to HTML_DIR and PNG to SCREENSHOT_DIR; create AuditArtifact with detector placeholder “source”.
- Test: Known public page captured; files exist; artifact recorded.

ID LLM-001 — Gemini client (JSON-style schema)
- Build: Client that calls gemini-2.5-flash with responseMimeType application/json; returns companyName, jobTitle, workMode, jobLevel, salary raw snippets, keywords, evidence quotes; client-side validation and sanitization.
- Test: Inject controlled HTML text; confirm explicit tokens get classified; no evidence → “n/a”; keywords 10–25.

ID PARSE-AGG-001 — Aggregator extraction (LLM-first)
- Build: From the fetched aggregator HTML, produce preliminary fields using LLM; also extract visible title/company text to feed into LLM; normalize salary only if explicit USD min-max.
- Test: Three Indeed fixtures; values match page; no salary when not explicit.

ID CANON-001 — Company domain locator
- Build: From aggregator HTML + LLM companyName, identify the official domain (prefer links within the page; else brand token matching among link hosts).
- Test: Given fixtures that contain company homepage links, resolve correct domain; ambiguous cases pick brand-matching .com/.ai.

ID CANON-002 — Careers discovery on company domain
- Build: Visit domain root; attempt /careers, /jobs, /join and nav anchors (“Careers”, “Jobs”, “Join Us”, “Open Roles”); BFS depth ≤ 2; remain on registrable domain or ATS linked from it.
- Test: Company fixtures with a Careers nav item resolve to careers page; absence produces n/a with Medium confidence later.

ID CANON-003 — Role match finder
- Build: On careers/ATS pages, compute title token similarity to aggregator title; threshold ≥ 0.75; prefer pages with “Apply” buttons; choose best candidate.
- Test: Positive fixtures select correct JD; negative fixtures yield no match.

ID PARSE-CANON-001 — Canonical extraction (LLM-first)
- Build: Fetch canonical page, capture artifacts, send to LLM; override aggregator fields where canonical provides explicit values.
- Test: Fixture ensures canonical overrides aggregator for company/title; salary accepted only if explicit.

ID CONTACT-001 — Strict contact discovery
- Build: On canonical JD and up to two linked same-domain pages, search explicit name+title, recruiting email (mailto/visible), and US phone; normalize phone; else n/a.
- Test: Positive and negative fixtures behave accordingly.

ID CONF-001 — Confidence calculator
- Build: Decide High/Medium/Low per Section 5.I; store companyCareerUrl accordingly.
- Test: Canonical found → High; not found → Medium; conflicting signals → Low.

ID FORMAT-001 — Row and keywords composition
- Build: Compute dates; produce 20-field tab-delimited line with 19 tabs; ensure field sanitization; produce keyword list lines.
- Test: Paste into Excel aligns columns; dates recognized; exact tab count verified.

ID API-002 — Wire the pipeline
- Build: A→I pipeline: fetch aggregator → LLM extract → company domain → careers → match → canonical fetch → LLM re-extract → contacts → confidence → compose → persist → respond.
- Test: Full E2E on three URLs; artifacts and outputs match expectations.

ID UI-001 — Minimal frontend
- Build: URL input, submit, results boxes, copy buttons, audit link; fallback textarea on FETCH_FAILED.
- Test: Manual run: copy-paste row to Excel; keywords list looks correct; audit opens.

ID AUDIT-001 — Audit viewer route
- Build: Display screenshot and link to raw HTML for auditId; show sourceUrl/canonicalUrl/detector/createdAt.
- Test: Open recent audit; verify visuals match extracted values.

ID SAFE-001 — Workers proxy and rate-limit
- Build: Proxy /api/* to API_ORIGIN; inject X-Request-Id; apply ROUTE_RATE_LIMIT; enforce FETCH_MAX_REDIRECTS.
- Test: Health via proxy; burst to trigger rate-limit; confirm error message.

------------------------------------------------------------------------------------------------------------------------------------

14) Operator Runbook (local)
- Prereqs: Node 20+, .NET 8 SDK, PostgreSQL 14+, Playwright browsers installed, GEMINI_API_KEY.
- Steps: Set env vars; start API at :5050; start FE at :3000; optional Workers at :8787.
- Verify: Health 200; POST /extract with a reachable Indeed URL → get row+keywords and auditId; paste row into Excel; open audit viewer; confirm accuracy; try a blocked page then retry with htmlFallback.

------------------------------------------------------------------------------------------------------------------------------------

15) Guardrails
- Never guess contacts, salary, work mode, or level. Only populate when explicit on page; otherwise n/a.
- Canonical page must come from the company’s registrable domain or ATS linked from that domain; do not “search the web” arbitrarily in v0.1.
- LLM outputs must include short evidence quotes for workMode and jobLevel when not n/a; lack of evidence → n/a.
- Salary must have both min and max in USD; otherwise n/a.
- Always produce exactly 20 fields in the row; unknowns as n/a; dates in MM/DD/YYYY.


# Changelog

All notable changes to the n8n Job Search Automation workflow are documented here.

---

## [v9.11] — 2026-05-09

### Changed
- **Tightened Filter Stack & Quality based on execution #39 analysis** — analysis of the 28-job run showed 93/141 (66%) of OpenAI-scored jobs scored <50, wasting tokens and adding ~7 minutes to runtime. Root cause: title pattern was too narrow.
  - **Broader stack patterns**: `\bjava\b`, `\bpython\b`, `\bnode\.?js\b`, `\bgolang\b`, `\b(c\+\+|cpp)\b`, `\bkernel\b` — match anywhere in title (catches "Senior Software Engineer - Java", "Java API Engineer", "Lead Software Engineer - C++")
  - **Frontend/mobile additions**: `angular`, `vue`, `laravel`, `react native` (already had), `mobile`
  - **Niche/product platforms**: `salesforce`, `sap`, `abap`, `temenos`, `workday`, `labware`, `genesys`, `servicenow`, `aveva`, `nugenesis`, `fenergo`, `sprinklr`
  - **Role-type filter**: test/QA/SDET, DevOps/SRE/CI-CD, Cloud Engineer, Network Engineer, Security Engineer, Support roles, Project/Program Manager, Business Analyst — none of these are .NET backend roles
  - **Junior pattern hardened**: `\bsoftware\s*engineer\s*i\b`, `\bsde[-\s]?[1i]\b`, `\bintern\b` — but yields to `.NET-in-title` so "Lead Software Engineer I (C#.NET)" still passes
  - **Senior pattern broadened**: added `\bsr\.?\s*manager\b`, `\bdirector,?\s*software\b`, `\bsde\s*iii\b`/`\bsde\s*3\b`
  - **Description-level non-.NET check**: catches "Java is required", "Python is the primary stack" when title is generic
- **Expected impact** based on simulating new filter against execution #39's 193 input jobs:
  - Pre-filter: 193 → 94 (was 141) — **47 fewer OpenAI scoring calls**
  - Saves ~2.3 minutes runtime, ~$0.013 OpenAI tokens per run
  - 4 high-score (>=70) drops are scoring hallucinations (pure Java/Python jobs OpenAI inflated to 70-90), not real matches

---

## [v9.10] — 2026-05-09

### Fixed
- **"Nothing to repeat" crash in Filter Stack & Quality** — all `\b`, `\s`, `\d` regex escape sequences were stored as literal `b`/`s`/`d` in the live workflow due to backslashes being dropped during a previous JSON serialization step. This made `/(\d+)\s*\+/` become `/(d+)s*+/` at runtime — a quantifier-on-quantifier (`s*` followed by `+`) that V8 rejects with "Nothing to repeat". All other title/seniority patterns also silently matched wrong (e.g. `/\bjava\s*/` became `/bjava s*/` — missed "java" at word boundaries). Fixed by rewriting node code via PowerShell here-string to preserve backslashes, importing via n8n CLI, and re-exporting.

---

## [v9.9] — 2026-05-05

### Fixed
- **Mode regression in v9.8** — when applying the v9.8 fixes, the base JSON used was an outdated pre-v9.7 snapshot rather than a fresh API fetch. All 9 Code nodes that had been set to `runOnceForAllItems` in v9.7 reverted to `runOnceForEachItem` again. Re-applied all mode fixes by fetching fresh from the live n8n REST API and patching in-place. Root cause noted in dev guidelines: always `curl` fresh from API before any workflow edit.
- **minScore confirmed at 70** — v9.8 set minScore to 50 in docs, but user raised it to 70 via n8n UI. Updated Set Job Preferences node via REST API to persist `minScore: 70` in code (not just UI override), exported to JSON.

### Changed
- **Audit script** (`audit_workflow.js`) — wrote a comprehensive automated audit that checks all 28 nodes for: Code node mode vs `$input` usage pattern, node name references in code that don't exist, placeholder values, HTTP Authorization header expression syntax, Merge node input counts, Google Sheets credentials/documentId/matchingColumns, Gmail credentials/sendTo, unreachable nodes (no incoming connections), dead-end nodes (no outgoing connections), and specific logic checks for key cross-node references. Run with `node audit_workflow.js` after loading `wf_audit2.json` from live API.

---

## [v9.8] — 2026-05-05

### Fixed
- **39 zero-score jobs in Parse AI Score** — root cause: OpenAI occasionally wraps the JSON response in markdown code fences or adds preamble text, causing `JSON.parse` to throw and fall back to `{}` (score 0). Fix: added a regex extraction fallback `aiText.match(/\{[\s\S]*\}/)` that pulls the JSON block out of any surrounding text before parsing.
- **Mobile UI and Java+framework roles slipping through Filter Stack & Quality** — `SDE-II Mobile UI @ ZET` scored 100, `Senior Software Engineer (java, React) @ everbridge` scored 78. Both passed the title filter because their patterns weren't covered. Added 3 new title blacklist patterns: `/\bjava[\s,/|]+react\b/i`, `/\bjava[\s,/|]+spring\b/i`, `/\bmobile\s*(ui|developer|engineer)\b/i`, `/\bui\s*developer\b/i`.

### Changed
- **minScore lowered 75 → 50** — execution #28 analysis showed 23 jobs scored 65-74 (just below old cutoff) that are legitimate matches. At 50, borderline .NET roles surface instead of being silently dropped. Expected matches per run: 11 → 30-50.

---

## [v9.7] — 2026-05-05

### Fixed
- **Code node execution modes** — 9 nodes were missing explicit `mode` setting and defaulting to `runOnceForEachItem` despite their code using `$input.all()`. This would cause each node to run once per input item rather than processing all items together, breaking deduplication, aggregation, and filtering logic.
  - Set to `runOnceForAllItems`: `parseJobs`, `filterDuplicates`, `aggregateJobs`, `filterNewJobs`, `filterStackQuality`, `filterMatches`, `buildEmail`
  - Set to `runOnceForAllItems`: `setNaukriQueries`, `setNaukriQueriesCSharp` (use `$input.first()` — must run once only to avoid duplicate Apify calls)
- **Merge Search Results** — `mode` was set to empty `{}` instead of `append`. All 3 search branches (JSearch + Naukri .NET + Naukri C#) were not being combined correctly.
- **Save to Google Sheets** — `matchingColumns` was set to `["Date"]` (single incorrect column) and `value` mapping was empty `{}`. Fixed to match on `["Job Title", "Company", "Apply Link"]` and mapped all 10 data columns correctly.
- **Read Existing Jobs** — missing explicit `operation: "read"` parameter added.
- **Sync Dedup Inputs** — missing explicit `mode: "append"` parameter added.

### Changed
- **Workflow import method**: Added CLI-based import using `docker exec n8n n8n import:workflow` as the recommended approach — preserves the canonical workflow ID `8vs888j57KeLaGtH` and avoids manual node ID drift.
- **API keys**: Confirmed `keys.json` is correctly mounted at `/home/node/jobsearch-keys.json` inside Docker; `yourEmail` and `spreadsheetId` remain inline in Set Job Preferences.

---

## [v9.6] — 2026-04-12

### Changed
- **API keys moved to `keys.json` file** — all three API keys (`rapidApiKey`, `openAiApiKey`, `apifyToken`) are now read from a Docker-mounted file at runtime instead of being hardcoded in the Set Job Preferences node
  - File: `keys.json` in repo root → mounted read-only to `/home/node/jobsearch-keys.json` inside Docker
  - `N8N_RESTRICT_FILE_ACCESS_TO` in docker-compose.yml updated to include `/home/node/jobsearch-keys.json`
  - `Set Job Preferences` reads keys with `const keys = JSON.parse(fs.readFileSync('/home/node/jobsearch-keys.json', 'utf8'))` using Node.js `fs` (available via `NODE_FUNCTION_ALLOW_BUILTIN=fs`)
  - File read wrapped in try/catch — produces clear error message if mount is missing
  - `yourEmail` and `spreadsheetId` remain inline in the node (not sensitive enough to warrant file storage)

### Fixed
- **"Unexpected token '.'" in Set Job Preferences** — object spread `{ ...baseConfig, searchQuery }` caused a syntax error in n8n's Code node sandbox. Replaced with `Object.assign({}, baseConfig, { searchQuery })`. Also added `.trim()` on `rapidApiKey` to handle trailing spaces in the JSON file.
- **Success Summary always showing `matchCount: 0, topScore: 0`** — root cause: the Gmail node output does not forward metadata fields from its input. Fix: `Success Summary` now reads from `$('Build Email Digest').first().json` directly instead of `$input.first().json`. Confirmed working in execution #117: 25 matches, top score 100.

---

## [v9.5] — 2026-03-17

### Fixed
- Naukri (Apify) jobs silently dropped in Parse Job Results — only JSearch results were being scored and emailed

### Root Cause
The `parseJobs` Code node had two format checks:
1. `item.json.data && Array.isArray(item.json.data)` → catches JSearch (single response with `data[]` array) ✓
2. `Array.isArray(item.json)` → intended to catch Apify Naukri responses, but **never matched**

The assumption was that the Apify HTTP Request node would return the entire JSON array as a single item (`item.json = [job1, job2, ...]`). In practice, n8n HTTP Request v4.2 **unpacks** JSON array responses into individual items — each Naukri job becomes a separate item with `item.json = { title, company, apply_link, ... }` (a plain object, not an array). So `Array.isArray(item.json)` always returned false, and 20-40 Naukri jobs per run were silently skipped.

Execution #80 confirmed: Search Naukri (Apify) output 20 items, Search Naukri C# output 40 items, but zero naukri.com URLs appeared in Parse Job Results output.

### Fix
Added a third case to `parseJobs`:
```javascript
} else if (item.json.title || item.json.jobTitle || item.json.positionName || item.json.companyName) {
  // Individual Naukri/Apify job object — n8n HTTP Request unpacks JSON arrays into individual items
  allJobs.push(parseNaukriJob(item.json));
}
```
Also extracted a `parseNaukriJob(job)` helper to avoid code duplication between the array and individual-object cases.

### Impact
Naukri adds ~50-60 additional jobs per run (.NET Developer + C# Developer, 30 each). Previous runs processed only JSearch results (~140 jobs). Expected raw total now back to design spec: ~190-200 unique jobs/run.

---

## [v9.4] — 2026-03-16

### Fixed
- Scoring ceiling stuck at 85 across all runs

### Root Cause
GPT-4o-mini (and LLMs generally) anchor "good match" in the middle of any scale when given no arithmetic guidance. With a 0-100 range and vague "BOOST by 8" instructions, the model interprets "good match" as 75-85 and shifts scores "a bit higher" rather than doing literal arithmetic. The previous rubric also had the 85-100 band start at 85, meaning boosts were supposed to push scores *within* that band rather than above it — a job starting at 78 with two boosts would land at 78+16=94 in theory, but the model instead classified it as "good match, 80" and stopped.

### Fix
Rewrote `scoreMatch` system prompt with:
1. **4 explicit arithmetic steps**: "Step 1: assign BASE from rubric. Step 2: ADD 8 per boost (max +16). Step 3: SUBTRACT 15 per penalty. Step 4: Clamp to 0-100."
2. **Lowered base rubric top band to 82-88** (was 85-100) — so two boosts on an 84-base job → 84+16=100, and one boost on an 82-base job → 82+8=90. Scores above 88 are now achievable without exotic matches.
3. **6 calibration examples** showing exact arithmetic: "Fintech .NET + microservices → base 86 + Fintech +8 + microservices +8 = 102 → capped at 100", "Azure DevOps required → base 83 + 8 = 91", etc.
4. `matchReason` field now required to state which boosts/penalties were applied — forces the model to show its working.

### Expected Score Distribution (next run)
- Fintech/Healthcare .NET + microservices/Azure/Kafka → 90-100
- Strong .NET backend + one domain/tech boost → 88-93
- Good .NET backend, no boost → 75-82
- Partial/penalised matches → below 75 (filtered by minScore)

---

## [v9.3] — 2026-03-16

### Added
- **Bangalore location guard** in `filterStackQuality`: removes jobs whose location field mentions another Indian city (Mumbai, Hyderabad, Chennai, Pune, Delhi, Gurugram/Gurgaon, Noida, Kolkata, Ahmedabad, Jaipur, Indore, Coimbatore) without mentioning Bangalore or Bengaluru. Jobs with blank location or "Remote" pass through unchanged.
- **Company cap** in `filterStackQuality`: limits results to 3 roles per company per run. When a company has more than 3 listings, keeps the highest-seniority titles (Senior > SDE 2/SE II > other) and drops the rest. Company name is normalised by stripping common suffixes (India, Pvt, Ltd, Technologies, Solutions) before comparison.

### Why
Prevents Wipro/TCS/Infosys-style bulk listings from dominating the email, and removes the occasional non-Bangalore result that JSearch returns despite "Bangalore India" being embedded in every query.

---

## [v9.2] — 2026-03-16

### Changed
- **Experience range filter** added to `filterStackQuality`: hard-blocks jobs whose description contains "8+ years", "minimum 8 years", or "at least 8 years required". Ranges like "3-8 years" are not blocked (candidate falls within range). Target range: 4-6 years.
- **Scoring rubric tightened** in `scoreMatch` system prompt:
  - 85-100 band: changed "3-7 years required" → "3-6 years required"
  - 70-84 band: changed "3-8 years required" → "3-7 years required"
  - PENALIZE threshold: changed ">10 years required" → ">7 years required" (-15 points)

### Why
Candidate has ~5 years experience, targeting 4-6 year roles. Jobs requiring 8+ years as a minimum were previously passing through to OpenAI scoring and getting -15 points at most — still potentially reaching the inbox. Now they're blocked before scoring entirely. Jobs requiring 7 years get -15 in scoring and are unlikely to clear minScore 75.

---

## [v9.1] — 2026-03-16

### Added
- **Filter Stack & Quality** node (id: `filterStackQuality`, Code v2) inserted between `Filter New Jobs Only` and `Prepare Score Request`. Pre-score filter that blocks irrelevant jobs before OpenAI is called, reducing token cost and improving result quality.
- Filters: Java/Node.js/Python/Ruby/Go/PHP/Android/iOS/Flutter/React Native/Data Science/Salesforce/SAP primary roles; Fresher/Intern/Junior/Trainee titles; Principal/Staff/VP/CTO/Director seniority; stub descriptions < 150 chars. `.NET`/`C#` anywhere in title overrides stack blacklist.
- **Build Email Digest** updated: `keyMatchingSkills` pulled from `$('Attach Cover Letter').all()` (not available from Google Sheets output) and displayed as green tags (up to 5) in each job row.
- **Canvas layout fix**: all nodes from `filterStackQuality` onward repositioned to 250px spacing — was 120px causing overlap.

### Node Count
27 → 28 nodes, 26 → 27 connections

---

## [v9.0] — 2026-03-16

### Changed
- **JSearch queries** (2 replacements targeting Garima's specific profile):
  - `Full Stack Engineer React .NET Bangalore India` → `Microservices .NET Backend Engineer Bangalore India` (targets architecture depth)
  - `C# Developer Bangalore India` → `Azure .NET Developer Bangalore India` (targets Azure DevOps background)
- **Scoring system prompt** (`scoreMatch`) fully rewritten with candidate-specific rubric:
  - Candidate profile: C#/.NET Core, Kafka/RabbitMQ/Redis, Azure DevOps CI/CD, microservices, Fintech+Healthcare domains
  - Scoring bands: 85-100 (strong .NET, 3-7yr, microservices/domain), 70-84 (good .NET, 3-8yr), 50-69 (partial), 0-49 (no .NET)
  - Domain boost: +8 points for Fintech/BFSI/Healthcare/Azure/Kafka/microservices (stackable, max +16)
  - Penalty: -15 for frontend-primary, <2yr required, >10yr required, no .NET mention
- **Cover letter system prompt** (`generateCoverLetter`) updated with candidate name and quantified achievements (60% latency via Redis, 85% test coverage via TDD, 15% build time via .NET 6-to-8 migration)
- **minScore** corrected: live workflow had drifted to 60 despite v8.3 raising it to 75. Restored to 75.

### Why
Previous scoring was fully generic — no rubric, no domain awareness, no stack-specific rules. Previous queries included "Full Stack React .NET" (pulls frontend-heavy roles) and "C# Developer" (attracts junior/contractor roles). Both were the weakest queries by relevance. Expected quality improvement: 6/10 → 9/10.

---

## [v8.3] — 2026-03-09

### Changed
- Raised `minScore` from 60 → 75 in Set Job Preferences

### Why
Execution #70 (first successful end-to-end run) produced 71 final matches, all scoring between 75-85. The threshold of 60 was effectively doing nothing — the AI never scored qualifying jobs below 75 for this profile. At 60, every job that reached scoring also got a cover letter generated, which meant 71 cover letters to review — too many. At 75, the filter cuts to the genuinely strong matches (~30-50/run).

### Impact
- Final matches per run: ~71 → ~30-50 (higher confidence)
- Cover letters generated: proportionally fewer
- Runtime: slightly shorter (fewer cover letter API calls)

---

## [v8.2] — 2026-03-09

### Fixed
- Pipeline silently stopping on first run when Google Sheet is empty

### Root Cause
When `Read Existing Jobs` returns 0 rows (empty sheet), n8n skips all downstream nodes entirely — `Filter New Jobs Only` never ran, nothing got scored, no email sent. The workflow would show "success" in n8n but only 14 nodes had actually executed.

### Solution
Added `Sync Dedup Inputs` Merge node (mode: append) that receives:
- Input 0: `Aggregate Jobs` output — always exactly 1 item
- Input 1: `Read Existing Jobs` output — 0 rows on empty sheet, N rows otherwise

Since input 0 always has data, the Merge always fires regardless of sheet state. Updated `Filter New Jobs Only` to separate the two input shapes by data structure:
- Jobs item: `Array.isArray(i.json.jobs) === true`
- Sheet rows: everything else

Also fixed: Naukri `apply_link` field was being missed in `Parse Job Results` — the Apify actor uses snake_case (`apply_link`), not camelCase (`applyLink`, `jobUrl`). Added `job.apply_link` as first check in the fallback chain.

### Nodes Added
- `Sync Dedup Inputs` (Merge, mode: append, position [2450, 175])

### Node Count
26 → 27 nodes, 25 → 26 connections

---

## [v8.1] — 2026-03-09

### Fixed
- Google Sheets 429 "Too many requests" rate limit on `Read Existing Jobs`

### Root Cause
`Filter Duplicates` emits one item per unique job (~200 items). `Read Existing Jobs` (Google Sheets node) ran once per input item by default → ~200 API calls in rapid succession → rate limit hit.

### Solution
Added `Aggregate Jobs` Code node (`runOnceForAllItems`) between `Filter Duplicates` and `Read Existing Jobs`. It collapses N job items into a single item `{ jobs: [...], count: N }`, so `Read Existing Jobs` receives exactly 1 item and calls the Sheets API exactly once.

Verified with a separate test workflow ("Test: Cross-Run Dedup Fix") before applying to the main workflow.

### Nodes Added
- `Aggregate Jobs` (Code, runOnceForAllItems, position [2200, 50])

### Node Count
25 → 26 nodes, 24 → 25 connections

---

## [v8.0] — 2026-03-09

### Added
- 7th JSearch query: `Software Engineer II .NET Bangalore India`
- C# Naukri branch: 3rd parallel search branch — `Set Naukri Queries (C#)` → `Search Naukri C# (Apify)` → `Merge Search Results` (input 2)
- Cross-run deduplication: `Read Existing Jobs` (Google Sheets read) → `Filter New Jobs Only` inserted before scoring. Jobs already in the sheet from previous runs are skipped.
- `hashJob()` now uses normalized apply link URL (`hostname + pathname`, query params stripped) as the primary dedup key — more reliable than company+title across different job boards

### Changed
- Naukri queries split: was 1 query with `no_of_jobs: 60`, now 2 parallel queries (`.NET Developer` + `C# Developer`) with `no_of_jobs: 30` each — same volume, broader coverage
- Replaced `Software Developer .NET React Bangalore India` JSearch query with `C# Developer Bangalore India`
- Google Sheets `matchingColumns` updated to include `Apply Link` (was just Job Title + Company)
- Apify billing confirmed: $5/month **recurring free credit** (not one-time). Net Apify cost effectively $0/month at current usage.

### Architecture After v8.0

```
Set Job Preferences (7 items)
    |           |           |
 JSearch    Naukri .NET  Naukri C#
    |           |           |
    Merge Search Results (3 inputs)
    |
 Parse → Filter Duplicates → Aggregate Jobs
                                    |        \
                              Sync Dedup   Read Existing Jobs
                                    |
                             Filter New Jobs Only
                                    |
                            Score → Filter → Cover Letters → Sheet → Email
```

### Monthly Cost
~$0.30/month (JSearch free + Apify ~$0 within credit + OpenAI ~$0.30)

### Expected Volume
- Raw unique: ~200-210/run (7 JSearch queries × 4 pages + ~60 Naukri)
- After cross-run dedup (week 2+): ~20-60 new jobs/week
- Final matches (score ≥ 75): ~30-50/run

### Node Count
22 → 25 nodes (added: Set Naukri Queries C#, Search Naukri C# Apify, Read Existing Jobs, Filter New Jobs Only, Aggregate Jobs — wait, some were added across v8.0/8.1/8.2)

---

## [v7.0] — 2026-03-09

### Fixed
- Email digest showing `topScore: 0` and unsorted jobs

### Root Cause
After `Save to Google Sheets`, the node output uses column headers as object keys (`j['Match Score']`) not camelCase (`j.score`). `Build Email Digest` was only checking `j.score` → always 0. Sort was also broken for the same reason.

### Fix
Updated `getScore` in Build Email Digest: `Number(j['Match Score'] || j.score || 0)` — checks Sheets column name first, falls back to camelCase.

### Changed
- JSearch `num_pages`: 3 → 4 (more raw jobs per query)
- Naukri `no_of_jobs`: 40 → 60 (more Naukri results)
- API keys hardcoded with correct `sk-proj-` prefix (previous runs were failing auth)

### Monthly Cost
~$0.57/month

---

## [v6.1] — 2026-03-09

### Fixed
- `rapidApiKey` had a trailing space — JSearch calls were returning 401
- Apify actor timeout: 120000ms → 300000ms (actor takes 4-5 min; 2 min always timed out)
- Apify SSL: added `allowUnauthorizedCerts: true` to Search Naukri node
- Merge node had empty `parameters` object causing validation error

### Changed
- Tuned Naukri `no_of_jobs`: 60 → 40 (then further tuned in v7.0)
- JSearch `num_pages`: 2 → 3

### Monthly Cost
~$0.43/month

---

## [v6.0] — 2026-03-08

### Added
- **Naukri job source** via Apify actor `nuclear_quietude~naukri-job-scraper`
- Parallel search architecture: `Set Job Preferences` now forks into two branches simultaneously:
  - Branch A: JSearch (LinkedIn, Indeed, Glassdoor, Google Jobs)
  - Branch B: Set Naukri Queries → Search Naukri (Apify)
- `Merge Search Results` node (mode: append) combines both branches before parsing
- `apifyToken` added to Set Job Preferences base config
- `Parse Job Results` updated to handle both response formats:
  - JSearch: `item.json.data[]` array
  - Naukri/Apify: top-level `Array.isArray(item.json)` array

### Why Naukri
JSearch aggregates LinkedIn/Indeed/Glassdoor but has limited Naukri coverage. Adding a dedicated Naukri scraper captures the large India-specific job market that JSearch misses.

### Actor Selection Note
Pay-per-usage actor chosen over rented actor — rented = $19.99/month flat fee regardless of usage. Pay-per-usage charges only compute units from the $5/month free credit.

### Node Count
18 → 21 nodes

### Monthly Cost
~$0.17/month

---

## [v5.0] — 2026-03-08

### Changed
- **Job search**: SerpAPI → JSearch (RapidAPI)
  - JSearch aggregates LinkedIn + Indeed + Glassdoor + Google Jobs in a single API
  - Better dedup (one API, one response format, consistent fields)
  - Free tier: 200 req/month (used 12/run at the time)
- **AI model**: Groq llama-3.3-70b (free) → OpenAI GPT-4o-mini (paid, ~$0.03/run)
  - Groq free tier exhausted after multiple failed debug runs (100K tokens/day limit)
  - GPT-4o-mini: faster, more reliable JSON output, 500 RPM on pay-as-you-go
- **Resume**: Now sends full resume text (no 3000-char substring limit)
- **Batch interval**: 15s → 3s (OpenAI Tier 1 = 500 RPM, no need to throttle heavily)
- **Retries**: Disabled `retryOnFail` on both AI HTTP nodes — fail fast is better than wasting tokens on bad payloads
- **Payload field**: `groqPayload` → `aiPayload`
- **API key field**: `groqApiKey` → `openAiApiKey`
- **minScore**: Raised to 75 (Groq was scoring generously; GPT-4o-mini is stricter)
- **Cross-run dedup**: `Save to Google Sheets` switched to `appendOrUpdate` matching on Job Title + Company — re-running same week won't create duplicate rows

### Why Switch from Groq
Multiple failed debug runs exhausted the 100K token/day free limit. Retries (`retryOnFail: true, maxTries: 3`) made it worse — each retry consumed more tokens. Switching to OpenAI paid tier eliminated rate limit concerns and produced more reliable structured JSON output.

### Monthly Cost
~$0.11/month (mostly OpenAI; JSearch free)

---

## [v4.0] — 2026-03-08

### Fixed
- **Critical**: n8n Code node task runner has no HTTP client
  - `this.helpers.httpRequest` returns 400
  - `$helpers` is undefined
  - `fetch` is undefined
  - **Fix**: Split each AI step into two nodes: Prep Code node (builds JSON payload as string) + HTTP Request node (sends it)
- `json_validate_failed` 400 error from Groq API
  - **Root cause**: Embedding a JSON example (`{"score": 0, ...}`) in the system prompt when `response_format: json_object` is set causes the model to output the template verbatim, which fails schema validation
  - **Fix**: Removed all JSON syntax from system prompt — use plain field descriptions only
- PDF null character artifacts (`\u0000` replacing `+`, `%`, `~`, `()`)
  - **Fix**: `rawText.replace(/\u0000/g, '')` in Set Job Preferences

### Changed
- Architecture restructured to 18 nodes (up from 14 with monolithic Code nodes doing HTTP)
- Search strategy: India-wide → **Bangalore-only** (6 targeted queries with location embedded in query string)
- SerpAPI: added geo-targeting (`gl=in`, `hl=en`)
- Batch interval: 15s → 5s
- Resume file renamed: `resume.pdf` → `your-resume.pdf`

### JSearch Queries (6, Bangalore-targeted)
- `.NET Developer Bangalore India`
- `Backend Engineer .NET Bangalore India`
- `Full Stack Engineer React .NET Bangalore India`
- `SDE 2 .NET Bangalore India`
- `Senior Software Engineer .NET Bangalore India`
- `Software Developer .NET React Bangalore India`

### Node Count
14 → 18 nodes

---

## [v3.0] — 2026-02-24

### Added
- **PDF resume reading**: Two new nodes — `Read Resume PDF` (readBinaryFile) + `Extract PDF Text` (extractFromFile, joinPages: true)
- **Docker volume mount**: `./JobSearchAutomation/resumes:/home/node/.n8n-files` — required since n8n restricts file access to `/home/node/.n8n` and `/home/node/.n8n-files`
- **HTTP batching** on AI nodes: `batchSize=1`, `batchInterval=20000ms`
- **Retry logic**: `retryOnFail: true`, `maxTries=3`, `waitBetweenTries=15000ms` (later removed in v5.0)

### Changed
- AI model: Groq Llama 3.3 → **Groq Llama 3 70B** (better instruction following)
- Search scope: Bangalore only → **Dual-location** (Bangalore onsite/hybrid + India remote)
- Target roles: expanded to 17
- `minScore`: 70 → 50 (Llama 3 70B scored more conservatively)

### Node Count
14 → 16 nodes

---

## [v2.0] — 2026-02-09

### Changed
- **AI model**: GPT-5 → **Groq Llama 3.3** (free tier, 6000 RPM)
- Monthly cost: ~$13.50/month → **$0/month**
- Added `groqApiKey` to Set Job Preferences
- AI nodes call Groq API directly via HTTP Request with `Authorization: Bearer <key>` header
- Endpoint: `https://api.groq.com/openai/v1/chat/completions` (OpenAI-compatible)

### Why
GPT-5 cost was unsustainable for a personal job search tool running multiple times per week.

---

## [v1.0] — 2026-02-09 — Initial Release

### Architecture
14 nodes, manual trigger, linear pipeline:

```
Manual Trigger
→ Set Job Preferences (search queries + API keys)
→ Search Google Jobs (SerpAPI)
→ Parse Job Results
→ Filter Duplicates
→ Prepare Score Request (Code)
→ Score GPT-5 (HTTP)
→ Parse AI Score
→ Filter Top Matches (score ≥ 70)
→ Prepare Cover Letter (Code)
→ Cover Letter GPT-5 (HTTP)
→ Attach Cover Letter
→ Save to Google Sheets (append)
→ Build Email Digest
→ Send Gmail Digest
```

### Job Source
- SerpAPI Google Jobs — India-wide search, ~14 queries

### AI
- GPT-5 for scoring (structured JSON) + cover letter generation (plain text)
- Monthly cost: ~$13.50 (GPT-5 pricing)

### Scoring
- `minScore`: 70
- Fields: score, recommendation, matchReason, missingSkills

### Target
- Backend/.NET/React developer roles in India (remote + onsite)
- ~14 search queries

---

## Summary: Evolution at a Glance

| Version | Date | Nodes | Job Sources | Raw Jobs/Run | Final Jobs/Run | Cost/Month |
|---|---|---|---|---|---|---|
| v1.0 | 2026-02-09 | 14 | SerpAPI (India-wide) | ~40 | ~10-15 | ~$13.50 |
| v2.0 | 2026-02-09 | 14 | SerpAPI | ~40 | ~10-15 | $0.00 |
| v3.0 | 2026-02-24 | 16 | SerpAPI (dual-location) | ~60 | ~15-20 | $0.00 |
| v4.0 | 2026-03-08 | 18 | SerpAPI (Bangalore-only) | ~60 | ~15-25 | $0.00 |
| v5.0 | 2026-03-08 | 18 | JSearch (Bangalore) | ~80 | ~20-30 | ~$0.11 |
| v6.0 | 2026-03-08 | 21 | JSearch + Naukri .NET | ~140 | ~30-50 | ~$0.17 |
| v6.1 | 2026-03-09 | 21 | JSearch + Naukri .NET | ~140 | ~30-50 | ~$0.43 |
| v7.0 | 2026-03-09 | 21 | JSearch + Naukri .NET | ~170 | ~50-70 | ~$0.57 |
| v8.0 | 2026-03-09 | 25 | JSearch + Naukri .NET + C# | ~210 | ~30-60* | ~$0.30 |
| v8.1 | 2026-03-09 | 26 | JSearch + Naukri .NET + C# | ~210 | ~30-60* | ~$0.30 |
| v8.2 | 2026-03-09 | 27 | JSearch + Naukri .NET + C# | ~210 | ~30-60* | ~$0.30 |
| v8.3 | 2026-03-09 | 27 | JSearch + Naukri .NET + C# | ~210 | ~30-50 | ~$0.30 |
| v9.0 | 2026-03-16 | 28 | JSearch (profile-targeted) + Naukri .NET + C# | ~210 | ~25-45 | ~$0.30 |
| v9.1 | 2026-03-16 | 28 | same + pre-score stack/seniority filter | ~150** | ~20-35** | ~$0.30 |
| v9.2 | 2026-03-16 | 28 | same + 8+ yr exp filter | ~130** | ~18-32** | ~$0.30 |
| v9.3 | 2026-03-16 | 28 | same + location guard + company cap | ~110** | ~15-28** | ~$0.30 |
| v9.4 | 2026-03-16 | 28 | same | ~110** | ~15-28 (90-100 scores now possible) | ~$0.30 |
| v9.5 | 2026-03-17 | 28 | All 3 sources confirmed working | ~210 | ~25-45 | ~$0.30 |
| v9.6 | 2026-04-12 | 28 | same | ~210 | ~25-45 | ~$0.30 |
| **v9.7** | **2026-05-05** | **28** | **same** | **~210** | **~25-45** | **~$0.30** |

*After cross-run dedup — first run always passes all ~210 jobs to scoring.
**Pre-score filter blocks irrelevant stacks, overexperienced, off-location before OpenAI — reduces token usage and noise.

### Key Milestones

| Milestone | Version | Impact |
|---|---|---|
| Free AI (Groq) | v2.0 | Cost: $13.50 → $0 |
| PDF resume reading | v3.0 | AI now sees full resume, not just preferences |
| Split Code+HTTP (n8n architecture fix) | v4.0 | Pipeline actually works end-to-end |
| JSearch replaces SerpAPI | v5.0 | 4 job boards in 1 API, better coverage |
| Naukri via Apify added | v6.0 | India-specific listings now captured |
| Dual Naukri branches + cross-run dedup | v8.0 | No duplicate emails week over week |
| Rate limit fix (Aggregate Jobs) | v8.1 | Sheets API called once not 200 times |
| Empty sheet fix (Sync Dedup) | v8.2 | First run now works reliably |
| Proven run: 211 → 71 matches | v8.3 | Validated end-to-end, minScore tuned |
| Profile-targeted queries + candidate scoring | v9.0 | Quality: 6/10 → 9/10 (rubric, domain boosts, penalties) |
| Pre-score filter (stack/exp/location/company) | v9.1–v9.3 | Token savings + eliminates Java/overexperienced/off-city noise |
| Score ceiling fix (was stuck at 85 → now 90-100) | v9.4 | Best-fit roles now correctly score 90-100 |
| Naukri parse fix (60 jobs recovered/run) | v9.5 | All 3 job sources confirmed working end-to-end |
| keys.json + Success Summary fix | v9.6 | Secure key storage; correct match count/score in summary |

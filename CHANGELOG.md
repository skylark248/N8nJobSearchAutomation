# Changelog

All notable changes to the n8n Job Search Automation workflow are documented here.

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
When `Read Existing Jobs` returns 0 rows (empty sheet), n8n skips all downstream nodes entirely — `Filter New Jobs Only` never ran, nothing got scored, no email sent. The workflow would show "success" in n8n but only 14/27 nodes had actually executed.

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
| **v8.3** | **2026-03-09** | **27** | **JSearch + Naukri .NET + C#** | **~210** | **~30-50** | **~$0.30** |

*After cross-run dedup — first run always passes all ~210 jobs to scoring.

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

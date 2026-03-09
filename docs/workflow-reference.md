# Job Search Automation — Workflow Reference

## Overview

- **Trigger**: Manual (click to run) — replace with Schedule Trigger for weekly automation
- **Nodes**: 27 | **Connections**: 26
- **Estimated Runtime**: ~21 minutes per run
  - Naukri Apify actors: ~5 min (2 parallel calls)
  - AI scoring ~200 jobs at 3s/job: ~10 min
  - AI cover letters ~40 jobs at 3s/job: ~2 min
  - Other nodes: ~4 min
- **Monthly Cost**: ~$0.30 (JSearch free + Apify ~$0.04 + OpenAI ~$0.26 + Google APIs free)
- **Import file**: `exports/job-search-automation.json`
- **Proven results (execution #70)**: 211 unique raw → 71 final matches (score 75-85)

---

## Pipeline Architecture

### Three Parallel Search Branches

```
Set Job Preferences (7 items)
    |               |               |
    v               v               v
[Branch A]      [Branch B]      [Branch C]
JSearch         Naukri .NET     Naukri C#
7 HTTP calls    Apify actor     Apify actor
(4 pages each)  ~20-30 jobs     ~30-40 jobs
    |               |               |
    v               v               v
    Merge Search Results (append, inputs 0/1/2)
```

### Cross-Run Deduplication Sub-Graph

```
Filter Duplicates
    |
    v
Aggregate Jobs (collapse N items → 1)
    |               |
    v               v
Sync Dedup     Read Existing Jobs (Sheets, called once)
Inputs (Merge, append)
    |
    v
Filter New Jobs Only (skip already-seen jobs)
```

### Scoring + Output Pipeline

```
Filter New Jobs Only
    → Prepare Score Request
    → Score OpenAI API (batch 1, 3s)
    → Parse AI Score
    → Filter Top Matches (≥ 75)
    → Prepare Cover Letter
    → Cover Letter OpenAI API (batch 1, 3s)
    → Attach Cover Letter
    → Save to Google Sheets
    → Build Email Digest
    → Send Gmail Digest
    → Success Summary
```

---

## Node Details (27 Nodes)

### Node 1: Manual Trigger
- **ID**: `trigger` | **Type**: `n8n-nodes-base.manualTrigger` v1
- No parameters. Replace with Schedule Trigger for automation (see Scheduling section).

### Node 2: Read Resume PDF
- **ID**: `readResumePdf` | **Type**: `n8n-nodes-base.readBinaryFile` v1
- **filePath**: `/home/node/.n8n-files/your-resume.pdf`
- Maps to `resumes/your-resume.pdf` on host via Docker volume mount.
- To use a different resume: rename to `your-resume.pdf` or update `filePath` in this node.

### Node 3: Extract PDF Text
- **ID**: `extractPdfText` | **Type**: `n8n-nodes-base.extractFromFile` v1.1
- **operation**: `pdf`
- **options**: `{ joinPages: true }` — concatenates all pages into a single text string
- Output: `{ text: "full resume text..." }`

### Node 4: Set Job Preferences
- **ID**: `setPreferences` | **Type**: `n8n-nodes-base.code` v2
- Central config. Reads resume text, strips null chars, outputs **7 items** (one per JSearch query).
- All 7 items share the same `baseConfig` (API keys, minScore, email, spreadsheetId, resumeText).

**Key config values:**
```javascript
minScore: 75             // Jobs must score >= 75 to pass filter
openAiApiKey: '...'      // Must start with sk-proj-
rapidApiKey: '...'
apifyToken: '...'
yourEmail: '...'
spreadsheetId: '...'
```

**7 JSearch queries (Bangalore embedded in string — better than location param):**
```
.NET Developer Bangalore India
Backend Engineer .NET Bangalore India
Full Stack Engineer React .NET Bangalore India
SDE 2 .NET Bangalore India
Senior Software Engineer .NET Bangalore India
C# Developer Bangalore India
Software Engineer II .NET Bangalore India
```

**Why null char cleanup:** `extractFromFile` produces `\u0000` replacing `+`, `%`, `~`, `()` from some PDFs. Fixed with `rawText.replace(/\u0000/g, '')`.

### Node 5: Search Google Jobs (JSearch)
- **ID**: `searchJobs` | **Type**: `n8n-nodes-base.httpRequest` v4.2
- **URL**: `https://jsearch.p.rapidapi.com/search`
- **Headers**: `X-RapidAPI-Key: {{ $json.rapidApiKey }}`, `X-RapidAPI-Host: jsearch.p.rapidapi.com`
- **Query params**: `query={{ $json.searchQuery }}`, `page=1`, `num_pages=4`, `country=in`, `date_posted=week`, `employment_types=FULLTIME`, `language=en`
- **Batching**: `batchSize=1`, `batchInterval=1000ms` (rate limit protection)
- **Timeout**: 30000ms
- Runs once per input item = 7 searches × 4 pages = up to **~140 raw JSearch jobs**

### Node 6: Set Naukri Queries
- **ID**: `setNaukriQueries` | **Type**: `n8n-nodes-base.code` v2
- `runOnceForAllItems` — reads config from `$input.first().json`
- Returns **1 item**: `.NET Developer`, Bangalore with `apifyToken`

### Node 7: Search Naukri (Apify)
- **ID**: `searchNaukri` | **Type**: `n8n-nodes-base.httpRequest` v4.2
- **URL**: `https://api.apify.com/v2/acts/nuclear_quietude~naukri-job-scraper/run-sync-get-dataset-items`
- **Auth**: `token={{ $json.apifyToken }}` as query param
- **Body** (JSON): `{ job_title, location, experience: 5, no_of_jobs: 30, job_age: "7" }`
- **Timeout**: 300000ms — do NOT reduce. Actor takes 4-5 min.
- **Options**: `allowUnauthorizedCerts: true`
- Response: top-level JSON array (20-30 Naukri .NET jobs)

### Node 8: Set Naukri Queries (C#)
- **ID**: `setNaukriQueriesCSharp` | **Type**: `n8n-nodes-base.code` v2
- Same as Set Naukri Queries but returns `naukriJobTitle: 'C# Developer'`

### Node 9: Search Naukri C# (Apify)
- **ID**: `searchNaukriCSharp` | **Type**: `n8n-nodes-base.httpRequest` v4.2
- Identical config to Search Naukri (Apify) — runs in parallel for C# listings
- Response: top-level JSON array (30-40 Naukri C# jobs)

### Node 10: Merge Search Results
- **ID**: `mergeSearchResults` | **Type**: `n8n-nodes-base.merge` v3
- **mode**: `append`
- Input 0: 7 JSearch responses
- Input 1: 1 Naukri .NET response
- Input 2: 1 Naukri C# response
- Passes all responses to Parse Job Results

### Node 11: Parse Job Results
- **ID**: `parseJobs` | **Type**: `n8n-nodes-base.code` v2
- Handles two response formats in the same loop:
  - **JSearch**: `item.json.data[]` array → fields: `employer_name`, `job_title`, `job_city`, `job_state`, `job_apply_link`, `job_publisher`
  - **Naukri/Apify**: `Array.isArray(item.json)` → fields: `companyName`, `jobTitle`, `location`, `apply_link` (snake_case!)
- `hashJob()`: normalized URL (strips query params via `new URL(url).hostname + pathname`) as jobId; falls back to `company|title` if no URL
- Truncates description to 3000 chars
- Returns `{ jobs: [...allJobs], totalFound: N }` as single item

> **Naukri field name gotcha**: The actor uses `apply_link` (snake_case), not `jobUrl`, `url`, or `applyLink`. Parsing checks `job.apply_link` first.

### Node 12: Filter Duplicates
- **ID**: `filterDuplicates` | **Type**: `n8n-nodes-base.code` v2
- Map-based dedup on `jobId`
- Processes all items from Parse Job Results (which outputs a single `{jobs:[...]}` item)
- Emits **one item per unique job** (~200-210 items from ~220-240 raw)

### Node 13: Aggregate Jobs ← Rate Limit Fix
- **ID**: `aggregateJobs` | **Type**: `n8n-nodes-base.code` v2
- `runOnceForAllItems`
- **Purpose**: Collapses N individual job items into 1 item `{ jobs: [...], count: N }`
- Without this, Read Existing Jobs would run once per job item (200+ times → Google Sheets 429 rate limit)

```javascript
const jobs = $input.all().map(item => item.json);
return [{ json: { jobs, count: jobs.length } }];
```

### Node 14: Read Existing Jobs
- **ID**: `readExistingJobs` | **Type**: `n8n-nodes-base.googleSheets` v4.5
- **operation**: `read` — reads all rows from the Job Tracker sheet
- Triggered by Aggregate Jobs (always exactly 1 item → Sheets called exactly once)
- Returns 0 rows on first run (empty sheet), N rows on subsequent runs
- **Credential**: Google Sheets OAuth2

### Node 15: Sync Dedup Inputs ← Empty Sheet Fix
- **ID**: `syncDedup` | **Type**: `n8n-nodes-base.merge` v3
- **mode**: `append`
- Input 0: Aggregate Jobs output (always 1 item)
- Input 1: Read Existing Jobs output (0 rows on empty sheet, N rows otherwise)
- **Purpose**: Ensures Filter New Jobs Always runs. Without this, n8n skips all downstream nodes when Sheets returns 0 rows.

### Node 16: Filter New Jobs Only
- **ID**: `filterNewJobs` | **Type**: `n8n-nodes-base.code` v2
- `runOnceForAllItems`
- Separates jobs from sheet rows by shape:
  ```javascript
  const jobsItem = all.find(i => Array.isArray(i.json.jobs));  // from Aggregate Jobs
  const sheetRows = all.filter(i => !Array.isArray(i.json.jobs)); // from Read Existing Jobs
  ```
- Dedup logic:
  1. Build `seenLinks` Set from sheet `Apply Link` column (normalized URLs)
  2. Build `seenTitles` Set from `Company|Job Title` for jobs without URLs
  3. Filter: if job has a link → check `seenLinks`; otherwise → check `seenTitles`
- On first run (empty sheet): all ~200-210 jobs pass
- On subsequent runs: only truly new listings pass (~20-60/week)
- Throws "No new jobs this week" if everything was already seen

### Node 17: Prepare Score Request
- **ID**: `scoreMatch` | **Type**: `n8n-nodes-base.code` v2
- Maps all new job items to scoring payloads
- Builds `aiPayload = JSON.stringify({ model: 'gpt-4o-mini', messages, temperature: 0.1, response_format: { type: 'json_object' } })`
- System prompt: recruiter scoring a Backend/.NET/React dev with 5yr exp. No JSON examples in prompt (causes `json_validate_failed` 400 error).
- Passes `openAiApiKey` inline on each item

### Node 18: Score OpenAI API
- **ID**: `scoreGroqApi` | **Type**: `n8n-nodes-base.httpRequest` v4.2
- **URL**: `https://api.openai.com/v1/chat/completions`
- **Auth**: `Authorization: Bearer {{ $json.openAiApiKey }}`
- **Body**: `={{ $json.aiPayload }}` with `contentType: raw`, `rawContentType: application/json`
- **Batching**: `batchSize=1`, `batchInterval=3000ms`
- **retryOnFail**: false (fail fast — retries waste tokens)

### Node 19: Parse AI Score
- **ID**: `parseScore` | **Type**: `n8n-nodes-base.code` v2 | `runOnceForEachItem`
- Reads OpenAI response from `$input.item.json.choices[0].message.content`
- Gets job data from `$('Prepare Score Request').item` — NOT from Filter New Jobs
- Output fields: `score`, `matchReason`, `missingSkills[]`, `keyMatchingSkills[]`, `experienceFit`, `recommendation`

### Node 20: Filter Top Matches
- **ID**: `filterMatches` | **Type**: `n8n-nodes-base.code` v2
- Filters `score >= minScore` (75). Throws error with top score if nothing passes.
- Reads `minScore` from `$('Set Job Preferences').first().json.minScore`

### Node 21: Prepare Cover Letter
- **ID**: `generateCoverLetter` | **Type**: `n8n-nodes-base.code` v2
- Same pattern as Prepare Score Request
- Model: `gpt-4o-mini`, no `response_format` (plain text output)
- System: "under 300 words, plain text, no markdown"

### Node 22: Cover Letter OpenAI API
- **ID**: `coverLetterGroqApi` | **Type**: `n8n-nodes-base.httpRequest` v4.2
- Identical config to Score OpenAI API: Bearer auth, raw body, batchSize=1, batchInterval=3s, retryOnFail: false

### Node 23: Attach Cover Letter
- **ID**: `addCoverLetter` | **Type**: `n8n-nodes-base.code` v2 | `runOnceForEachItem`
- Reads cover letter from `$input.item.json.choices[0].message.content`
- Gets job data from `$('Prepare Cover Letter').item` — NOT from Filter Top Matches
- Adds `processedDate` field

### Node 24: Save to Google Sheets
- **ID**: `saveToSheet` | **Type**: `n8n-nodes-base.googleSheets` v4.5
- **Operation**: `appendOrUpdate`
- **Matching columns**: Job Title + Company + Apply Link (prevents duplicates across runs)
- **Sheet**: `Job Tracker`
- **10 columns mapped**: Date, Job Title, Company, Location, Match Score, Recommendation, Match Reason, Missing Skills, Apply Link, Cover Letter
- **Status column excluded** — preserves user's "Applied"/"Interview" edits on re-runs
- Uses `__rl: true` with `mode: "id"` for document reference

### Node 25: Build Email Digest
- **ID**: `buildEmail` | **Type**: `n8n-nodes-base.code` v2
- Sorts jobs by score using `getScore(j) = Number(j['Match Score'] || j.score || 0)`
  - Must check `j['Match Score']` first — after `Save to Google Sheets`, output uses column headers as keys, not camelCase
- HTML features:
  - Gradient header (#667eea → #764ba2)
  - Stats section: total matches + top score
  - Score colors: green ≥ 85, yellow ≥ 70, red < 70
  - Expandable cover letters via `<details>` tags
  - Apply buttons as HTML links

### Node 26: Send Gmail Digest
- **ID**: `sendDigest` | **Type**: `n8n-nodes-base.gmail` v2.1
- `emailType: "html"` (default in v2.1 — no explicit `operation` needed)
- Subject: `{N} Job Matches (Top: {score}/100) - {date}`

### Node 27: Success Summary
- **ID**: `successOutput` | **Type**: `n8n-nodes-base.code` v2
- Returns `{ success: true, matchCount, topScore, emailSent, sentTo, timestamp }`

---

## Connection Map (26 Connections)

```
Manual Trigger              → Read Resume PDF               (main)
Read Resume PDF             → Extract PDF Text              (main)
Extract PDF Text            → Set Job Preferences           (main)
Set Job Preferences         → Search Google Jobs (JSearch)  (main, branch A, input 0)
Set Job Preferences         → Set Naukri Queries            (main, branch B)
Set Job Preferences         → Set Naukri Queries (C#)       (main, branch C)
Set Naukri Queries          → Search Naukri (Apify)         (main)
Set Naukri Queries (C#)     → Search Naukri C# (Apify)      (main)
Search Google Jobs (JSearch)→ Merge Search Results          (main, input 0)
Search Naukri (Apify)       → Merge Search Results          (main, input 1)
Search Naukri C# (Apify)    → Merge Search Results          (main, input 2)
Merge Search Results        → Parse Job Results             (main)
Parse Job Results           → Filter Duplicates             (main)
Filter Duplicates           → Aggregate Jobs                (main)
Aggregate Jobs              → Sync Dedup Inputs             (main, input 0)
Aggregate Jobs              → Read Existing Jobs            (main)
Read Existing Jobs          → Sync Dedup Inputs             (main, input 1)
Sync Dedup Inputs           → Filter New Jobs Only          (main)
Filter New Jobs Only        → Prepare Score Request         (main)
Prepare Score Request       → Score OpenAI API              (main)
Score OpenAI API            → Parse AI Score                (main)
Parse AI Score              → Filter Top Matches            (main)
Filter Top Matches          → Prepare Cover Letter          (main)
Prepare Cover Letter        → Cover Letter OpenAI API       (main)
Cover Letter OpenAI API     → Attach Cover Letter           (main)
Attach Cover Letter         → Save to Google Sheets         (main)
Save to Google Sheets       → Build Email Digest            (main)
Build Email Digest          → Send Gmail Digest             (main)
Send Gmail Digest           → Success Summary               (main)
```

---

## Google Sheet Schema (12 Columns)

| Col | Header | Source | Notes |
|---|---|---|---|
| A | Date | `processedDate` from Attach Cover Letter | ISO date `YYYY-MM-DD` |
| B | Job Title | JSearch or Naukri | Raw title from listing |
| C | Company | JSearch or Naukri | Raw company name |
| D | Location | JSearch or Naukri | City/state or "Bangalore" fallback |
| E | Match Score | GPT-4o-mini (0-100) | Higher = better fit |
| F | Recommendation | GPT-4o-mini | `strong-apply` / `apply` / `maybe` / `skip` |
| G | Match Reason | GPT-4o-mini | 2-3 sentence explanation |
| H | Missing Skills | GPT-4o-mini | Comma-separated gaps |
| I | Apply Link | JSearch or Naukri | Direct URL to job posting |
| J | Cover Letter | GPT-4o-mini | Under 300 words, plain text |
| K | Status | Default: empty | **NOT overwritten** — user fills: Applied, Interview, etc. |
| L | Applied Date | Empty | User fills manually |

---

## Actual Pipeline Funnel (Execution #70)

| Stage | Count | Notes |
|---|---|---|
| Set Job Preferences | 7 items | 7 search queries |
| Search Google Jobs (JSearch) | 7 responses | 4 pages × 7 queries |
| Search Naukri .NET (Apify) | 20 jobs | Requested 30; fewer available |
| Search Naukri C# (Apify) | 40 jobs | Requested 30; actor returned more |
| Parse Job Results | 1 item | All sources combined |
| Filter Duplicates | 211 unique | From ~260 raw |
| Aggregate Jobs | 1 item | Rate limit fix working |
| Read Existing Jobs | 0 rows | First run — empty sheet |
| Sync Dedup Inputs | 1 item | Empty sheet fix — pipeline continued |
| Filter New Jobs Only | 211 new | All new on first run |
| Score OpenAI API | 211 scored | ~10 min at 3s/job |
| Filter Top Matches (≥ 75) | 71 final | Score range: 75-85 |
| Save to Google Sheets | 71 rows | |
| Gmail Digest | 1 email sent | |
| Total Runtime | ~21 min | |

---

## Key Technical Gotchas

### Code Nodes Cannot Make HTTP Requests
n8n task runner sandbox blocks all HTTP from Code nodes (`this.helpers.httpRequest` returns 400, `fetch` is undefined). Pattern for external APIs: Prep Code node builds payload → HTTP Request node sends it.

### OpenAI `json_validate_failed` 400 Error
Embedding a JSON example in the system prompt (`{"score": 0}`) confuses the model — it outputs the template, which fails JSON schema validation. Use plain field descriptions only.

### PDF Null Character Artifacts
`extractFromFile` on some PDFs outputs `\u0000` replacing `+`, `%`, `~`, `()`. Fixed with `rawText.replace(/\u0000/g, '')` in Set Job Preferences.

### Google Sheets Rate Limit (429)
Without Aggregate Jobs, Filter Duplicates emits ~200 items → Read Existing Jobs runs 200 times → rate limit. Aggregate Jobs collapses to 1 item → Sheets called once.

### Empty Sheet Pipeline Stop
When Read Existing Jobs returns 0 rows, n8n skips all downstream nodes (no data = no execution). Sync Dedup Inputs Merge node receives the Aggregate Jobs output (always 1 item) on input 0 — guaranteeing Filter New Jobs always fires.

### `runOnceForEachItem` Node References
- Parse AI Score → `$('Prepare Score Request').item` — must reference the immediately preceding Prep node
- Attach Cover Letter → `$('Prepare Cover Letter').item` — same pattern
- Using the wrong reference node causes paired data mismatch

### Build Email Digest: Column Name Keys
After `Save to Google Sheets`, node output uses column headers as object keys: `j['Match Score']` not `j.score`. Always use `j['Match Score'] || j.score` for robustness.

### Naukri Apply Link Field Name
The `nuclear_quietude~naukri-job-scraper` actor uses `apply_link` (snake_case). The Parse Job Results node checks `job.apply_link` first before other fallbacks.

---

## Scheduling

### Weekly (Recommended)

Replace Manual Trigger with Schedule Trigger — Monday 9 AM:

```json
{
  "type": "n8n-nodes-base.scheduleTrigger",
  "typeVersion": 1.2,
  "parameters": {
    "rule": {
      "interval": [{
        "triggerAtHour": 9,
        "triggerAtMinute": 0,
        "triggerAtDay": 1
      }]
    }
  }
}
```

Monday mornings are optimal — many companies post over the weekend.

---

## Version History

- **v8.3** (2026-03-09): Raised minScore 60 → 75. Execution #70 showed all 71 matches scored 75-85; minScore 60 was generating excessive cover letters without benefit.
- **v8.2** (2026-03-09): Fixed pipeline stopping when Google Sheet is empty. Added Sync Dedup Inputs Merge node — receives Aggregate Jobs (always 1 item, input 0) + Read Existing Jobs (0+ rows, input 1). Merge fires regardless of row count. Updated Filter New Jobs to separate jobs vs sheet rows by data shape. Fixed Naukri `apply_link` snake_case parsing in parseJobs. Node count: 27.
- **v8.1** (2026-03-09): Fixed Google Sheets 429 rate limit on Read Existing Jobs. Added Aggregate Jobs Code node (collapses N job items → 1) before Read Existing Jobs. Node count: 26.
- **v8.0** (2026-03-09): Added C# Naukri branch (3rd parallel source). Added cross-run dedup (Read Existing Jobs → Filter New Jobs Only) — only unseen jobs are scored. hashJob uses normalized apply link URL. Sheets matchingColumns includes Apply Link. Apify $5/month recurring credit confirmed. Expected raw: ~200-210 unique/run; after dedup: ~20-60 new/run.
- **v7.0** (2026-03-09): Fixed email topScore=0 (Build Email Digest uses `j['Match Score'] || j.score`). JSearch num_pages 3→4. Naukri no_of_jobs →60. Hardcoded API keys with correct sk-proj- prefix.
- **v6.1** (2026-03-09): Tuned volume. Fixed Apify timeout 120s→300s. Fixed SSL. Fixed Merge node parameters.
- **v6.0** (2026-03-08): Added Naukri via Apify as parallel source. Parse Job Results handles both JSearch and Naukri formats. 21 nodes.
- **v5.0** (2026-03-08): Switched to JSearch (RapidAPI) from SerpAPI. Switched to OpenAI GPT-4o-mini from Groq. Full resume sent. retryOnFail: false. minScore 75.
- **v4.0** (2026-03-08): Restructured to 18 nodes. Bangalore-only strategy. PDF null char fix. Fixed json_validate_failed.
- **v1.0-v3.0**: Initial build → Groq LLama → PDF resume → SerpAPI.

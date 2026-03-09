# n8n Job Search Automation

## Instance Configuration
- **URL**: http://localhost:5678
- **Environment**: Local Docker container (shared n8n instance with YtShortsAutomation)
- **Focus**: Automated job search with AI matching and cover letter generation
- **n8n Workflow ID**: `8vs888j57KeLaGtH` (deployed to running instance)
- **Workflow Name**: "Job Search Automation"
- **GitHub Repo**: https://github.com/skylark248/N8nJobSearchAutomation.git

## Parent Directory Context

This project lives at `/Users/shivanshchoudhary/Downloads/n8n/JobSearchAutomation/`. The parent `/n8n/` directory contains:
- `YtShortsAutomation/` -- sibling project (YouTube Shorts, separate repo)
- n8n runtime data: `database.sqlite`, `binaryData/`, `config`, `.mcp.json`, `crash.journal`
- The n8n Docker instance mounts the parent `/n8n/` directory as `/home/node/.n8n`
- Resumes volume: `./JobSearchAutomation/resumes` is mounted to `/home/node/.n8n-files` inside the container
- Workflow JSON files are imported via the n8n UI, not read from the filesystem at runtime

## User Profile

- **Role**: Backend-heavy Full Stack Developer (.NET + React), ~5 years experience
- **Target Roles** (12 total): SDE 2, Software Developer, Software Developer 2, Software Development Engineer 2, Software Engineer 2, Senior Software Engineer, .NET Developer, Senior .NET Developer, Backend Engineer, Senior Backend Engineer, Full Stack Engineer, Senior Full Stack Engineer
- **Location**: Bangalore only (onsite/hybrid/remote) — no India-wide remote
- **Resume**: `resumes/your-resume.pdf` (volume-mounted to `/home/node/.n8n-files/your-resume.pdf`)

## Quick Start

```bash
# 1. Start n8n (from parent /n8n/ directory)
docker compose up -d

# 2. Copy env file and fill in your API keys
cp .env.example .env

# 3. Import the workflow
# Open http://localhost:5678 -> Import from file -> select exports/job-search-automation.json

# 4. Configure credentials (see docs/setup-guide.md)

# 5. Place your resume PDF in resumes/ folder (filename must match Read Resume PDF node)

# 6. Create Google Sheet with 12-column headers (A-L)

# 7. Test run
```

## MCP Server Setup

### n8n MCP Server (czlonkowski/n8n-mcp)
Bridges Claude Code with n8n's workflow automation platform. Provides structured access to 1,084+ n8n nodes.

#### Setup Instructions

```bash
# Configure MCP server (replace YOUR_N8N_API_KEY with your actual key from n8n Settings -> API)
claude mcp add n8n-mcp \
  -e MCP_MODE=stdio \
  -e LOG_LEVEL=error \
  -e DISABLE_CONSOLE_OUTPUT=true \
  -e N8N_API_URL=http://localhost:5678 \
  -e N8N_API_KEY=YOUR_N8N_API_KEY \
  -s local \
  -- npx n8n-mcp
```

**Get your API key**: Open n8n -> Settings -> API -> Create API Key

**Verify setup:**
```bash
claude mcp list          # Should show n8n-mcp
claude mcp get n8n-mcp   # Check config
```

**Important Notes:**
- MCP tools are injected when a **new conversation begins** -- they don't appear retroactively
- "Failed to connect" when idle is **normal** -- it only connects during active conversations
- If tools aren't available after fixing issues, start a **new conversation**

#### Available MCP Tools (39 total)

**Documentation Tools:** `search_nodes`, `get_node`, `validate_node`, `validate_workflow`, `search_templates`, `get_template`, `tools_documentation`

**Workflow Management Tools** (requires API key): `n8n_create_workflow`, `n8n_get_workflow`, `n8n_update_partial_workflow`, `n8n_update_full_workflow`, `n8n_delete_workflow`, `n8n_list_workflows`, `n8n_validate_workflow`, `n8n_autofix_workflow`, `n8n_deploy_template`, `n8n_test_workflow`, `n8n_executions`, `n8n_health_check`

#### MCP Gotchas (Learned from Building This Workflow)

- **`n8n_update_partial_workflow` updateNode operation**: Use `"updates": {...}` key, NOT `"properties": {...}`. The latter causes "Missing required parameter 'updates'" error.
- **Gmail node requires explicit `operation: "send"`**: Without it, validation fails with "Invalid value for 'operation'".
- **Google Sheets resource locator**: Use `__rl: true` with `mode: "id"` for document/sheet references.
- **Background agents**: Bash commands are auto-denied when agents run in background mode. Use the Write tool directly for file operations.
- **Workflow creation**: Create with all nodes and connections in one `n8n_create_workflow` call for best results, then fix individual nodes with `n8n_update_partial_workflow`.

#### Troubleshooting

**npm cache permission errors:**
```bash
sudo chown -R $(id -u):$(id -g) "$HOME/.npm"
npx n8n-mcp --help   # verify fix
```

**Reconfigure from scratch:**
```bash
claude mcp remove n8n-mcp -s local
# Then re-run the setup command above
```

## Claude Skills

### n8n Skills (czlonkowski/n8n-skills)
Expert guidance for building production-ready n8n workflows. Skills activate automatically based on context.

**Available Skills:**
1. **Expression Syntax** (`n8n-expression-syntax`) - Correct expression patterns and variable access
2. **MCP Tools Expert** (`n8n-mcp-tools-expert`) - Proper use of n8n-mcp server tools
3. **Workflow Patterns** (`n8n-workflow-patterns`) - 5 proven architectural approaches
4. **Validation Expert** (`n8n-validation-expert`) - Interpret and fix validation errors
5. **Node Configuration** (`n8n-node-configuration`) - Operation-specific node setup requirements
6. **Code JavaScript** (`n8n-code-javascript`) - JavaScript implementation in Code nodes
7. **Code Python** (`n8n-code-python`) - Python limitations and standard library usage

## Active Workflows

### Job Search Automation
- **Status**: Inactive (manual trigger)
- **n8n Workflow ID**: `8vs888j57KeLaGtH`
- **Nodes**: 27 | **Connections**: 26
- **Estimated Runtime**: ~21 minutes per run (confirmed execution #70: Naukri ~5 min parallel + ~10 min scoring 211 jobs + ~3.5 min cover letters)
- **Export File**: `exports/job-search-automation.json`
- **Documentation**: See `docs/workflow-reference.md` for full node-by-node breakdown
- **Setup Guide**: See `docs/setup-guide.md` for credential configuration

**Pipeline**: Read Resume PDF -> Extract PDF Text -> Set Job Preferences (7 JSearch queries) -> [Branch A] Search Jobs x7 (JSearch, 4 pages each) + [Branch B] Set Naukri Queries -> Search Naukri .NET x1 (Apify, 30 jobs) + [Branch C] Set Naukri Queries (C#) -> Search Naukri C# x1 (Apify, 30 jobs) -> Merge Search Results (3 inputs) -> Parse & deduplicate results (~210 unique) -> Aggregate Jobs (collapse N→1) -> [Read Existing Jobs (Sheets) + Sync Dedup Inputs (Merge)] -> Filter New Jobs Only (cross-run dedup) -> Prepare Score Request (Code) -> Score OpenAI API (HTTP) -> Parse AI Score -> Filter top matches (score >= 75) -> Prepare Cover Letter (Code) -> Cover Letter OpenAI API (HTTP) -> Attach Cover Letter -> Save to Google Sheets (appendOrUpdate, dedup by Job Title+Company+Apply Link) -> Build Email Digest -> Send Gmail digest email

**Required Credentials** (configure in n8n before running):
1. **RapidAPI / JSearch** - `rapidApiKey` in Set Job Preferences. JSearch Basic is **free** (200 req/month, we use 24). Sign up at rapidapi.com, subscribe to JSearch free plan. Gets jobs from LinkedIn + Indeed + Glassdoor + Google Jobs.
2. **OpenAI API** - `openAiApiKey` in Set Job Preferences. Pay-as-you-go key from platform.openai.com. Uses `gpt-4o-mini` for scoring + cover letters (~$0.03/run).
3. **Apify** - `apifyToken` in Set Job Preferences. $5/month recurring free credit from apify.com. Used for Naukri scraping (2 searches/run: `.NET Developer` + `C# Developer`, 30 jobs each). Actor: `nuclear_quietude~naukri-job-scraper` (pay-per-usage, NOT a rented actor). Duration: ~4-5 min/run.
4. **Google OAuth2** - for Google Sheets (saving results) and Gmail (digest email). Enable both Sheets API and Gmail API in Google Cloud Console. Redirect URI: `http://localhost:5678/rest/oauth2-credential/callback`

**Monthly Cost**: ~$0.30/month (4 runs/month) -- JSearch free + Apify $0 (within $5/month credit) + OpenAI ~$0.30 + Google APIs free

### Complete Node List (27 nodes)

| # | Node Name | Type | Version | ID | Purpose |
|---|---|---|---|---|---|
| 1 | Manual Trigger | `n8n-nodes-base.manualTrigger` | v1 | `trigger` | Click to run (replace with Schedule for daily) |
| 2 | Read Resume PDF | `n8n-nodes-base.readBinaryFile` | v1 | `readResumePdf` | Reads PDF from `/home/node/.n8n-files/your-resume.pdf` |
| 3 | Extract PDF Text | `n8n-nodes-base.extractFromFile` | v1.1 | `extractPdfText` | Extracts text from PDF binary (operation: pdf, joinPages: true) |
| 4 | Set Job Preferences | `n8n-nodes-base.code` | v2 | `setPreferences` | Central config: cleans null chars from PDF text, 7 Bangalore searches, minScore 75, openAiApiKey + rapidApiKey + apifyToken |
| 5 | Search Google Jobs (JSearch) | `n8n-nodes-base.httpRequest` | v4.2 | `searchJobs` | GET jsearch.p.rapidapi.com/search. Headers: X-RapidAPI-Key. Params: query, num_pages=4, country=in, date_posted=week |
| 6 | Set Naukri Queries | `n8n-nodes-base.code` | v2 | `setNaukriQueries` | Reads config from first item, returns 1 Naukri query item (.NET Developer, Bangalore) |
| 7 | Search Naukri (Apify) | `n8n-nodes-base.httpRequest` | v4.2 | `searchNaukri` | POST Apify nuclear_quietude~naukri-job-scraper. Fields: job_title, location, experience=5, no_of_jobs=30, job_age=7. Timeout: 300000ms |
| 8 | Set Naukri Queries (C#) | `n8n-nodes-base.code` | v2 | `setNaukriQueriesCSharp` | Returns 1 Naukri query item (C# Developer, Bangalore) for parallel C# branch |
| 9 | Search Naukri C# (Apify) | `n8n-nodes-base.httpRequest` | v4.2 | `searchNaukriCSharp` | POST Apify nuclear_quietude~naukri-job-scraper. no_of_jobs=30. Timeout: 300000ms |
| 10 | Merge Search Results | `n8n-nodes-base.merge` | v3 | `mergeSearchResults` | Appends JSearch (input 0) + Naukri .NET (input 1) + Naukri C# (input 2) responses before parsing |
| 11 | Parse Job Results | `n8n-nodes-base.code` | v2 | `parseJobs` | Handles JSearch (data[]) and Apify (top-level array). Naukri apply link: `apply_link` (snake_case). hashJob by normalized URL |
| 12 | Filter Duplicates | `n8n-nodes-base.code` | v2 | `filterDuplicates` | Map-based dedup on jobId (normalized apply link), emits one item per unique job |
| 13 | Aggregate Jobs | `n8n-nodes-base.code` | v2 | `aggregateJobs` | Collapses N job items into 1 item `{jobs:[...]}` — ensures Read Existing Jobs runs exactly once |
| 14 | Read Existing Jobs | `n8n-nodes-base.googleSheets` | v4.5 | `readExistingJobs` | Reads all rows from Job Tracker sheet to get previously seen apply links |
| 15 | Sync Dedup Inputs | `n8n-nodes-base.merge` | v3 | `syncDedup` | Merge (append): input 0 = Aggregate Jobs (always 1 item), input 1 = Read Existing Jobs (0+ rows). Guarantees Filter New Jobs runs even on empty sheet |
| 16 | Filter New Jobs Only | `n8n-nodes-base.code` | v2 | `filterNewJobs` | Cross-run dedup: separates jobs vs sheet rows by shape, filters already-seen apply links |
| 17 | Prepare Score Request | `n8n-nodes-base.code` | v2 | `scoreMatch` | Builds aiPayload JSON string per job with full resume, passes openAiApiKey inline |
| 18 | Score OpenAI API | `n8n-nodes-base.httpRequest` | v4.2 | `scoreGroqApi` | POST api.openai.com/v1/chat/completions. Auth: Bearer header. batchSize=1, batchInterval=3000ms. retryOnFail: false |
| 19 | Parse AI Score | `n8n-nodes-base.code` | v2 | `parseScore` | runOnceForEachItem. Reads OpenAI response + job data from $('Prepare Score Request').item |
| 20 | Filter Top Matches | `n8n-nodes-base.code` | v2 | `filterMatches` | Filters score >= minScore (75). Throws if no matches |
| 21 | Prepare Cover Letter | `n8n-nodes-base.code` | v2 | `generateCoverLetter` | Builds cover letter aiPayload per job with full resume |
| 22 | Cover Letter OpenAI API | `n8n-nodes-base.httpRequest` | v4.2 | `coverLetterGroqApi` | POST api.openai.com/v1/chat/completions. batchSize=1, batchInterval=3000ms. retryOnFail: false |
| 23 | Attach Cover Letter | `n8n-nodes-base.code` | v2 | `addCoverLetter` | runOnceForEachItem. Reads cover letter from $input.item + job from $('Prepare Cover Letter').item |
| 24 | Save to Google Sheets | `n8n-nodes-base.googleSheets` | v4.5 | `saveToSheet` | appendOrUpdate matching on Job Title+Company+Apply Link. Status column excluded (preserves user edits) |
| 25 | Build Email Digest | `n8n-nodes-base.code` | v2 | `buildEmail` | HTML email, maps both Google Sheets column names and camelCase field names |
| 26 | Send Gmail Digest | `n8n-nodes-base.gmail` | v2.1 | `sendDigest` | Send HTML digest |
| 27 | Success Summary | `n8n-nodes-base.code` | v2 | `successOutput` | Return matchCount, topScore, emailSent, timestamp |

### Connection Map (26 connections)

```
Manual Trigger                -> Read Resume PDF               (main)
Read Resume PDF               -> Extract PDF Text              (main)
Extract PDF Text              -> Set Job Preferences           (main)
Set Job Preferences           -> Search Google Jobs (JSearch)  (main, branch A)
Set Job Preferences           -> Set Naukri Queries            (main, branch B)
Set Job Preferences           -> Set Naukri Queries (C#)       (main, branch C)
Set Naukri Queries            -> Search Naukri (Apify)         (main)
Set Naukri Queries (C#)       -> Search Naukri C# (Apify)     (main)
Search Google Jobs (JSearch)  -> Merge Search Results          (main, input 0)
Search Naukri (Apify)         -> Merge Search Results          (main, input 1)
Search Naukri C# (Apify)      -> Merge Search Results          (main, input 2)
Merge Search Results          -> Parse Job Results             (main)
Parse Job Results             -> Filter Duplicates             (main)
Filter Duplicates             -> Aggregate Jobs                (main)
Aggregate Jobs                -> Sync Dedup Inputs             (main, input 0)
Aggregate Jobs                -> Read Existing Jobs            (main)
Read Existing Jobs            -> Sync Dedup Inputs             (main, input 1)
Sync Dedup Inputs             -> Filter New Jobs Only          (main)
Filter New Jobs Only          -> Prepare Score Request         (main)
Prepare Score Request         -> Score OpenAI API              (main)
Score OpenAI API              -> Parse AI Score                (main)
Parse AI Score                -> Filter Top Matches            (main)
Filter Top Matches            -> Prepare Cover Letter          (main)
Prepare Cover Letter          -> Cover Letter OpenAI API       (main)
Cover Letter OpenAI API       -> Attach Cover Letter           (main)
Attach Cover Letter           -> Save to Google Sheets         (main)
Save to Google Sheets         -> Build Email Digest            (main)
Build Email Digest            -> Send Gmail Digest             (main)
Send Gmail Digest             -> Success Summary               (main)
```

### Key Node Implementation Details

**Read Resume PDF** (readBinaryFile node):
- Reads from `/home/node/.n8n-files/your-resume.pdf` (volume-mounted from `./resumes/your-resume.pdf`)
- n8n restricts file access to `/home/node/.n8n` and `/home/node/.n8n-files` via `N8N_RESTRICT_FILE_ACCESS_TO`
- To change resume: place new PDF in `resumes/`, rename to `your-resume.pdf` (or update `filePath` in this node)

**Extract PDF Text** (extractFromFile node):
- Operation: `pdf`
- Options: `joinPages: true` (concatenates all pages into single text field)
- Outputs `{ text: "..." }` with the full resume text

**Set Job Preferences** (Code node):
- Reads resume text from `$input.first().json.text` (from Extract PDF Text)
- **Cleans null chars**: `rawText.replace(/\u0000/g, '')` — PDF extraction produces `\u0000` artifacts replacing `+`, `%`, `~`, `()` etc.
- Validates resume is not empty or too short (< 50 chars)
- Returns **7 items** for Bangalore-only targeted searches (location embedded in query string for JSearch):
  - `.NET Developer Bangalore India`, `Backend Engineer .NET Bangalore India`
  - `Full Stack Engineer React .NET Bangalore India`, `SDE 2 .NET Bangalore India`
  - `Senior Software Engineer .NET Bangalore India`, `C# Developer Bangalore India`
  - `Software Engineer II .NET Bangalore India`
- `minScore`: 75
- API keys `openAiApiKey` (OpenAI), `rapidApiKey` (RapidAPI/JSearch), `apifyToken` (Apify/Naukri) stored here
- `yourEmail` and `spreadsheetId`: configured inline
- Triggers all 3 branches in parallel: JSearch, Naukri .NET, Naukri C#

**Search Jobs (JSearch)** (HTTP Request, id: `searchJobs`):
- JSearch endpoint: `https://jsearch.p.rapidapi.com/search`
- Headers: `X-RapidAPI-Key: {{ $json.rapidApiKey }}`, `X-RapidAPI-Host: jsearch.p.rapidapi.com`
- Query params: `query={{ searchQuery }}`, `page=1`, `num_pages=4`, `country=in`, `date_posted=week`, `employment_types=FULLTIME`, `language=en`
- `num_pages=4` returns up to 40 jobs per query (4 pages × 10 jobs) in a single response
- Sources: LinkedIn, Indeed, Glassdoor, Google Jobs combined
- Runs once per input item (7 searches total = up to 280 raw → ~135 unique after dedup)
- Timeout: 30000ms

**Set Naukri Queries** (Code node, id: `setNaukriQueries`):
- Runs once for all items (reads config from `$input.first().json`)
- Returns **1 Naukri query item**: `.NET Developer`, Bangalore
- Item contains `apifyToken`, `naukriJobTitle`, `naukriLocation` for the Apify call

**Search Naukri (Apify)** (HTTP Request, id: `searchNaukri`):
- POST `https://api.apify.com/v2/acts/nuclear_quietude~naukri-job-scraper/run-sync-get-dataset-items`
- Auth: `token={{ $json.apifyToken }}` as query param
- Body (JSON): `{ job_title: ..., location: ..., experience: 5, no_of_jobs: 30, job_age: "7" }`
- Timeout: 300000ms (Apify sync runs take 4-5 min)
- Response: top-level JSON array of job objects

**Set Naukri Queries (C#)** (Code node, id: `setNaukriQueriesCSharp`):
- Runs once for all items (reads config from `$input.first().json`)
- Returns **1 Naukri query item**: `C# Developer`, Bangalore
- Item contains `apifyToken`, `naukriJobTitle`, `naukriLocation` for the Apify call

**Search Naukri C# (Apify)** (HTTP Request, id: `searchNaukriCSharp`):
- POST `https://api.apify.com/v2/acts/nuclear_quietude~naukri-job-scraper/run-sync-get-dataset-items`
- Auth: `token={{ $json.apifyToken }}` as query param
- Body (JSON): `{ job_title: 'C# Developer', location: 'Bangalore', experience: 5, no_of_jobs: 30, job_age: "7" }`
- Timeout: 300000ms, `allowUnauthorizedCerts: true`
- Response: top-level JSON array of job objects

**Merge Search Results** (Merge node, id: `mergeSearchResults`):
- mode: `append` — concatenates items from all 3 inputs
- Input 0: JSearch responses (7 items)
- Input 1: Naukri .NET/Apify responses
- Input 2: Naukri C#/Apify responses
- Passes all items to Parse Job Results

**Parse Job Results** (Code node):
- Processes `$input.all()` to handle merged JSearch + Naukri responses (9 items total)
- **JSearch format** (`item.json.data` is array): extracts `employer_name`, `job_title`, `job_city/state`, `job_apply_link`, `job_publisher`
- **Apify/Naukri format** (`Array.isArray(item.json)`): extracts `companyName/company`, `jobTitle/title/positionName`, `location/jobLocation`, `jobUrl/url`, `salary/ctc`
- Generates `jobId` hash from **normalized apply link URL** (strips query params via `new URL(url).hostname + pathname`); falls back to `company|title` if no URL
- **Naukri apply link field**: `apply_link` (snake_case) — code checks `job.apply_link || job.jobUrl || job.url || job.applyLink`
- **Naukri posted date field**: `posted` — code checks `job.datePosted || job.postedDate || job.posted || job.date`
- Truncates description to 3000 chars for AI context window
- Returns single item with `{ jobs: [...], totalFound: N }`

**Filter Duplicates** (Code node):
- Map-based dedup on `jobId` (normalized apply link), emits one item per unique job
- Typically ~180 unique jobs after within-run deduplication

**Aggregate Jobs** (Code node, id: `aggregateJobs`):
- Runs `runOnceForAllItems` — collapses N job items into a single item `{ jobs: [...], count: N }`
- **Why**: `readExistingJobs` would run once per input item (~180 calls → rate limit). With this node: 1 item → 1 API call.
- Outputs to both `Sync Dedup Inputs` (input 0) AND `Read Existing Jobs` in parallel

**Read Existing Jobs** (Google Sheets v4.5, id: `readExistingJobs`):
- Operation: `read`, sheet: `Job Tracker`
- Reads all existing rows to extract previously seen apply links
- Passes all rows as items to Filter New Jobs Only

**Sync Dedup Inputs** (Merge node v3, id: `syncDedup`):
- mode: `append` — combines input 0 (Aggregate Jobs, always 1 item) + input 1 (Read Existing Jobs, 0+ rows)
- **Why needed**: If sheet is empty, `readExistingJobs` outputs 0 items → n8n would skip `filterNewJobs`. The Merge guarantees `filterNewJobs` always receives the aggregate item even on first run.

**Filter New Jobs Only** (Code node, id: `filterNewJobs`):
- Receives merged items from `Sync Dedup Inputs`
- Separates by shape: `Array.isArray(i.json.jobs)` = aggregate jobs item; others = sheet rows
- Normalizes apply links (strip query params) for comparison; falls back to company|title
- Throws if 0 new jobs (all already seen)
- Output: only jobs not previously saved to the sheet

**Prepare Score Request** (Code node, id: `scoreMatch`):
- Runs `$input.all().map()` to build one payload item per job
- Builds `aiPayload` = `JSON.stringify({model: 'gpt-4o-mini', messages, temperature: 0.1, response_format: {type: 'json_object'}})`
- Sends **full resume** (no substring limit)
- System message: plain field list — **do NOT include embedded JSON example** (causes `json_schema_invalid` error)
- User message: `"RESUME:\n" + resume + "\n\nJOB POSTING:\n" + jobDesc`
- Passes `openAiApiKey` inline on each item

**Score OpenAI API** (HTTP Request, id: `scoreGroqApi`):
- POST `https://api.openai.com/v1/chat/completions`
- Auth: `Authorization: Bearer {{ $json.openAiApiKey }}` header
- Body: `={{ $json.aiPayload }}` with `contentType: raw`, `rawContentType: application/json`
- **HTTP batching**: batchSize=1, batchInterval=3000ms
- **retryOnFail**: false (fail fast — retries waste tokens and mask real errors)
- Timeout: 60000ms

**Parse AI Score** (Code node, runOnceForEachItem):
- Uses `$input.item.json` for the OpenAI response (`choices[0].message.content`)
- References job data via `$('Prepare Score Request').item` (NOT Filter Duplicates)
- Parses JSON with try/catch, falls back to `{}` on error
- Merges: score, recommendation, matchReason, missingSkills, keyMatchingSkills, experienceFit

**Filter Top Matches** (Code node):
- Uses `$input.all()`, filters `score >= minScore`
- Throws error with top score if no matches found
- minScore default: 75

**Prepare Cover Letter** (Code node, id: `generateCoverLetter`):
- Same pattern as Prepare Score Request but for cover letter
- Model: `gpt-4o-mini`, no `response_format` (plain text output)
- Sends full resume; system message instructs plain text, under 300 words

**Cover Letter OpenAI API** (HTTP Request, id: `coverLetterGroqApi`):
- Same config as Score OpenAI API: Bearer auth, raw body, batchSize=1, batchInterval=3000ms, retryOnFail: false

**Attach Cover Letter** (Code node, runOnceForEachItem):
- Reads cover letter from `$input.item.json.choices[0].message.content`
- References job data via `$('Prepare Cover Letter').item` (NOT Filter Top Matches)
- Adds `processedDate` field

**Save to Google Sheets** (v4.5):
- Operation: `appendOrUpdate` with `matchingColumns: ["Job Title", "Company", "Apply Link"]` — prevents duplicate rows (same title at same company but different URL = different row)
- Sheet name: `Job Tracker`
- 10 columns mapped (Date, Job Title, Company, Location, Match Score, Recommendation, Match Reason, Missing Skills, Apply Link, Cover Letter)
- **Status column excluded from mapping** — prevents overwriting user's "Applied"/"Interview" status on re-runs
- Uses `__rl: true` resource locator format for document/sheet references

**Build Email Digest** (Code node):
- **Maps both Google Sheets column names and camelCase field names** for robustness:
  - `j['Job Title'] || j.title` -- Google Sheets returns column headers as keys
  - `j['Match Score'] || j.score` -- camelCase is used before Sheets step
- HTML with gradient header (#667eea to #764ba2)
- Stats section showing total matches and top score
- Job table with color-coded scores: green (>=85), yellow (>=70), red (<70)
- Apply buttons as HTML links
- Expandable cover letters using `<details>` HTML tags

**Send Gmail Digest** (v2.1):
- `emailType: "html"` (default, no explicit operation needed in v2.1)
- Subject and body from Build Email Digest output
- To address from Set Job Preferences `yourEmail`

### Google Sheet Schema (12 columns)

| Column | Header | Source |
|---|---|---|
| A | Date | `new Date().toISOString().split('T')[0]` |
| B | Job Title | JSearch or Naukri result |
| C | Company | JSearch or Naukri result |
| D | Location | JSearch or Naukri result |
| E | Match Score | GPT-4o-mini scoring (0-100) |
| F | Recommendation | GPT-4o-mini (strong-apply/apply/maybe/skip) |
| G | Match Reason | GPT-4o-mini (2-3 sentences) |
| H | Missing Skills | GPT-4o-mini (comma-separated) |
| I | Apply Link | JSearch or Naukri result |
| J | Cover Letter | GPT-4o-mini generation (under 300 words) |
| K | Status | Default: "Not Applied" — user fills (NOT overwritten on re-run) |
| L | Applied Date | Empty (user fills manually) |

### Placeholder Values to Configure

These values in the exported JSON need to be replaced before running:

| Placeholder | Where | Replace With |
|---|---|---|
| `YOUR_RAPIDAPI_KEY` | Set Job Preferences node, `rapidApiKey` field | Your RapidAPI key from rapidapi.com (subscribe to JSearch Basic, free tier) |
| `YOUR_OPENAI_API_KEY` | Set Job Preferences node, `openAiApiKey` field | Your OpenAI API key from platform.openai.com/api-keys |
| `YOUR_APIFY_TOKEN` | Set Job Preferences node, `apifyToken` field | Your Apify API token from console.apify.com/account/integrations |
| ~~`YOUR_APIFY_NAUKRI_ACTOR_ID`~~ | Search Naukri (Apify) node URL | **Already set**: `nuclear_quietude~naukri-job-scraper` (pay-per-usage, free tier compatible) |
| `YOUR_EMAIL@gmail.com` | Set Job Preferences node, `yourEmail` field | Your Gmail address |
| `YOUR_GOOGLE_SHEET_ID` | Set Job Preferences node + Save to Google Sheets node | Spreadsheet ID from Google Sheets URL |
| `your-resume.pdf` | Read Resume PDF node, `filePath` field | Place resume at `resumes/your-resume.pdf` |

### Switching to Daily Schedule

Replace "Manual Trigger" with Schedule Trigger:
```json
{
  "type": "n8n-nodes-base.scheduleTrigger",
  "typeVersion": 1.2,
  "parameters": {
    "rule": {
      "interval": [
        {
          "triggerAtHour": 9,
          "triggerAtMinute": 0
        }
      ]
    }
  }
}
```
Runs weekday mornings at 9 AM. Use cron `0 9 * * 1-5` for weekdays only.

## Project Structure

```
.
├── README.md                 # Start here -- project overview & quickstart
├── CLAUDE.md                 # Claude Code instructions (this file)
├── CHANGELOG.md              # Full version history v1.0 → current, with rationale per change
├── .gitignore                # Excludes sensitive files, resumes, debug artifacts, .agents/
├── .gitmodules               # Git submodule references (n8n-mcp, n8n-skills)
│
├── resumes/                  # Resume PDFs (volume-mounted to /home/node/.n8n-files)
│   ├── your-resume.pdf       # Current resume (rename yours to this)
│   ├── README.txt            # Instructions for resume placement
│   └── .gitkeep              # Keeps folder in git
│
├── docs/                     # All documentation
│   ├── setup-guide.md        # Step-by-step credential setup (RapidAPI, OpenAI, Apify, Google OAuth2)
│   └── workflow-reference.md # Complete 27-node workflow documentation with funnel data
│
├── exports/                  # Importable workflow JSON files
│   └── job-search-automation.json  # Clean export (no credentials — all replaced with YOUR_* placeholders)
│
├── n8n-mcp/                  # MCP server (git submodule: github.com/czlonkowski/n8n-mcp)
└── n8n-skills/               # Claude skills (git submodule: github.com/czlonkowski/n8n-skills)
```

## Workflow Development Guidelines

### Best Practices
1. Use descriptive names for workflows and nodes
2. Add notes to complex logic for maintainability
3. Handle errors gracefully with Error Trigger nodes
4. Test workflows with sample data before activating
5. Never commit API keys or credentials -- use `.env` + `.gitignore`
6. Always validate workflows after creation with `n8n_validate_workflow`
7. Use `n8n_autofix_workflow` to fix expression format and version issues
8. When updating nodes via MCP, use `n8n_update_partial_workflow` with `updateNode` operation and `"updates": {...}` key
9. Export clean workflow JSON by fetching via `n8n_get_workflow` and stripping credentials/metadata

### Common Automation Patterns
- **Scheduled tasks**: Use Cron/Schedule triggers
- **AI Agent pipelines**: Use AI Agent node + tools for research/content generation
- **Job search**: Read Resume PDF -> Extract Text -> HTTP Request (JSearch/RapidAPI) -> Code (parse) -> OpenAI GPT-4o-mini (score/generate) -> Google Sheets (save) -> Gmail (notify)

### n8n Expression Syntax Quick Reference
- Reference current item: `{{ $json.fieldName }}`
- Reference another node: `{{ $('Node Name').first().json.fieldName }}`
- Reference same-index item from another node: `{{ $('Node Name').item.json.fieldName }}`
- Current date: `{{ $now.format('yyyy-MM-dd') }}`
- n8n expressions start with `=` when used in node parameters: `={{ $json.field }}`

## Cost Estimate

| Service | Per Run | Monthly (4 runs) |
|---|---|---|
| JSearch Basic (RapidAPI) | — | $0.00 (free tier: 200 req/mo, uses 28/run at 7 queries × num_pages=4) |
| Apify Naukri scraper (2 searches/run, `nuclear_quietude~naukri-job-scraper`) | ~$0.10 | $0.00 (within $5/month recurring credit) |
| OpenAI GPT-4o-mini scoring (~50 new jobs/run × ~5500 tokens) | ~$0.060 | ~$0.24 |
| OpenAI GPT-4o-mini cover letters (~20 jobs × ~5500 tokens) | ~$0.015 | ~$0.06 |
| Google Sheets + Gmail | $0.00 | $0.00 |
| **Total** | **~$0.075** | **~$0.30** |

## Version History

- **v8.3** (2026-03-09): Raised minScore 60 → 75. Execution #70 produced 71 jobs (all scored 75-85); minScore 60 was passing everything and generating excessive cover letters. At 75 threshold, expect ~30-50 final jobs/run going forward.
- **v8.2** (2026-03-09): Fixed pipeline stopping when sheet is empty. Root cause: `readExistingJobs` outputs 0 rows on first run → n8n skips `filterNewJobs` → nothing scored/emailed. Fix: added `Sync Dedup Inputs` Merge node receiving Aggregate Jobs (input 0, always 1 item) + Read Existing Jobs (input 1, 0+ rows) — Merge always fires even with empty sheet. Updated `filterNewJobs` to separate jobs vs sheet rows by shape. Fixed `parseJobs` Naukri apply link: `apply_link` (snake_case) was not being captured. Node count: 27.
- **v8.1** (2026-03-09): Fixed Google Sheets rate limit on Read Existing Jobs. Root cause: node ran once per job item (~180 calls). Fix: added Aggregate Jobs Code node between Filter Duplicates and Read Existing Jobs — collapses N items into 1, so Sheets is called exactly once. Updated Filter New Jobs Only to reference `$('Aggregate Jobs').first().json.jobs`. Node count: 26.
- **v8.0** (2026-03-09): Added 7th JSearch query (`C# Developer Bangalore India` + `Software Engineer II .NET Bangalore India`, dropped `Software Developer .NET React`). Added C# Naukri branch (3rd parallel branch: Set Naukri Queries C# → Search Naukri C# → Merge input 2). Split Naukri no_of_jobs 60 → 30 each. Added cross-run dedup: Read Existing Jobs → Filter New Jobs Only (before scoring). hashJob now uses normalized apply link URL. Sheets matchingColumns updated to include Apply Link. Apify confirmed $5/month recurring credit → net $0/month for Apify. Monthly cost: ~$0.30/month. Expected raw: ~195 unique/run; after cross-run dedup: ~50-70 new jobs/run.
- **v7.0** (2026-03-09): Fixed email topScore=0 and sort order (Build Email Digest uses `j['Match Score'] || j.score`). JSearch num_pages 3→4. Naukri no_of_jobs 40→60. Hardcoded API keys (sk-proj- prefix fixed). Monthly cost: ~$0.57/month. Expected output: ~50-70 final jobs/run.
- **v6.1** (2026-03-09): Tuned job volume to ~60/run (40 JSearch + 20 Naukri). Fixed rapidApiKey trailing space. Fixed Apify timeout 120s→300s + SSL ignore. Fixed Merge node empty parameters. Monthly cost: ~$0.43/month.
- **v6.0** (2026-03-08): Added Naukri scraper via Apify as parallel job source. Architecture: Set Job Preferences forks into JSearch branch + Naukri branch; Merge Search Results (append mode) combines both before Parse Job Results. Parse Job Results now handles both JSearch (`data[]`) and Apify (top-level array) formats. Added apifyToken to Set Job Preferences. Node count: 21. Monthly cost: ~$0.17/month.
- **v5.0** (2026-03-08): Switched job search from SerpAPI to JSearch (RapidAPI) — now aggregates LinkedIn + Indeed + Glassdoor + Google Jobs, 2 pages per query (~120 raw → ~80 unique jobs vs ~39 before). Switched AI from Groq llama-3.3-70b (free) to OpenAI GPT-4o-mini (paid, ~$0.03/run). Full resume now sent (no substring limit). Batch interval reduced 15s→3s (paid tier). Retries disabled (`retryOnFail: false`) — fail fast is better than wasting tokens. Payload field renamed `groqPayload`→`aiPayload`, key renamed `groqApiKey`→`openAiApiKey`. Monthly cost: ~$0.11 for 4 runs. Cross-run deduplication via `appendOrUpdate` (matchingColumns: Job Title + Company). Status column preserved on re-runs. minScore raised to 75.
- **v4.0** (2026-03-08): Restructured to 18 nodes. n8n Code node task runner sandbox has no HTTP client — split each AI step into Prep Code node + HTTP Request node. Groq API now called via `api.groq.com/openai/v1/chat/completions` with Bearer auth header. Batch interval reduced 15s→5s (3x faster). Added PDF null char cleanup (`\u0000` artifacts). Switched to Bangalore-only 6-search strategy targeting 12 specific roles. Added SerpAPI geo-targeting (`gl=in`, `hl=en`). Fixed `json_validate_failed` 400 error by removing embedded JSON example from system prompt. Resume file renamed to `your-resume.pdf`.
- **v3.0** (2026-02-24): Switched to Groq Llama 3 70B (from Groq Llama 3.3). Added PDF resume reading (Read Resume PDF + Extract PDF Text nodes, 16 nodes total). Dual-location search: Bangalore onsite/hybrid + India remote. Expanded to 17 target roles. Lowered minScore from 70 to 50. Added HTTP batching (batchSize=1, batchInterval=20000ms) and retryOnFail (maxTries=3, waitBetweenTries=15000ms) on AI nodes for rate limit handling. Volume mount for resumes: `./JobSearchAutomation/resumes:/home/node/.n8n-files`.
- **v2.0** (2026-02-09): Replaced GPT-5 with Google Groq Llama 3.3 for job scoring and cover letter generation. Monthly cost reduced from ~$13.50 to $0. Added `groqApiKey` to Set Job Preferences. HTTP Request nodes call Groq API directly with `responseMimeType: "application/json"` for structured scoring output.
- **v1.0** (2026-02-09): Initial creation. 14 nodes, manual trigger. SerpAPI Google Jobs + GPT-5 scoring + cover letter generation + Google Sheets + Gmail digest. Target: Backend/.NET/React developer roles in India.

## Development History

This workflow was built across multiple sessions:
1. Phase 0: Folder reorganization -- split parent `/n8n/` into `YtShortsAutomation/` and `JobSearchAutomation/`
2. Phase 1: Built 14-node workflow via n8n MCP tools (`n8n_create_workflow` + `n8n_update_partial_workflow`)
3. Phase 2: Exported clean JSON, wrote all documentation (README, CLAUDE.md, setup-guide, workflow-reference, .env.example)
4. Phase 3: Migrated from Groq Llama 3.3 to Groq Llama 3 70B. Added PDF resume reading. Expanded search to dual-location and 17 roles.

Key issues encountered and resolved:
- **n8n Code node task runner has no HTTP client** -- `this.helpers.httpRequest` returns 400, `$helpers` is undefined, `fetch` is undefined. Solution: Code nodes build payloads, separate HTTP Request nodes make the calls.
- **Groq API auth** -- uses `Authorization: Bearer <key>` header, NOT query param. Endpoint: `https://api.groq.com/openai/v1/chat/completions` (OpenAI-compatible).
- **`json_validate_failed` 400 from Groq** -- embedding a JSON example `{"score": 0, ...}` in the system prompt confuses the model when `response_format: json_object` is set. Model outputs `{ free-form text ... json template }` which fails validation. Fix: use plain field descriptions, no JSON syntax in system prompt.
- **PDF null char artifacts** -- `extractFromFile` produces `\u0000` in place of `+`, `%`, `~`, `()` from certain PDF encodings. Fix: `rawText.replace(/\u0000/g, '')` in Set Job Preferences.
- **`n8n_update_partial_workflow` updateNode** -- use `nodeId` (not `name`) for nodes with special characters. Use `"updates": {...}` key, NOT `"properties": {...}`.
- **Groq TPD exhaustion** -- Groq free tier has 100K tokens/day limit. Multiple failed retry-heavy runs exhaust it. Fix: disabled retries (`retryOnFail: false`) and switched to OpenAI paid tier.
- **JSearch response format** -- `data[]` array (not `jobs_results[]` like SerpAPI). Fields: `employer_name`, `job_title`, `job_city`, `job_state`, `job_description`, `job_apply_link`, `job_publisher`, `job_employment_type`.
- **JSearch location targeting** -- embed location in query string (e.g., ".NET Developer Bangalore India") rather than separate `location` param. Use `country=in` for India filtering.
- **Code node `runOnceForEachItem`** -- Parse AI Score references `$('Prepare Score Request').item`, Attach Cover Letter references `$('Prepare Cover Letter').item`. Must match the immediately-preceding Prep node, not the Filter node.
- **Google Sheets column name mapping in Build Email Digest** -- map both `j['Job Title'] || j.title` for robustness.
- **n8n file access restrictions** -- Resumes must be volume-mounted to `/home/node/.n8n-files` via docker-compose.
- MCP `updateNode` requires `"updates"` key, not `"properties"`
- Background agents can't use Bash tool (auto-denied) -- use Write tool directly

## Commands
- Start n8n: `docker compose up -d` (from parent `/n8n/` directory)
- Rebuild: `docker compose build && docker compose up -d`
- Stop n8n: `docker compose down`
- Health check: `curl -s http://localhost:5678/healthz`
- Get workflow via MCP: `n8n_get_workflow` with ID `8vs888j57KeLaGtH`
- Validate workflow: `n8n_validate_workflow` with ID `8vs888j57KeLaGtH`
- List all workflows: `n8n_list_workflows`

# n8n Job Search Automation

## Instance Configuration
- **URL**: http://localhost:5678
- **Environment**: Local Docker container (shared n8n instance with YtShortsAutomation)
- **Focus**: Automated job search with AI matching and cover letter generation
- **n8n Workflow ID**: `r1rpLsVRxOrsVtC3` (deployed to running instance)
- **Workflow Name**: "Job Search Automation - AI Matching & Cover Letters"
- **GitHub Repo**: https://github.com/skylark248/N8nJobSearchAutomation.git

## Parent Directory Context

This project lives at `/Users/shivanshchoudhary/Downloads/n8n/JobSearchAutomation/`. The parent `/n8n/` directory contains:
- `YtShortsAutomation/` -- sibling project (YouTube Shorts, separate repo)
- n8n runtime data: `database.sqlite`, `binaryData/`, `config`, `.mcp.json`, `crash.journal`
- The n8n Docker instance mounts the parent `/n8n/` directory as `/home/node/.n8n`
- Resumes volume: `./JobSearchAutomation/resumes` is mounted to `/home/node/.n8n-files` inside the container
- Workflow JSON files are imported via the n8n UI, not read from the filesystem at runtime

## User Profile

- **Role**: Backend-heavy Full Stack Developer (.NET + React)
- **Target Roles** (17 total): Backend Developer, Senior Backend Developer, .NET Developer, Full Stack Developer, Senior Full Stack Developer, Software Engineer 2, SDE 2, Senior Software Engineer, C# Developer, ASP.NET Developer, Software Engineer, Backend Engineer, Application Developer, Software Developer, AI Platform Engineer, Backend Engineer AI, Gen AI Engineer
- **Location**: Bangalore onsite/hybrid + India remote (dual-location search)
- **Resume**: PDF file in `resumes/` folder, read automatically by the workflow via Read Resume PDF node

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

### Job Search Automation - AI Matching & Cover Letters
- **Status**: Inactive (manual trigger)
- **n8n Workflow ID**: `r1rpLsVRxOrsVtC3`
- **Nodes**: 16 | **Connections**: 15
- **Estimated Runtime**: 3-8 minutes per run (depending on job count and API rate limits)
- **Export File**: `exports/job-search-automation.json`
- **Documentation**: See `docs/workflow-reference.md` for full node-by-node breakdown
- **Setup Guide**: See `docs/setup-guide.md` for credential configuration

**Pipeline**: Read Resume PDF -> Extract PDF Text -> Set Job Preferences (dual-location: Bangalore onsite/hybrid + India remote) -> Search Google Jobs x2 (SerpAPI) -> Parse & deduplicate results -> Gemma 3 27B scores each job against resume (0-100) -> Filter top matches (score >= 50) -> Generate tailored cover letters -> Save to Google Sheets -> Send Gmail digest email

**Required Credentials** (configure in n8n before running):
1. **SerpAPI** - API key as query parameter for Google Jobs search (free tier: 100 searches/month)
2. **Google Gemini API** - free API key from https://aistudio.google.com/apikey for Gemma 3 27B (job scoring + cover letters). Free tier: 14K RPD.
3. **Google OAuth2** - for Google Sheets (saving results) and Gmail (digest email). Enable both Sheets API and Gmail API in Google Cloud Console. Redirect URI: `http://localhost:5678/rest/oauth2-credential/callback`

**Monthly Cost**: ~$0 (all free tiers) -- Gemma 3 27B free + SerpAPI free + Google APIs free

### Complete Node List (16 nodes)

| # | Node Name | Type | Version | ID | Purpose |
|---|---|---|---|---|---|
| 1 | Read Resume PDF | `n8n-nodes-base.readBinaryFile` | v1 | `readResumePdf` | Reads PDF from `/home/node/.n8n-files/GarimaResume.pdf` |
| 2 | Extract PDF Text | `n8n-nodes-base.extractFromFile` | v1.1 | `extractPdfText` | Extracts text from PDF binary (operation: pdf, joinPages: true) |
| 3 | Manual Trigger | `n8n-nodes-base.manualTrigger` | v1 | `trigger` | Click to run (replace with Schedule for daily) |
| 4 | Set Job Preferences | `n8n-nodes-base.code` | v2 | `setPreferences` | Central config: reads resume from PDF, dual-location search, 17 roles, minScore 50 |
| 5 | Search Google Jobs (SerpAPI) | `n8n-nodes-base.httpRequest` | v4.2 | `searchJobs` | GET serpapi.com/search.json with engine=google_jobs, chips=date_posted:week |
| 6 | Parse Job Results | `n8n-nodes-base.code` | v2 | `parseJobs` | Processes $input.all() for multiple search responses, normalizes fields, generates jobId hash |
| 7 | Filter Duplicates | `n8n-nodes-base.code` | v2 | `filterDuplicates` | Emits individual items from aggregated jobs array |
| 8 | Score & Match (Gemini) | `n8n-nodes-base.httpRequest` | v4.2 | `scoreMatch` | POST Gemma 3 27B API, score 0-100 + recommendation. Batching: batchSize=1, batchInterval=20000ms. retryOnFail: maxTries=3, waitBetweenTries=15000ms |
| 9 | Parse AI Score | `n8n-nodes-base.code` | v2 | `parseScore` | runOnceForEachItem mode, uses $input.item. Parse Gemma JSON response, merge with job data from Filter Duplicates |
| 10 | Filter Top Matches | `n8n-nodes-base.code` | v2 | `filterMatches` | runOnceForAllItems mode, uses $input.all(). Filters score >= minScore (50) |
| 11 | Generate Cover Letter (Gemini) | `n8n-nodes-base.httpRequest` | v4.2 | `generateCoverLetter` | POST Gemma 3 27B API, cover letter under 300 words. Batching: batchSize=1, batchInterval=20000ms. retryOnFail: maxTries=3, waitBetweenTries=15000ms |
| 12 | Attach Cover Letter | `n8n-nodes-base.code` | v2 | `addCoverLetter` | runOnceForEachItem mode, uses $input.item. Merges cover letter text with job data from Filter Top Matches |
| 13 | Save to Google Sheets | `n8n-nodes-base.googleSheets` | v4.5 | `saveToSheet` | Append row with 12 columns to Job Tracker sheet |
| 14 | Build Email Digest | `n8n-nodes-base.code` | v2 | `buildEmail` | HTML email with gradient header, stats, job table, cover letters. Maps both Google Sheets column names and camelCase field names |
| 15 | Send Gmail Digest | `n8n-nodes-base.gmail` | v2.1 | `sendDigest` | Send HTML digest with emailType: "html" |
| 16 | Success Summary | `n8n-nodes-base.code` | v2 | `successOutput` | Return matchCount, topScore, emailSent, timestamp |

### Connection Map (15 connections)

```
Manual Trigger           -> Read Resume PDF             (main)
Read Resume PDF          -> Extract PDF Text            (main)
Extract PDF Text         -> Set Job Preferences         (main)
Set Job Preferences      -> Search Google Jobs          (main)
Search Google Jobs       -> Parse Job Results           (main)
Parse Job Results        -> Filter Duplicates           (main)
Filter Duplicates        -> Score & Match (Gemini)      (main)
Score & Match (Gemini)   -> Parse AI Score              (main)
Parse AI Score           -> Filter Top Matches          (main)
Filter Top Matches       -> Generate Cover Letter       (main)
Generate Cover Letter    -> Attach Cover Letter         (main)
Attach Cover Letter      -> Save to Google Sheets       (main)
Save to Google Sheets    -> Build Email Digest          (main)
Build Email Digest       -> Send Gmail Digest           (main)
Send Gmail Digest        -> Success Summary             (main)
```

### Key Node Implementation Details

**Read Resume PDF** (readBinaryFile node):
- Reads from `/home/node/.n8n-files/GarimaResume.pdf` (volume-mounted from `./JobSearchAutomation/resumes/`)
- n8n restricts file access to `/home/node/.n8n` and `/home/node/.n8n-files` via `N8N_RESTRICT_FILE_ACCESS_TO`
- To change resume: place a new PDF in `resumes/` and update the `filePath` in this node

**Extract PDF Text** (extractFromFile node):
- Operation: `pdf`
- Options: `joinPages: true` (concatenates all pages into single text field)
- Outputs `{ text: "..." }` with the full resume text

**Set Job Preferences** (Code node):
- Reads resume text from `$input.first().json.text` (from Extract PDF Text)
- Validates resume is not empty or too short (< 50 chars)
- Returns **two items** for dual-location search:
  - Item 1: Bangalore onsite/hybrid roles (`location: "Bangalore, Karnataka, India"`)
  - Item 2: India remote roles (`location: "India"`)
- Contains `searchQuery` with 17 target roles joined by OR
- `minScore`: 50
- API keys for SerpAPI and Gemini stored here
- `yourEmail` and `spreadsheetId`: configured inline

**Search Google Jobs** (HTTP Request):
- SerpAPI endpoint: `https://serpapi.com/search.json`
- Query params: `engine=google_jobs`, `q={{ searchQuery }}`, `location={{ location }}`, `chips=date_posted:week`, `api_key={{ serpApiKey }}`
- Runs once per input item (2 searches: Bangalore + India remote)
- Timeout: 15000ms

**Parse Job Results** (Code node):
- Processes `$input.all()` to handle multiple search responses (Bangalore + India)
- Extracts `jobs_results` from each SerpAPI response
- Generates `jobId` hash from `company+title` (lowercased, trimmed)
- Extracts `applyLink` from `apply_options[0].link` or `share_link`
- Truncates `description` to 3000 chars for AI context window
- Returns single item with `{ jobs: [...], totalFound: N }`

**Filter Duplicates** (Code node):
- Takes aggregated jobs array and emits individual items: `jobs.map(job => ({ json: job }))`
- One output item per job for per-job processing downstream

**Score & Match (Gemini)** (HTTP Request):
- POST `https://generativelanguage.googleapis.com/v1beta/models/gemma-3-27b-it:generateContent`
- API key passed as query parameter from Set Job Preferences
- **No systemInstruction** -- Gemma 3 27B does not support `systemInstruction` or `responseMimeType`. System prompt is merged into the user message.
- User message includes: system instructions + resume text + job details + JSON output format
- Temperature: 0.7
- **HTTP batching**: batchSize=1, batchInterval=20000ms (one request every 20 seconds to avoid rate limits)
- **retryOnFail**: maxTries=3, waitBetweenTries=15000ms
- Timeout: 30000ms

**Parse AI Score** (Code node):
- **Mode**: `runOnceForEachItem` -- processes each AI response individually
- Uses `$input.item.json` to access the current Gemini response
- References original job data via `$('Filter Duplicates').item` (same-index item)
- Parses JSON with try/catch + fallback regex (`/\{[\s\S]*\}/`)
- Merges parsed fields (score, recommendation, matchReason, missingSkills, keyMatchingSkills, experienceFit) with original job data

**Filter Top Matches** (Code node):
- **Mode**: `runOnceForAllItems` (default) -- processes all items at once
- Uses `$input.all()` to get all scored items
- Filters: `items.filter(item => item.json.score >= minScore)`
- Throws error if no jobs pass the threshold (includes the top score in error message)
- minScore default: 50

**Generate Cover Letter (Gemini)** (HTTP Request):
- POST `https://generativelanguage.googleapis.com/v1beta/models/gemma-3-27b-it:generateContent`
- **No systemInstruction** -- system prompt merged into user message (same Gemma limitation)
- Instructs: professional cover letter under 300 words, no markdown, no JSON
- **HTTP batching**: batchSize=1, batchInterval=20000ms
- **retryOnFail**: maxTries=3, waitBetweenTries=15000ms

**Attach Cover Letter** (Code node):
- **Mode**: `runOnceForEachItem` -- processes each cover letter response individually
- Uses `$input.item.json` to access the current Gemini response
- References job data via `$('Filter Top Matches').item` (same-index item)
- Extracts cover letter from `candidates[0].content.parts[0].text`
- Adds `processedDate` field

**Save to Google Sheets** (v4.5):
- Operation: `append` with `defineBelow` mapping mode
- Sheet name: `Job Tracker`
- 12 columns mapped with Google Sheets column names: Date, Job Title, Company, Location, Match Score, Recommendation, Match Reason, Missing Skills, Apply Link, Cover Letter, Status ("Not Applied"), Applied Date
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
| B | Job Title | SerpAPI result |
| C | Company | SerpAPI result |
| D | Location | SerpAPI result |
| E | Match Score | Gemma 3 27B scoring (0-100) |
| F | Recommendation | Gemma 3 27B (strong-apply/apply/maybe/skip) |
| G | Match Reason | Gemma 3 27B (2-3 sentences) |
| H | Missing Skills | Gemma 3 27B (comma-separated) |
| I | Apply Link | SerpAPI result |
| J | Cover Letter | Gemma 3 27B generation (under 300 words) |
| K | Status | Default: "Not Applied" |
| L | Applied Date | Empty (user fills manually) |

### Placeholder Values to Configure

These values in the exported JSON need to be replaced before running:

| Placeholder | Where | Replace With |
|---|---|---|
| `YOUR_SERPAPI_KEY` | Set Job Preferences node, `serpApiKey` field | Your SerpAPI API key from serpapi.com/dashboard |
| `YOUR_GEMINI_API_KEY` | Set Job Preferences node, `geminiApiKey` field | Your Gemini API key from https://aistudio.google.com/apikey |
| `YOUR_EMAIL@gmail.com` | Set Job Preferences node, `yourEmail` field | Your Gmail address |
| `YOUR_GOOGLE_SHEET_ID` | Set Job Preferences node + Save to Google Sheets node | Spreadsheet ID from Google Sheets URL |
| `GarimaResume.pdf` | Read Resume PDF node, `filePath` field | Your resume PDF filename in `resumes/` folder |

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
├── .env.example              # Template for secrets (copy to .env)
├── .gitignore                # Excludes sensitive files from git
├── .gitmodules               # Git submodule references (n8n-mcp, n8n-skills)
│
├── resumes/                  # Resume PDFs (volume-mounted to /home/node/.n8n-files)
│   ├── GarimaResume.pdf      # Current resume
│   ├── README.txt            # Instructions for resume placement
│   └── .gitkeep              # Keeps folder in git
│
├── docs/                     # All documentation
│   ├── setup-guide.md        # Step-by-step credential setup (SerpAPI, Gemini, Google OAuth2)
│   └── workflow-reference.md # Complete 16-node workflow documentation
│
├── exports/                  # Importable workflow JSON files
│   └── job-search-automation.json  # Clean export (no credentials, no metadata)
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
- **Job search**: Read Resume PDF -> Extract Text -> HTTP Request (SerpAPI) -> Code (parse) -> Gemma 3 27B (score/generate) -> Google Sheets (save) -> Gmail (notify)

### n8n Expression Syntax Quick Reference
- Reference current item: `{{ $json.fieldName }}`
- Reference another node: `{{ $('Node Name').first().json.fieldName }}`
- Reference same-index item from another node: `{{ $('Node Name').item.json.fieldName }}`
- Current date: `{{ $now.format('yyyy-MM-dd') }}`
- n8n expressions start with `=` when used in node parameters: `={{ $json.field }}`

## Cost Estimate

| Service | Per Run | Monthly (30 runs) |
|---|---|---|
| SerpAPI (2 searches/run) | $0.00 | $0.00 (free tier: 100/mo) |
| Gemma 3 27B scoring (~20 jobs) | $0.00 | $0.00 (free tier: 14K RPD) |
| Gemma 3 27B cover letters (~5-10 jobs) | $0.00 | $0.00 (free tier) |
| Google Sheets + Gmail | $0.00 | $0.00 |
| **Total** | **$0.00** | **$0.00** |

## Version History

- **v3.0** (2026-02-24): Switched to Gemma 3 27B (from Gemini 2.0 Flash). Added PDF resume reading (Read Resume PDF + Extract PDF Text nodes, 16 nodes total). Dual-location search: Bangalore onsite/hybrid + India remote. Expanded to 17 target roles. Lowered minScore from 70 to 50. Added HTTP batching (batchSize=1, batchInterval=20000ms) and retryOnFail (maxTries=3, waitBetweenTries=15000ms) on AI nodes for rate limit handling. Volume mount for resumes: `./JobSearchAutomation/resumes:/home/node/.n8n-files`.
- **v2.0** (2026-02-09): Replaced GPT-5 with Google Gemini 2.0 Flash for job scoring and cover letter generation. Monthly cost reduced from ~$13.50 to $0. Added `geminiApiKey` to Set Job Preferences. HTTP Request nodes call Gemini API directly with `responseMimeType: "application/json"` for structured scoring output.
- **v1.0** (2026-02-09): Initial creation. 14 nodes, manual trigger. SerpAPI Google Jobs + GPT-5 scoring + cover letter generation + Google Sheets + Gmail digest. Target: Backend/.NET/React developer roles in India.

## Development History

This workflow was built across multiple sessions:
1. Phase 0: Folder reorganization -- split parent `/n8n/` into `YtShortsAutomation/` and `JobSearchAutomation/`
2. Phase 1: Built 14-node workflow via n8n MCP tools (`n8n_create_workflow` + `n8n_update_partial_workflow`)
3. Phase 2: Exported clean JSON, wrote all documentation (README, CLAUDE.md, setup-guide, workflow-reference, .env.example)
4. Phase 3: Migrated from Gemini 2.0 Flash to Gemma 3 27B. Added PDF resume reading. Expanded search to dual-location and 17 roles.

Key issues encountered and resolved:
- **Gemma 3 27B does not support `systemInstruction` or `responseMimeType`** -- system prompt must be merged into the user message as a single text block. Cannot force JSON output mode; instead instruct "respond with ONLY valid JSON" in the prompt.
- **Code node `runOnceForEachItem` mode** -- Parse AI Score and Attach Cover Letter must use this mode with `$input.item` to access the corresponding AI response for each job. Without it, they only see the first item.
- **Google Sheets column name mapping in Build Email Digest** -- Google Sheets returns data with column header names as keys (e.g., `"Job Title"` not `"title"`). The email builder must map both `j['Job Title'] || j.title` for robustness.
- **n8n file access restrictions** -- `N8N_RESTRICT_FILE_ACCESS_TO` limits Read Binary File to whitelisted paths. Resumes must be volume-mounted to `/home/node/.n8n-files` via docker-compose.
- Gmail node needed explicit `operation: "send"` -- validation caught this
- MCP `updateNode` requires `"updates"` key, not `"properties"`
- Background agents can't use Bash tool (auto-denied) -- use Write tool directly

## Commands
- Start n8n: `docker compose up -d` (from parent `/n8n/` directory)
- Rebuild: `docker compose build && docker compose up -d`
- Stop n8n: `docker compose down`
- Health check: `curl -s http://localhost:5678/healthz`
- Get workflow via MCP: `n8n_get_workflow` with ID `r1rpLsVRxOrsVtC3`
- Validate workflow: `n8n_validate_workflow` with ID `r1rpLsVRxOrsVtC3`
- List all workflows: `n8n_list_workflows`

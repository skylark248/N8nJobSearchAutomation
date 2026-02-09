# n8n Job Search Automation

## Instance Configuration
- **URL**: http://localhost:5678
- **Environment**: Local Docker container (shared n8n instance with YtShortsAutomation)
- **Focus**: Automated job search with AI matching and cover letter generation
- **n8n Workflow ID**: `r1rpLsVRxOrsVtC3` (deployed to running instance)
- **Workflow Name**: "Job Search Automation - AI Matching & Cover Letters"
- **GitHub Repo**: Not yet pushed (user will `git init` and push separately)

## Parent Directory Context

This project lives at `/Users/shivanshchoudhary/Downloads/n8n/JobSearchAutomation/`. The parent `/n8n/` directory contains:
- `YtShortsAutomation/` -- sibling project (YouTube Shorts, separate repo)
- n8n runtime data: `database.sqlite`, `binaryData/`, `config`, `.mcp.json`, `crash.journal`
- The n8n Docker instance mounts the parent `/n8n/` directory as `/home/node/.n8n`
- Workflow JSON files are imported via the n8n UI, not read from the filesystem at runtime

## User Profile

- **Role**: Backend-heavy Full Stack Developer (.NET + React)
- **Target Roles**: Software Engineer 2, SDE 2, Backend Developer, Senior Backend Developer, .NET Developer, Senior .NET Developer, Full Stack Developer, Senior Full Stack Developer, Senior Software Engineer
- **Location**: India
- **Resume**: User has a PDF resume; plain text is pasted into the Set Job Preferences node

## Quick Start

```bash
# 1. Start n8n (from parent /n8n/ directory)
docker run --rm --name n8n -p 5678:5678 -v $(pwd):/home/node/.n8n n8nio/n8n

# 2. Copy env file and fill in your API keys
cp .env.example .env

# 3. Import the workflow
# Open http://localhost:5678 -> Import from file -> select exports/job-search-automation.json

# 4. Configure credentials (see docs/setup-guide.md)

# 5. Create Google Sheet with 12-column headers (A-L), paste resume text

# 6. Test run
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
- **Nodes**: 14 | **Connections**: 13
- **Estimated Runtime**: 2-5 minutes per run
- **Export File**: `exports/job-search-automation.json`
- **Documentation**: See `docs/workflow-reference.md` for full node-by-node breakdown
- **Setup Guide**: See `docs/setup-guide.md` for credential configuration

**Pipeline**: Search Google Jobs (SerpAPI) -> Parse results -> Filter duplicates -> GPT-5 scores each job against resume (0-100) -> Filter top matches (score >= 70) -> Generate tailored cover letters -> Save to Google Sheets -> Send Gmail digest email

**Required Credentials** (configure in n8n before running):
1. **SerpAPI** - API key as query parameter for Google Jobs search (free tier: 100 searches/month)
2. **OpenAI API** - for GPT-5 job scoring and cover letter generation
3. **Google OAuth2** - for Google Sheets (saving results) and Gmail (digest email). Enable both Sheets API and Gmail API in Google Cloud Console. Redirect URI: `http://localhost:5678/rest/oauth2-credential/callback`

**Monthly Cost**: ~$13.50 (OpenAI API) + $0 (SerpAPI free tier) + $0 (Google APIs)

### Complete Node List (14 nodes)

| # | Node Name | Type | Version | ID | Purpose |
|---|---|---|---|---|---|
| 1 | Manual Trigger | `n8n-nodes-base.manualTrigger` | v1 | `trigger` | Click to run (replace with Schedule for daily) |
| 2 | Set Job Preferences | `n8n-nodes-base.code` | v2 | `setPreferences` | Central config: job titles, location, resume text, minScore |
| 3 | Search Google Jobs (SerpAPI) | `n8n-nodes-base.httpRequest` | v4.2 | `searchJobs` | GET serpapi.com/search.json with engine=google_jobs |
| 4 | Parse Job Results | `n8n-nodes-base.code` | v2 | `parseJobs` | Extract jobs_results, normalize fields, generate jobId hash |
| 5 | Filter Duplicates | `n8n-nodes-base.code` | v2 | `filterDuplicates` | Dedup by company+title, output individual items |
| 6 | Score & Match (GPT-5) | `n8n-nodes-base.openAi` | v1 | `scoreMatch` | Score 0-100 + recommendation + matchReason + missingSkills |
| 7 | Parse AI Score | `n8n-nodes-base.code` | v2 | `parseScore` | Parse GPT-5 JSON with fallback regex, merge with job data |
| 8 | Filter Top Matches | `n8n-nodes-base.code` | v2 | `filterMatches` | Keep only score >= minScore (default 70) |
| 9 | Generate Cover Letter (GPT-5) | `n8n-nodes-base.openAi` | v1 | `generateCoverLetter` | Tailored cover letter under 300 words per job |
| 10 | Attach Cover Letter | `n8n-nodes-base.code` | v2 | `attachCoverLetter` | Merge cover letter text with job data |
| 11 | Save to Google Sheets | `n8n-nodes-base.googleSheets` | v4.5 | `saveToSheet` | Append row with 12 columns to Job Tracker sheet |
| 12 | Build Email Digest | `n8n-nodes-base.code` | v2 | `buildEmail` | HTML email with gradient header, stats, job table, cover letters |
| 13 | Send Gmail Digest | `n8n-nodes-base.gmail` | v2.1 | `sendDigest` | Send HTML digest with operation: "send", emailType: "html" |
| 14 | Success Summary | `n8n-nodes-base.code` | v2 | `successOutput` | Return matchCount, topScore, emailSent, timestamp |

### Connection Map

```
Manual Trigger           -> Set Job Preferences       (main)
Set Job Preferences      -> Search Google Jobs         (main)
Search Google Jobs       -> Parse Job Results          (main)
Parse Job Results        -> Filter Duplicates          (main)
Filter Duplicates        -> Score & Match (GPT-5)      (main)
Score & Match (GPT-5)    -> Parse AI Score             (main)
Parse AI Score           -> Filter Top Matches         (main)
Filter Top Matches       -> Generate Cover Letter      (main)
Generate Cover Letter    -> Attach Cover Letter        (main)
Attach Cover Letter      -> Save to Google Sheets      (main)
Save to Google Sheets    -> Build Email Digest         (main)
Build Email Digest       -> Send Gmail Digest          (main)
Send Gmail Digest        -> Success Summary            (main)
```

### Key Node Implementation Details

**Set Job Preferences** (Code node):
- Contains `searchQuery` with all target roles joined by OR
- `location`: "India"
- `minScore`: 70
- `resumeText`: Placeholder for user's resume
- `yourEmail` and `spreadsheetId`: Placeholders to configure

**Search Google Jobs** (HTTP Request):
- SerpAPI endpoint: `https://serpapi.com/search.json`
- Query params: `engine=google_jobs`, `q={{ searchQuery }}`, `location={{ location }}`, `api_key=YOUR_SERPAPI_KEY`
- Timeout: 15000ms

**Parse Job Results** (Code node):
- Extracts from `$input.first().json.jobs_results`
- Generates `jobId` hash from `company+title` (lowercased, trimmed)
- Extracts `applyLink` from `apply_options[0].link` or `share_link`
- Truncates `description` to 3000 chars for GPT-5 context window

**Score & Match** (OpenAI node):
- Model: GPT-5
- System prompt: Job matching expert, returns JSON with score/matchReason/missingSkills/keyMatchingSkills/experienceFit/recommendation
- User message references resume via `$('Set Job Preferences').first().json.resumeText`

**Parse AI Score** (Code node):
- Parses GPT-5 response with try/catch
- Fallback regex: `/\{[\s\S]*\}/` to extract JSON from markdown-wrapped responses
- Merges parsed fields with original job data from `$('Filter Duplicates').item`

**Filter Top Matches** (Code node):
- Returns item if `score >= minScore`, returns empty `[]` otherwise
- Jobs below threshold are silently dropped (saves API costs on cover letters)

**Save to Google Sheets** (v4.5):
- Operation: `appendOrUpdate` with `defineBelow` mapping mode
- 12 columns: Date, Job Title, Company, Location, Match Score, Recommendation, Match Reason, Missing Skills, Apply Link, Cover Letter, Status ("Not Applied"), Applied Date
- Uses `__rl: true` resource locator format for document/sheet references

**Build Email Digest** (Code node):
- HTML with gradient header (#667eea to #764ba2)
- Stats section showing total matches and top score
- Job table with color-coded scores: green (>=85), yellow (>=70), red (<70)
- Apply buttons as HTML links
- Expandable cover letters using `<details>` HTML tags

**Send Gmail Digest** (v2.1):
- `operation: "send"` (REQUIRED -- missing this causes validation failure)
- `emailType: "html"`
- Subject: `Job Search Update: X new matches found (Top: Y/100)`

### Google Sheet Schema (12 columns)

| Column | Header | Source |
|---|---|---|
| A | Date | `new Date().toISOString().split('T')[0]` |
| B | Job Title | SerpAPI result |
| C | Company | SerpAPI result |
| D | Location | SerpAPI result |
| E | Match Score | GPT-5 scoring (0-100) |
| F | Recommendation | GPT-5 (strong-apply/apply/maybe/skip) |
| G | Match Reason | GPT-5 (2-3 sentences) |
| H | Missing Skills | GPT-5 (comma-separated) |
| I | Apply Link | SerpAPI result |
| J | Cover Letter | GPT-5 generation (250-350 words) |
| K | Status | Default: "Not Applied" |
| L | Applied Date | Empty (user fills manually) |

### Placeholder Values to Configure

These values in the exported JSON need to be replaced before running:

| Placeholder | Where | Replace With |
|---|---|---|
| `YOUR_SERPAPI_KEY` | Search Google Jobs node, query param `api_key` | Your SerpAPI API key from serpapi.com/dashboard |
| `YOUR_EMAIL@gmail.com` | Set Job Preferences node, `yourEmail` field | Your Gmail address |
| `YOUR_GOOGLE_SHEET_ID` | Set Job Preferences node + Save to Google Sheets node | Spreadsheet ID from Google Sheets URL |
| `PASTE YOUR RESUME TEXT HERE` | Set Job Preferences node, `resumeText` field | Your full resume as plain text |

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
├── docs/                     # All documentation
│   ├── setup-guide.md        # Step-by-step credential setup (SerpAPI, OpenAI, Google OAuth2)
│   └── workflow-reference.md # Complete 14-node workflow documentation
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
- **Job search**: HTTP Request (SerpAPI) -> Code (parse) -> OpenAI (score/generate) -> Google Sheets (save) -> Gmail (notify)

### n8n Expression Syntax Quick Reference
- Reference current item: `{{ $json.fieldName }}`
- Reference another node: `{{ $('Node Name').first().json.fieldName }}`
- Reference same-index item from another node: `{{ $('Node Name').item.json.fieldName }}`
- Current date: `{{ $now.format('yyyy-MM-dd') }}`
- n8n expressions start with `=` when used in node parameters: `={{ $json.field }}`

## Cost Estimate

| Service | Per Run | Monthly (30 runs) |
|---|---|---|
| SerpAPI (1 search/run) | $0.00 | $0.00 (free tier: 100/mo) |
| GPT-5 scoring (~20 jobs) | ~$0.30 | ~$9.00 |
| GPT-5 cover letters (~5 jobs) | ~$0.15 | ~$4.50 |
| Google Sheets + Gmail | $0.00 | $0.00 |
| **Total** | **~$0.45** | **~$13.50** |

## Version History

- **v1.0** (2026-02-09): Initial creation. 14 nodes, manual trigger. SerpAPI Google Jobs + GPT-5 scoring + cover letter generation + Google Sheets + Gmail digest. Target: Backend/.NET/React developer roles in India.

## Development History

This workflow was built in a single session:
1. Phase 0: Folder reorganization -- split parent `/n8n/` into `YtShortsAutomation/` and `JobSearchAutomation/`
2. Phase 1: Built 14-node workflow via n8n MCP tools (`n8n_create_workflow` + `n8n_update_partial_workflow`)
3. Phase 2: Exported clean JSON, wrote all documentation (README, CLAUDE.md, setup-guide, workflow-reference, .env.example)

Key issues encountered and resolved:
- Gmail node needed explicit `operation: "send"` -- validation caught this
- MCP `updateNode` requires `"updates"` key, not `"properties"`
- Background agents can't use Bash tool (auto-denied) -- use Write tool directly

## Commands
- Start n8n: `docker run --rm --name n8n -p 5678:5678 -v $(pwd):/home/node/.n8n n8nio/n8n`
- Stop n8n: `docker stop n8n`
- Health check: `curl -s http://localhost:5678/healthz`
- Get workflow via MCP: `n8n_get_workflow` with ID `r1rpLsVRxOrsVtC3`
- Validate workflow: `n8n_validate_workflow` with ID `r1rpLsVRxOrsVtC3`
- List all workflows: `n8n_list_workflows`

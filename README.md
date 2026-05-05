# n8n Job Search Automation

Automated pipeline that searches for .NET/React developer jobs in Bangalore, scores each against your resume with GPT-4o-mini, generates tailored cover letters, saves results to Google Sheets, and sends a Gmail digest — fully hands-free.

**Proven results:** 211 unique raw jobs → 71 final matches in first run (score ≥ 75). With v9.4 scoring fix, top matches now score 90-100.

---

## How It Works

```
Read Resume PDF
    |
    v
Extract PDF Text
    |
    v
Set Job Preferences  (7 queries, minScore=75, API keys)
    |           |           |
    v           v           v
Search JSearch  Naukri .NET  Naukri C#
(7 searches,    (Apify,      (Apify,
 4 pages each)   ~20-30 jobs) ~30-40 jobs)
    |           |           |
    v           v           v
    Merge Search Results (append)
    |
    v
Parse Job Results   (handles JSearch + Naukri dual formats)
    |
    v
Filter Duplicates   (~210 unique jobs)
    |
    v
Aggregate Jobs      (collapses N items → 1, prevents rate limit)
    |           |
    v           v
Read Existing Jobs  (Google Sheets, runs exactly once)
    |
    v
Sync Dedup Inputs   (ensures pipeline runs even on empty sheet)
    |
    v
Filter New Jobs Only  (cross-run dedup: skip already-seen jobs)
    |
    v
Filter Stack & Quality  (stack blacklist, exp range, location, company cap)
    |
    v
Prepare Score Request
    |
    v
Score OpenAI API    (GPT-4o-mini, batchSize=1, 3s interval)
    |
    v
Parse AI Score
    |
    v
Filter Top Matches  (score >= 75, ~30-50 jobs)
    |
    v
Prepare Cover Letter
    |
    v
Cover Letter OpenAI API
    |
    v
Attach Cover Letter
    |
    v
Save to Google Sheets  (appendOrUpdate, dedup on Title+Company+Link)
    |
    v
Build Email Digest  (HTML, sorted by score, expandable cover letters)
    |
    v
Send Gmail Digest
    |
    v
Success Summary
```

**28 nodes, 27 connections. Runtime: ~21 minutes per run.**

---

## Prerequisites

- **Docker** installed ([Get Docker](https://docs.docker.com/get-docker/))
- **Git** installed
- **RapidAPI account** — subscribe to JSearch Basic (free: 200 req/month) at [rapidapi.com](https://rapidapi.com)
- **OpenAI account** — pay-as-you-go API key from [platform.openai.com](https://platform.openai.com/api-keys) (~$0.075/run)
- **Apify account** — free $5/month credit from [apify.com](https://apify.com) (covers ~60+ runs)
- **Google account** with Sheets and Gmail access
- **Your resume** as `resumes/your-resume.pdf`

---

## Quick Start

### Step 1: Clone the Repository

```bash
git clone --recurse-submodules https://github.com/skylark248/N8nJobSearchAutomation.git
cd N8nJobSearchAutomation
```

> Already cloned without `--recurse-submodules`?
> ```bash
> git submodule init && git submodule update
> ```

### Step 2: Start n8n

This workflow lives inside a parent `n8n/` directory that also holds n8n's Docker runtime:

```
n8n/
├── docker-compose.yml
├── database.sqlite
├── N8nJobSearchAutomation/   ← this repo
│   └── resumes/your-resume.pdf
└── YtShortsAutomation/       ← optional sibling project
```

```bash
# From the parent n8n/ directory (where docker-compose.yml is)
docker compose up -d
```

Open [http://localhost:5678](http://localhost:5678) in your browser.

> **Key docker-compose.yml settings required:**
> ```yaml
> environment:
>   N8N_RUNNERS_TASK_TIMEOUT: 900   # 15 min — Naukri actor takes 4-5 min
>   N8N_RESTRICT_FILE_ACCESS_TO: /home/node/.n8n;/home/node/.n8n-files
> volumes:
>   - ./N8nJobSearchAutomation/resumes:/home/node/.n8n-files
> ```

### Step 3: Import the Workflow

**Option A — via n8n CLI (recommended, preserves node IDs):**

```bash
# Copy the JSON into the container and import
docker cp exports/job-search-automation.json n8n:/tmp/job-search-automation.json

# Add the required workflow ID then import
docker exec n8n node -e "
const fs = require('fs');
const wf = JSON.parse(fs.readFileSync('/tmp/job-search-automation.json', 'utf8'));
wf.id = '8vs888j57KeLaGtH';
fs.writeFileSync('/tmp/job-search-automation.json', JSON.stringify(wf));
"
docker exec n8n n8n import:workflow --input=/tmp/job-search-automation.json
```

**Option B — via n8n UI:**
1. In n8n → click **"Add workflow"** → **"Import from file"**
2. Select `exports/job-search-automation.json`

### Step 4: Configure Credentials

Open the **Set Job Preferences** node and fill in these 2 fields (API keys are now read from `keys.json`):

| Field | Value |
|---|---|
| `yourEmail` | Your Gmail address (digest is sent here) |
| `spreadsheetId` | Google Sheet ID from URL: `/d/SPREADSHEET_ID/edit` |

Fill in `keys.json` in the repo root:

```json
{
  "rapidApiKey": "YOUR_RAPIDAPI_KEY",
  "openAiApiKey": "sk-proj-...",
  "apifyToken": "apify_api_..."
}
```

Then configure Google OAuth2 (see [docs/setup-guide.md](docs/setup-guide.md)):
- **Google Sheets OAuth2** → link to "Save to Google Sheets" and "Read Existing Jobs" nodes
- **Gmail OAuth2** → link to "Send Gmail Digest" node

### Step 5: Place Your Resume

```bash
cp /path/to/your-resume.pdf resumes/your-resume.pdf
```

The Docker volume mount makes it available at `/home/node/.n8n-files/your-resume.pdf` inside the container.

### Step 6: Create Google Sheet

1. Create a new Google Sheet, rename to **Job Tracker**
2. Add these headers in row 1 (A–L):

| A | B | C | D | E | F | G | H | I | J | K | L |
|---|---|---|---|---|---|---|---|---|---|---|---|
| Date | Job Title | Company | Location | Match Score | Recommendation | Match Reason | Missing Skills | Apply Link | Cover Letter | Status | Applied Date |

3. Copy the spreadsheet ID from the URL and paste into `spreadsheetId` in Set Job Preferences.

### Step 7: Test Run

Click **"Test Workflow"** (play button). Wait ~21 minutes. Check your Google Sheet and Gmail inbox.

---

## What to Expect

### First Run (empty sheet)
All discovered jobs are new. Based on execution #70:

| Stage | Count |
|---|---|
| Raw unique jobs found | ~211 |
| After cross-run dedup | ~211 (all new — first run) |
| Final matches (score ≥ 75) | ~30-70 |
| Cover letters generated | Same as final matches |
| Rows saved to Google Sheets | Same as final matches |

### Subsequent Runs
Cross-run deduplication skips jobs already in the sheet. Expect 20-60 genuinely new jobs per run depending on weekly listing churn.

### Score Distribution
With the v9.4 explicit arithmetic scoring prompt:
- Score 90-100: Strong apply — .NET + domain boost (Fintech/Azure/microservices) + strong match
- Score 82-89: Apply — strong .NET backend, some boosts or no penalties
- Score 75-81: Apply — good .NET match, minor gaps
- Score below 75: Filtered out at current minScore threshold

---

## Project Structure

```
.
├── README.md
├── CLAUDE.md                 # Claude Code AI config (full technical reference)
├── keys.json                 # API keys — fill in, never committed (gitignored)
├── .env.example              # Template for environment variables
├── .gitignore
│
├── docs/
│   ├── setup-guide.md        # Step-by-step credential setup
│   ├── workflow-reference.md # Complete 28-node documentation
│   └── CHANGELOG.md          # Full version history v1.0 → current
│
├── exports/
│   └── job-search-automation.json  # Importable n8n workflow (no credentials)
│
├── resumes/
│   ├── your-resume.pdf       # Place your resume here (gitignored)
│   ├── README.txt
│   └── .gitkeep
│
├── n8n-mcp/                  # MCP server for Claude Code (git submodule)
└── n8n-skills/               # Claude Code skills (git submodule)
```

---

## Cost Breakdown

| Service | Per Run | Monthly (4 runs) |
|---|---|---|
| JSearch Basic (RapidAPI) | $0.00 | $0.00 (free: 200 req/mo, uses 28) |
| Apify Naukri scraper (2 searches, pay-per-event) | ~$0.38 | ~$1.52 (within $5/mo free credit) |
| OpenAI GPT-4o-mini scoring (~150 jobs × ~5500 tokens) | ~$0.041 | ~$0.16 |
| OpenAI GPT-4o-mini cover letters (~30 jobs × ~5500 tokens) | ~$0.008 | ~$0.03 |
| Google Sheets + Gmail | $0.00 | $0.00 |
| **Total** | **~$0.43** | **~$1.73** (within Apify $5/mo credit — net ~$0.19) |

---

## Target Roles

Backend-heavy Full Stack Developer (.NET + React), ~5 years experience, **Bangalore only** (no India-wide remote):

- SDE 2 / Software Development Engineer 2
- Software Developer / Software Developer 2
- Software Engineer 2 / Senior Software Engineer
- .NET Developer / Senior .NET Developer
- Backend Engineer / Senior Backend Engineer
- Full Stack Engineer / Senior Full Stack Engineer

### JSearch Queries (7 Bangalore-targeted)
- `.NET Developer Bangalore India`
- `Backend Engineer .NET Bangalore India`
- `Microservices .NET Backend Engineer Bangalore India`
- `SDE 2 .NET Bangalore India`
- `Senior Software Engineer .NET Bangalore India`
- `Azure .NET Developer Bangalore India`
- `Software Engineer II .NET Bangalore India`

### Naukri Queries (2 via Apify)
- `.NET Developer` — Bangalore, 5yr exp, last 7 days
- `C# Developer` — Bangalore, 5yr exp, last 7 days

---

## Job Sources

| Source | How | Jobs/Run | Cost |
|---|---|---|---|
| **JSearch** (RapidAPI) | LinkedIn + Indeed + Glassdoor + Google Jobs | ~140 raw (7 queries × 4 pages × 10 jobs) | Free |
| **Naukri .NET** (Apify) | `nuclear_quietude~naukri-job-scraper` | ~20-30 raw | ~$5/mo credit |
| **Naukri C#** (Apify) | same actor | ~30-40 raw | included |

After deduplication: ~200-210 unique. Cross-run dedup skips already-seen. ~30-50 pass score ≥ 75.

---

## Customization

| What | Where | Current Value |
|---|---|---|
| Min score threshold | `minScore` in Set Job Preferences | 75 |
| JSearch queries | `searchQuery` strings in Set Job Preferences | 7 Bangalore .NET/C# queries |
| Naukri .NET search | `naukriJobTitle`/`naukriLocation` in Set Naukri Queries | `.NET Developer`, Bangalore |
| Naukri C# search | `naukriJobTitle`/`naukriLocation` in Set Naukri Queries (C#) | `C# Developer`, Bangalore |
| JSearch pages/query | `num_pages` in Search Google Jobs | 4 |
| Naukri jobs/search | `no_of_jobs` in Search Naukri nodes | 30 |
| Run frequency | Trigger node | Manual (see schedule section below) |

### Switch to Weekly Schedule

Replace Manual Trigger with Schedule Trigger — Monday 9 AM:

```json
{
  "type": "n8n-nodes-base.scheduleTrigger",
  "typeVersion": 1.2,
  "parameters": {
    "rule": {
      "interval": [{ "triggerAtHour": 9, "triggerAtMinute": 0, "triggerAtDay": 1 }]
    }
  }
}
```

---

## Using with Claude Code (Optional)

This repo includes two git submodules for building/modifying workflows with [Claude Code](https://claude.ai/code):

- **`n8n-mcp/`** — MCP server giving Claude Code direct access to n8n workflow management
- **`n8n-skills/`** — Expert skills for n8n expression syntax, validation, and node configuration

```bash
claude mcp add n8n-mcp \
  -e MCP_MODE=stdio \
  -e LOG_LEVEL=error \
  -e DISABLE_CONSOLE_OUTPUT=true \
  -e N8N_API_URL=http://localhost:5678 \
  -e N8N_API_KEY=YOUR_N8N_API_KEY \
  -s local \
  -- npx n8n-mcp
```

Get your API key: n8n → Settings → API → Create API Key. Then start a **new** Claude Code conversation.

---

## Troubleshooting

| Error | Fix |
|---|---|
| "Authorization failed" on Score OpenAI API | Key must start with `sk-proj-` in Set Job Preferences |
| "Service refused connection" on Naukri | Timeout < 300000ms — check Search Naukri node → Options |
| "SSL Issue" on Naukri | Enable "Ignore SSL Issues" in Search Naukri node → Options |
| "Too many requests" on Google Sheets | Should not occur — Aggregate Jobs node collapses items before Sheets read |
| "Task request timed out" on Set Job Preferences | Another workflow using task runner — wait and retry, or add `N8N_RUNNERS_MAX_CONCURRENCY=10` to docker-compose.yml |
| Resume not found / ENOENT | Verify PDF at `resumes/your-resume.pdf` and Docker volume mount exists |
| "Access blocked" on Google OAuth | Add yourself as test user: Google Cloud Console → OAuth consent screen → Audience |
| 0 rows in Google Sheet | Check Sheets credential and that tab is named exactly **"Job Tracker"** |
| "No new jobs this week" error | All discovered jobs already in sheet — expected after first run if run again same day |

Full documentation: [docs/setup-guide.md](docs/setup-guide.md) | [docs/workflow-reference.md](docs/workflow-reference.md)

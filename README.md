# n8n Job Search Automation

Automated pipeline that searches for jobs matching your resume, scores them with AI, generates tailored cover letters, and delivers a digest email -- fully hands-free.

**Pipeline:** Read Resume PDF -> Extract PDF Text -> Set Job Preferences (roles, locations, API keys) -> Search Google Jobs via SerpAPI (dual-location: Bangalore onsite/hybrid + India remote) -> Parse & normalize results -> Filter duplicates -> Gemma 3 27B scores each job against resume (0-100) -> Filter top matches (score >= 50) -> Generate tailored cover letters -> Save to Google Sheets -> Send Gmail digest email

**Cost:** $0/month (all free tiers)

---

## How It Works

```
Read Resume PDF (Binary)
    |
    v
Extract PDF Text
    |
    v
Set Job Preferences (17 roles, dual-location, API keys, minScore=50)
    |
    v
Search Google Jobs - Bangalore (SerpAPI: onsite/hybrid, 25 results, date_posted:week)
    |
    v
Search Google Jobs - Remote India (SerpAPI: remote jobs, 25 results, date_posted:week)
    |
    v
Merge Job Results (combine both location searches)
    |
    v
Parse & Normalize Results (extract title, company, location, link)
    |
    v
Filter Duplicates (deduplicate by company + title)
    |
    v
Gemma 3 27B scores each job against your resume (0-100 match score)
    |           [batchSize=1, batchInterval=20000ms for rate limit protection]
    v
Parse AI Scores (extract score, recommendation, reasoning, missing skills)
    |
    v
Filter Top Matches (score >= 50 only)
    |
    v
Generate Tailored Cover Letters (Gemma 3 27B, one per job)
    |           [batchSize=1, batchInterval=20000ms]
    v
Format for Google Sheets (add date, status columns)
    |
    v
Save to Google Sheets ("Job Tracker" spreadsheet)
    |
    v
Build Email Digest (HTML summary of new matches)
    |
    v
Send Gmail Notification (digest with top matches + cover letters)
```

**16 nodes, 15 connections. Runtime: 5-15 minutes depending on job count.**

---

## Prerequisites

Before you start, make sure you have:

- **Docker** installed ([Get Docker](https://docs.docker.com/get-docker/))
- **Git** installed ([Get Git](https://git-scm.com/downloads))
- **SerpAPI account** (free tier: 100 searches/month) -- [serpapi.com](https://serpapi.com)
- **Google Gemini API key** (free, no billing required) -- [aistudio.google.com/apikey](https://aistudio.google.com/apikey)
- **Google account** with Google Sheets and Gmail access
- **Your resume** in PDF format

---

## Quick Start

### Step 1: Clone the Repository

```bash
git clone --recurse-submodules https://github.com/skylark248/N8nJobSearchAutomation.git
cd N8nJobSearchAutomation
```

> **Already cloned without `--recurse-submodules`?** Run this to fetch the submodules:
> ```bash
> git submodule init && git submodule update
> ```

### Step 2: Set Up Environment

```bash
cp .env.example .env
```

Open `.env` and fill in your API keys (this file is for your reference -- credentials are configured inside n8n, not read from `.env`).

### Step 3: Start n8n

```bash
docker compose up -d
```

Run this from the **parent n8n directory** (not from JobSearchAutomation/). This uses the custom Docker image with FFmpeg baked in. The `docker-compose.yml` mounts `./JobSearchAutomation/resumes` to `/home/node/.n8n-files` inside the container so your resume PDF survives restarts.

### Step 4: Open n8n

Open [http://localhost:5678](http://localhost:5678) in your browser. Create an account when prompted (this is your local instance, data stays on your machine).

### Step 5: Import the Workflow

1. In n8n, click **"Add workflow"** (or the import icon)
2. Click **"Import from file"**
3. Select `exports/job-search-automation.json` from the cloned repo
4. The full 16-node workflow appears ready to configure

### Step 6: Configure Credentials

You need 3 credentials configured **inside n8n**:

| # | Credential | Where to Get | Cost |
|---|---|---|---|
| 1 | **SerpAPI Key** | [serpapi.com](https://serpapi.com) -> Dashboard -> API Key | Free (100/mo) |
| 2 | **Gemini API Key** | [aistudio.google.com/apikey](https://aistudio.google.com/apikey) -> Create API Key | Free (30 RPM) |
| 3 | **Google OAuth2** | [Google Cloud Console](https://console.cloud.google.com) -> Sheets API + Gmail API | Free |

- The **SerpAPI key** goes in the **Set Job Preferences** node (`serpApiKey` field in the code)
- The **Gemini API key** goes in the **Set Job Preferences** node (`geminiApiKey` field in the code)
- The **Google OAuth2** credential is created in n8n's credential manager and linked to the Google Sheets and Gmail nodes

**Step-by-step instructions for each credential:** See [docs/setup-guide.md](docs/setup-guide.md)

### Step 7: Create Google Sheet

1. Go to [Google Sheets](https://sheets.google.com) and create a new spreadsheet
2. Name it **"Job Tracker"**
3. In row 1, add these headers (columns A through L):

| A | B | C | D | E | F | G | H | I | J | K | L |
|---|---|---|---|---|---|---|---|---|---|---|---|
| Date | Job Title | Company | Location | Match Score | Recommendation | Match Reason | Missing Skills | Apply Link | Cover Letter | Status | Applied Date |

4. Copy the **spreadsheet ID** from the URL: `https://docs.google.com/spreadsheets/d/SPREADSHEET_ID/edit`
5. In n8n, open the **"Set Job Preferences"** node and paste your spreadsheet ID in the `spreadsheetId` field

### Step 8: Place Your Resume PDF

1. Place your resume PDF file in the `resumes/` folder (any filename ending in `.pdf`)
2. The workflow uses a **Read Binary File** node that reads from `/home/node/.n8n-files/` inside the Docker container
3. The `docker-compose.yml` mounts `./JobSearchAutomation/resumes` to `/home/node/.n8n-files`, so the file is accessible automatically
4. The **Extract From File** node converts the PDF to plain text for AI scoring

> **Note:** Only one PDF should be in the `resumes/` folder. If multiple are present, update the filename in the Read Binary File node.

### Step 9: Update Set Job Preferences

In the **"Set Job Preferences"** node, update these fields:

- `yourEmail` -- your Gmail address for receiving the digest
- `spreadsheetId` -- your Google Sheet ID from Step 7
- `targetRoles` -- the job titles to search for (17 defaults provided)
- `locations` -- dual-location search configuration (Bangalore + India remote by default)

### Step 10: First Test Run

1. Click **"Test Workflow"** (play button in top-right)
2. Wait 5-15 minutes for the full pipeline to complete (rate-limited HTTP calls)
3. Check your Google Sheet -- new job matches should appear
4. Check your Gmail -- you should receive a digest email with top matches
5. Once verified, optionally switch to a daily schedule (see Customization below)

---

## Project Structure

```
.
├── README.md                              # You are here
├── CLAUDE.md                              # Claude Code AI assistant config
├── .env.example                           # Template for API keys (copy to .env)
├── .gitignore                             # Keeps secrets out of git
├── .gitmodules                            # Git submodule references
│
├── docs/
│   ├── setup-guide.md                     # Step-by-step credential setup
│   └── workflow-reference.md              # Complete node-by-node documentation
│
├── exports/
│   └── job-search-automation.json         # Importable n8n workflow file
│
├── resumes/
│   ├── README.txt                         # Instructions for placing your resume
│   └── .gitkeep                           # Keeps folder in git (PDFs are gitignored)
│
├── n8n-mcp/                               # MCP server for Claude Code (git submodule)
└── n8n-skills/                            # Claude Code skills (git submodule)
```

---

## Cost Breakdown

| Service | Per Run | Monthly (30 runs) |
|---|---|---|
| SerpAPI (job searches) | $0.00 | $0.00 (free tier: 100/mo) |
| Gemma 3 27B (job scoring) | $0.00 | $0.00 (free tier: 30 RPM, 15K TPM) |
| Gemma 3 27B (cover letters) | $0.00 | $0.00 (free tier) |
| Google Sheets API | $0.00 | $0.00 |
| Gmail API | $0.00 | $0.00 |
| **Total** | **$0.00** | **$0.00** |

---

## Target Roles

The workflow searches for these 17 roles by default (configurable in the Set Job Preferences node):

- Backend Developer
- Senior Backend Developer
- .NET Developer
- Senior .NET Developer
- Full Stack Developer
- Senior Full Stack Developer
- Software Engineer 2
- SDE 2
- Senior Software Engineer
- C# Developer
- ASP.NET Developer
- Software Engineer
- Backend Engineer
- Application Developer
- Software Developer
- AI Platform Engineer
- Backend Engineer AI
- Gen AI Engineer

---

## Customization

| What | How |
|---|---|
| Change target job titles | Edit "Set Job Preferences" node -- update `targetRoles` array |
| Change location | Edit "Set Job Preferences" node -- update location configuration (supports dual-location) |
| Change minimum match score | Edit "Set Job Preferences" node -- change `minScore` (default: 50) |
| Switch to daily schedule | Replace Manual Trigger with Schedule Trigger (cron: `0 9 * * *` for daily 9 AM) |
| Add more job sources | Add HTTP Request nodes for LinkedIn, Indeed, or other APIs before the merge step |
| Change AI model | Update the model name in Score & Match and Generate Cover Letter HTTP Request URLs (e.g., swap `gemma-3-27b-it` for another model) |
| Change email recipient | Edit "Set Job Preferences" node -- update `yourEmail` field |
| Add Slack notifications | Add Slack node after Gmail notification |

See [docs/workflow-reference.md](docs/workflow-reference.md) for detailed node-by-node documentation.

---

## Using with Claude Code (Optional)

This repo includes two git submodules for building/modifying n8n workflows with [Claude Code](https://claude.ai/code):

- **`n8n-mcp/`** -- MCP server that gives Claude Code direct access to 1,084+ n8n nodes and workflow management tools
- **`n8n-skills/`** -- Expert skills for expression syntax, validation, node configuration, and workflow patterns

### Setup

1. Make sure submodules are cloned (done automatically if you used `--recurse-submodules`):
   ```bash
   git submodule init && git submodule update
   ```

2. Install the MCP server dependencies:
   ```bash
   cd n8n-mcp && npm install && cd ..
   ```

3. Get your n8n API key: Open n8n -> **Settings** -> **API** -> **Create API Key**

4. Register the MCP server with Claude Code:
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

5. Start a **new Claude Code conversation** (MCP tools load on conversation start)

Then Claude Code can create, modify, validate, and deploy n8n workflows conversationally.

---

## Troubleshooting

| Error | Fix |
|---|---|
| "Invalid API key" on SerpAPI | Verify key at serpapi.com/dashboard, check free tier quota remaining |
| "Invalid API key" on Gemini | Check key at aistudio.google.com/apikey, verify it's in Set Job Preferences |
| "Access blocked" on Google OAuth | Add yourself as test user: Google Cloud Console -> OAuth consent screen -> Audience -> Add Users |
| "This app isn't verified" | Click Advanced -> "Go to n8n (unsafe)" -- this is your own app, safe to proceed |
| "Redirect URI mismatch" | Verify URI is exactly `http://localhost:5678/rest/oauth2-credential/callback` |
| "Spreadsheet not found" | Verify the spreadsheet ID is correct and your Google account has access |
| "Insufficient permissions" on Sheets | Re-authorize Google OAuth2 with Sheets scope enabled |
| No jobs found | Try broader role titles or different location in Set Job Preferences |
| Rate limit errors (429) | The workflow uses batchSize=1 with 20s intervals; if still hitting limits, increase batchInterval |
| Resume not found | Ensure PDF is in `resumes/` folder and Docker is running with the volume mount |
| "ENOENT" file read error | Check that `N8N_RESTRICT_FILE_ACCESS_TO` includes `/home/node/.n8n-files` in docker-compose.yml |
| n8n-mcp or n8n-skills folders are empty | Run `git submodule init && git submodule update` |

Full troubleshooting: [docs/setup-guide.md](docs/setup-guide.md#troubleshooting)

---

## Migrating to Another n8n Instance

1. Import `exports/job-search-automation.json` on the new instance
2. Re-configure all 3 credentials (SerpAPI key, Gemini API key, Google OAuth2)
3. Update the Google Sheet spreadsheet ID in Set Job Preferences
4. Place your resume PDF in the resumes directory and ensure the volume mount is configured
5. Update `N8N_RESTRICT_FILE_ACCESS_TO` environment variable to include the resumes mount path
6. Test run to verify jobs are found and saved

See [docs/setup-guide.md](docs/setup-guide.md#migrating-to-a-different-n8n-instance) for full migration guide.

---

## Future Plans

- Add LinkedIn Jobs API as an additional source
- Implement auto-apply for select job boards
- Add interview preparation notes generated by AI
- Track application status changes automatically
- A/B test different cover letter styles for response rates
- Add Slack/Discord notification channel support
- Explore upgrading to larger AI models as free tiers expand

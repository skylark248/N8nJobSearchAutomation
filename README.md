# n8n Job Search Automation

Automated pipeline that searches for jobs matching your resume, scores them with AI, generates tailored cover letters, and delivers a daily digest -- fully hands-free.

**Pipeline:** Search Google Jobs (SerpAPI) -> Parse results -> Filter duplicates -> GPT-5 scores each job against resume (0-100) -> Filter top matches (score >= 70) -> Generate tailored cover letters -> Save to Google Sheets -> Send Gmail digest email

**Cost:** ~$0.45 per run (~$13.50/month for daily runs)

---

## How It Works

```
Manual Trigger
    |
    v
Set Job Preferences (target roles, location, resume text)
    |
    v
Search Google Jobs via SerpAPI (multiple role queries)
    |
    v
Parse & Normalize Results (extract title, company, location, link)
    |
    v
Filter Duplicates (deduplicate by company + title)
    |
    v
GPT-5 scores each job against your resume (0-100 match score)
    |
    v
Parse AI Scores (extract score, recommendation, reasoning, missing skills)
    |
    v
Filter Top Matches (score >= 70 only)
    |
    v
Generate Tailored Cover Letters (GPT-5, one per job)
    |
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
Send Gmail Notification (daily digest with top matches)
    |
    v
Done Output (summary stats)
```

---

## Prerequisites

Before you start, make sure you have:

- **Docker** installed ([Get Docker](https://docs.docker.com/get-docker/))
- **Git** installed ([Get Git](https://git-scm.com/downloads))
- **SerpAPI account** (free tier: 100 searches/month) -- [serpapi.com](https://serpapi.com)
- **OpenAI account** with API billing enabled ($10-20 to start) -- [platform.openai.com](https://platform.openai.com)
- **Google account** with Google Sheets and Gmail access
- **Your resume** in plain text format

---

## Quick Start

### Step 1: Clone the Repository

```bash
git clone --recurse-submodules https://github.com/your-username/JobSearchAutomation.git
cd JobSearchAutomation
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
docker run --rm --name n8n -p 5678:5678 -v $(pwd):/home/node/.n8n n8nio/n8n
```

Open [http://localhost:5678](http://localhost:5678) in your browser. Create an account when prompted (this is your local instance, data stays on your machine).

### Step 4: Import the Workflow

1. In n8n, click **"Add workflow"** (or the import icon)
2. Click **"Import from file"**
3. Select `exports/job-search-automation.json` from the cloned repo
4. The full 14-node workflow appears ready to configure

### Step 5: Configure Credentials

You need 3 credentials. All are configured **inside n8n** (not in code):

| # | Credential | Where to Get | Cost |
|---|---|---|---|
| 1 | **SerpAPI Key** | [serpapi.com](https://serpapi.com) -> Dashboard -> API Key | Free (100/mo) |
| 2 | **OpenAI API Key** | [platform.openai.com](https://platform.openai.com) -> API Keys | ~$13.50/mo |
| 3 | **Google OAuth2** | [Google Cloud Console](https://console.cloud.google.com) -> Sheets API + Gmail API | Free |

**Step-by-step instructions for each credential:** See [docs/setup-guide.md](docs/setup-guide.md)

### Step 6: Create Google Sheet

1. Go to [Google Sheets](https://sheets.google.com) and create a new spreadsheet
2. Name it **"Job Tracker"**
3. In row 1, add these headers (columns A through L):

| A | B | C | D | E | F | G | H | I | J | K | L |
|---|---|---|---|---|---|---|---|---|---|---|---|
| Date | Job Title | Company | Location | Match Score | Recommendation | Match Reason | Missing Skills | Apply Link | Cover Letter | Status | Applied Date |

4. Copy the **spreadsheet ID** from the URL: `https://docs.google.com/spreadsheets/d/SPREADSHEET_ID/edit`
5. In n8n, open the **"Save to Google Sheets"** node and paste your spreadsheet ID

### Step 7: Paste Your Resume

1. In n8n, open the **"Set Job Preferences"** node
2. Find the `resumeText` field
3. Paste your full resume as plain text
4. Optionally update `targetLocation` and `targetRoles` to match your preferences

### Step 8: First Test Run

1. Click **"Test Workflow"** (play button in top-right)
2. Wait 2-5 minutes for the full pipeline to complete
3. Check your Google Sheet -- new job matches should appear
4. Check your Gmail -- you should receive a digest email with top matches
5. Once verified, optionally switch to a daily schedule (see Customization below)

---

## Project Structure

```
.
├── README.md                              # You are here
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
├── CLAUDE.md                              # Claude Code AI assistant config
├── n8n-mcp/                               # MCP server for Claude Code (git submodule)
└── n8n-skills/                            # Claude Code skills (git submodule)
```

---

## Cost Breakdown

| Service | Per Run | Monthly (30 runs) |
|---|---|---|
| SerpAPI (job searches) | $0.00 | $0.00 (free tier: 100/mo) |
| GPT-5 (job scoring) | $0.30 | $9.00 |
| GPT-5 (cover letters) | $0.15 | $4.50 |
| Google Sheets API | $0.00 | $0.00 |
| Gmail API | $0.00 | $0.00 |
| **Total** | **~$0.45** | **~$13.50** |

---

## Target Roles

The workflow searches for these roles by default (configurable in the Set Job Preferences node):

- Backend Developer
- Senior Backend Developer
- .NET Developer
- Senior .NET Developer
- Full Stack Developer
- Senior Full Stack Developer
- Software Engineer 2
- SDE 2
- Senior Software Engineer

---

## Customization

| What | How |
|---|---|
| Change target job titles | Edit "Set Job Preferences" node -- update `targetRoles` array |
| Change location | Edit "Set Job Preferences" node -- update `targetLocation` field |
| Change minimum match score | Edit "Filter Top Matches" node -- change threshold from 70 |
| Switch to daily schedule | Replace Manual Trigger with Schedule Trigger (cron: `0 9 * * *` for daily 9 AM) |
| Add more job sources | Add HTTP Request nodes for LinkedIn, Indeed, or other APIs before the dedup step |
| Change AI model | Edit "Score Job Match" and "Generate Cover Letter" nodes -- update model parameter |
| Change email format | Edit "Build Email Digest" node -- modify HTML template |
| Add Slack notifications | Add Slack node after Gmail notification |

See [docs/workflow-reference.md](docs/workflow-reference.md) -> Modification Guide for details.

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
| "Invalid API key" on OpenAI | Check key at platform.openai.com, verify billing credits |
| "Access blocked" on Google OAuth | Add yourself as test user: Google Cloud Console -> OAuth consent screen -> Audience -> Add Users |
| "This app isn't verified" | Click Advanced -> "Go to n8n (unsafe)" -- this is your own app, safe to proceed |
| "Redirect URI mismatch" | Verify URI is exactly `http://localhost:5678/rest/oauth2-credential/callback` |
| "Spreadsheet not found" | Verify the spreadsheet ID is correct and your Google account has access |
| "Insufficient permissions" on Sheets | Re-authorize Google OAuth2 with Sheets scope enabled |
| No jobs found | Try broader role titles or different location in Set Job Preferences |
| n8n-mcp or n8n-skills folders are empty | Run `git submodule init && git submodule update` |

Full troubleshooting: [docs/setup-guide.md](docs/setup-guide.md#troubleshooting)

---

## Migrating to Another n8n Instance

1. Import `exports/job-search-automation.json` on the new instance
2. Re-configure all 3 credentials
3. Update the Google Sheet spreadsheet ID
4. Paste your resume text into the Set Job Preferences node
5. Test run to verify jobs are found and saved

See [docs/setup-guide.md](docs/setup-guide.md#migrating-to-a-different-n8n-instance) for full migration guide.

---

## Future Plans

- Add LinkedIn Jobs API as an additional source
- Implement auto-apply for select job boards
- Add interview preparation notes generated by AI
- Track application status changes automatically
- A/B test different cover letter styles for response rates

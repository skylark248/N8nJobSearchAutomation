# Job Search Automation — Setup Guide

## Prerequisites

- n8n running locally in Docker (see Docker Setup below)
- RapidAPI account with JSearch Basic subscription (free)
- OpenAI API key (pay-as-you-go, ~$0.075/run)
- Apify account ($5/month free credit, covers ~60+ runs)
- Google account with Sheets and Gmail access
- Resume PDF at `resumes/your-resume.pdf`

---

## Docker Setup

### Directory Structure

This project expects to live inside a parent `n8n/` directory that holds Docker runtime data:

```
n8n/                          ← parent directory
├── docker-compose.yml
├── database.sqlite
├── binaryData/
├── N8nJobSearchAutomation/   ← this repo (clone here)
│   └── resumes/
│       └── your-resume.pdf
└── YtShortsAutomation/       ← optional sibling (ignore if not using)
```

### docker-compose.yml Requirements

Your `docker-compose.yml` must include these settings for the workflow to function:

```yaml
services:
  n8n:
    image: n8nio/n8n:latest
    ports:
      - "5678:5678"
    environment:
      N8N_RUNNERS_TASK_TIMEOUT: 900        # 15 min — Naukri Apify actor takes 4-5 min
      N8N_RESTRICT_FILE_ACCESS_TO: /home/node/.n8n;/home/node/.n8n-files
    volumes:
      - .:/home/node/.n8n                  # n8n runtime data
      - ./N8nJobSearchAutomation/resumes:/home/node/.n8n-files  # resume PDFs
```

### Starting n8n

```bash
# From the parent n8n/ directory (not from inside N8nJobSearchAutomation/)
docker compose up -d
```

n8n will be available at [http://localhost:5678](http://localhost:5678).

### Resume Volume Mount

Place your resume as `N8nJobSearchAutomation/resumes/your-resume.pdf`. The Docker volume maps this to `/home/node/.n8n-files/your-resume.pdf` inside the container. The **Read Resume PDF** node reads from this exact path.

---

## Credential 1: JSearch (RapidAPI)

**Used by**: Search Google Jobs (JSearch) node — aggregates LinkedIn, Indeed, Glassdoor, Google Jobs

### Get API Key

1. Go to [rapidapi.com](https://rapidapi.com) and create an account
2. Search for **"JSearch"** in the API marketplace
3. Subscribe to the **JSearch Basic** plan (free — 200 requests/month, no credit card)
4. Go to the JSearch API page → **"Endpoints"** tab → copy the `X-RapidAPI-Key` from the code examples

### Free Tier Usage
- **200 requests/month** limit
- This workflow uses **28 requests/run** (7 queries × 4 pages)
- Supports ~7 runs/month on free tier — more than enough for weekly runs

### Add to Workflow
Open the **Set Job Preferences** node and set:
```javascript
rapidApiKey: 'YOUR_RAPIDAPI_KEY'
```

---

## Credential 2: OpenAI API

**Used by**: Score OpenAI API node (job scoring) + Cover Letter OpenAI API node

**Model**: `gpt-4o-mini` — fast, accurate for structured JSON and text generation

### Get API Key

1. Go to [platform.openai.com/api-keys](https://platform.openai.com/api-keys)
2. Sign in → click **"Create new secret key"**
3. Copy the key — it starts with `sk-proj-...`

> **Important**: The key must include the `sk-proj-` prefix. Without it, you'll get "Authorization failed" errors.

### Cost
- Scoring ~200 jobs × ~5500 tokens each: ~$0.055/run
- Cover letters ~40 jobs × ~5500 tokens each: ~$0.011/run
- **Total**: ~$0.076/run, ~$0.30/month at 4 runs/month

### Add to Workflow
Open the **Set Job Preferences** node and set:
```javascript
openAiApiKey: 'sk-proj-YOUR_KEY_HERE'
```

---

## Credential 3: Apify (Naukri Scraper)

**Used by**: Search Naukri (Apify) + Search Naukri C# (Apify) nodes

**Actor**: `nuclear_quietude~naukri-job-scraper` (pay-per-usage — do NOT rent)

### Get API Token

1. Go to [apify.com](https://apify.com) and create an account
2. Go to [console.apify.com/account/integrations](https://console.apify.com/account/integrations)
3. Copy your **Personal API token** (starts with `apify_api_...`)

### Free Credit & Cost
- **$5/month** recurring free credit (auto-credited each month)
- Actor costs ~$0.005-0.01/run (2 searches × ~$0.003-0.005 each)
- At ~4 runs/month, total Apify cost is ~$0.04/month — well within free credit
- **Do NOT rent the actor** (rented actors = $19.99/month flat fee regardless of usage)

### Actor Runtime
- ~4-5 minutes per call (uses `run-sync-get-dataset-items` — waits for completion)
- Workflow runs **2 Apify calls in parallel** (.NET + C#) — total Naukri time: ~4-5 min
- Timeout is set to **300000ms** (5 minutes) — do not reduce this

### Add to Workflow
Open the **Set Job Preferences** node and set:
```javascript
apifyToken: 'apify_api_YOUR_TOKEN_HERE'
```

---

## Credential 4: Google OAuth2 (Sheets + Gmail)

**Used by**: Read Existing Jobs + Save to Google Sheets + Send Gmail Digest

Both Google Sheets and Gmail use the same OAuth2 client credentials.

### Step 1: Create Google Cloud Project

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Click the project dropdown → **"New Project"**
3. Name: `n8n Job Search` → Create
4. Make sure this project is selected

### Step 2: Enable Required APIs

1. Go to **APIs & Services** → **Library**
2. Enable **Google Sheets API**
3. Enable **Gmail API**

### Step 3: Configure OAuth Consent Screen

1. Go to **APIs & Services** → **OAuth consent screen**
2. Select **External** → Create
3. Fill in:
   - App name: `n8n Job Search`
   - Support email: your Gmail address
   - Developer contact email: your Gmail address
4. Click through Scopes section without changes
5. **Test users** — CRITICAL, do not skip:
   - Click **Add Users** → add your exact Gmail address
   - Without this step, you'll get "Access blocked: This app hasn't been verified"
6. Save and Continue

### Step 4: Create OAuth2 Credentials

1. Go to **APIs & Services** → **Credentials**
2. Click **+ Create Credentials** → **OAuth client ID**
3. Application type: **Web application**
4. Name: `n8n`
5. Under **Authorized redirect URIs** → Add: `http://localhost:5678/rest/oauth2-credential/callback`
6. Click Create → copy **Client ID** and **Client Secret**

### Step 5: Add Google Sheets Credential to n8n

1. Open n8n → Credentials (top menu) → **Add Credential**
2. Search for **Google Sheets OAuth2 API** → Select
3. Paste your Client ID and Client Secret
4. Click **Sign in with Google** → authorize with your Google account
5. If you see "This app isn't verified": click **Advanced** → **Go to n8n Job Search (unsafe)** — this is safe since it's your own OAuth app
6. Name the credential (e.g., `Google Sheets account`) and save
7. Open the **Save to Google Sheets** node and the **Read Existing Jobs** node → link this credential in both

### Step 6: Add Gmail Credential to n8n

1. Open n8n → Credentials → **Add Credential**
2. Search for **Gmail OAuth2 API** → Select
3. Use the **same** Client ID and Client Secret from Step 4
4. Sign in and authorize → Save
5. Open the **Send Gmail Digest** node → link this credential

---

## Google Sheet Setup

### Create the Sheet

1. Go to [sheets.google.com](https://sheets.google.com) → **Blank spreadsheet**
2. Rename the file to **Job Tracker** (click the title at the top)
3. Rename the tab (bottom) to **Job Tracker** as well
4. Add these exact headers in row 1 (columns A–L):

| A | B | C | D | E | F | G | H | I | J | K | L |
|---|---|---|---|---|---|---|---|---|---|---|---|
| Date | Job Title | Company | Location | Match Score | Recommendation | Match Reason | Missing Skills | Apply Link | Cover Letter | Status | Applied Date |

5. Freeze row 1: **View** → **Freeze** → **1 row**
6. Optional: Add data validation on column K (Status):
   - Right-click column K → Data validation
   - Criteria: List of items: `Not Applied, Applied, Interview`, `Rejected, Offer`

### Get Spreadsheet ID

From the URL: `https://docs.google.com/spreadsheets/d/SPREADSHEET_ID/edit`

Copy the long string between `/d/` and `/edit`.

### Configure in n8n

Open **Set Job Preferences** and set:
```javascript
spreadsheetId: 'YOUR_SPREADSHEET_ID'
```

> The workflow uses `appendOrUpdate` matching on Job Title + Company + Apply Link — re-running the same week won't create duplicates. The Status column is excluded from updates, so your manual "Applied" / "Interview" edits are preserved across runs.

---

## Final Configuration Checklist

Open the **Set Job Preferences** node and verify all 5 values are set:

```javascript
const baseConfig = {
  minScore: 75,                          // Jobs must score >= 75 to generate cover letter
  yourEmail: 'YOUR_EMAIL@gmail.com',    // Where to send the digest
  spreadsheetId: 'YOUR_SPREADSHEET_ID', // Google Sheet ID
  openAiApiKey: 'sk-proj-YOUR_KEY',     // Must start with sk-proj-
  rapidApiKey: 'YOUR_RAPIDAPI_KEY',     // From RapidAPI JSearch
  apifyToken: 'apify_api_YOUR_TOKEN',   // From Apify console
  resumeText: resumeText                // Populated automatically from PDF
};
```

---

## First Test Run

### Before Running, Verify

- [ ] Resume PDF at `resumes/your-resume.pdf`
- [ ] `rapidApiKey` set in Set Job Preferences
- [ ] `openAiApiKey` (with `sk-proj-` prefix) set in Set Job Preferences
- [ ] `apifyToken` set in Set Job Preferences
- [ ] `yourEmail` and `spreadsheetId` set in Set Job Preferences
- [ ] Google Sheets OAuth2 credential linked in Save to Google Sheets + Read Existing Jobs
- [ ] Gmail OAuth2 credential linked in Send Gmail Digest
- [ ] Google Sheet created with correct 12 headers and tab named "Job Tracker"

### Run

1. Click **"Test Workflow"** in n8n (▶ button)
2. Wait ~21 minutes (Naukri Apify calls take 4-5 min, scoring ~200 jobs at 3s each takes ~10 min)
3. Watch nodes turn green one by one

### Expected Results (First Run)

| Stage | Expected Count |
|---|---|
| Set Job Preferences output | 7 items |
| After JSearch (7 queries × 4 pages) | ~140 raw jobs |
| After Naukri .NET + C# (Apify) | ~50-70 more raw jobs |
| After deduplication | ~200-210 unique jobs |
| After cross-run dedup (first run = 0 seen) | ~200-210 new jobs |
| After AI scoring + filter (≥ 75) | ~30-70 final matches |
| Google Sheet rows added | Same as final matches |
| Gmail digest | 1 email with all matches |

### Common First-Run Issues

| Issue | Fix |
|---|---|
| "Authorization failed" on Score OpenAI API | Key missing `sk-proj-` prefix in Set Job Preferences |
| "Service refused connection" on Naukri | Timeout < 300000ms in Search Naukri node options |
| "SSL Issue" on Search Naukri | Enable "Ignore SSL Issues" in Search Naukri node → Options |
| "Task request timed out" on Set Job Preferences | Another workflow using task runner — wait and retry |
| No rows in Google Sheet | Check that Sheets OAuth2 is linked and tab is named "Job Tracker" |
| No email received | Check Gmail OAuth2 credential and `yourEmail` value |
| "Resume PDF appears empty" error | PDF is image-based (scanned) — use a text-based PDF |
| 0 unique jobs after parse | JSearch returned empty — verify `rapidApiKey` and RapidAPI subscription active |

---

## Switching to Weekly Schedule

Replace the **Manual Trigger** node with a **Schedule Trigger**:

1. In n8n, delete the Manual Trigger node
2. Add a Schedule Trigger node
3. Configure: Every week on Monday at 09:00

Equivalent cron: `0 9 * * 1`

Jobs are freshest early Monday morning — many companies post over the weekend.

---

## Credential Summary

| # | Credential Type | Where Used | Cost |
|---|---|---|---|
| 1 | `rapidApiKey` (code field) | Set Job Preferences | Free (200 req/mo) |
| 2 | `openAiApiKey` (code field) | Set Job Preferences | ~$0.076/run |
| 3 | `apifyToken` (code field) | Set Job Preferences | ~$0/run (within $5 credit) |
| 4 | Google Sheets OAuth2 | Read Existing Jobs + Save to Google Sheets | Free |
| 5 | Gmail OAuth2 | Send Gmail Digest | Free |

---

## Troubleshooting

### "Authorization failed" on OpenAI nodes
- Verify the key in Set Job Preferences starts with `sk-proj-`
- Check the key is active at platform.openai.com/api-keys
- Ensure billing is set up on your OpenAI account

### "The service refused the connection" on Naukri
- Timeout must be 300000ms (5 minutes) — check Search Naukri node → Options
- The actor takes 4-5 minutes; 120000ms (2 min) will always fail

### "SSL Issue" on Search Naukri
- Open Search Naukri node → Options → enable **"Ignore SSL Issues"**
- Same for Search Naukri C# node

### "Too many requests" on Google Sheets
- The Aggregate Jobs node should prevent this (collapses all jobs into 1 item before the Sheets read)
- If it still occurs, check that the Aggregate Jobs → Read Existing Jobs connection exists

### "No new jobs this week"
- All discovered jobs are already in the sheet from a previous run
- Expected if you run twice in one day — wait for next week when listings refresh

### "Task request timed out after 60 seconds" on Set Job Preferences
- The n8n task runner is busy with another workflow
- Wait for the other workflow to complete, then retry
- Permanent fix: add `N8N_RUNNERS_MAX_CONCURRENCY: 10` to docker-compose.yml environment

### Google OAuth "Access blocked"
- Go to Google Cloud Console → APIs & Services → OAuth consent screen → **Audience** tab
- Click **Add Users** → add your Gmail address → Save
- Re-authorize in n8n

### "Spreadsheet not found" or rows not appearing
- Verify spreadsheet ID (long string in URL — not the sheet name)
- Verify the tab is named exactly **"Job Tracker"** (case-sensitive)
- Verify the Google Sheets credential has edit access to that spreadsheet

### Resume PDF not found (ENOENT error)
- Ensure file is named exactly `your-resume.pdf`
- Check volume mount in docker-compose.yml: `./N8nJobSearchAutomation/resumes:/home/node/.n8n-files`
- Run `docker compose ps` to verify container is running

### Apify billing — checking usage
- Go to [console.apify.com/billing](https://console.apify.com/billing)
- $5/month free credit auto-renews monthly
- At ~$0.01/run × 4 runs/month = ~$0.04/month — far within free credit

---

## Migration to New n8n Instance

If moving to a new server or fresh Docker install:

- [ ] Import `exports/job-search-automation.json`
- [ ] Set `rapidApiKey`, `openAiApiKey`, `apifyToken`, `yourEmail`, `spreadsheetId` in Set Job Preferences
- [ ] Create and link **Google Sheets OAuth2** credential to Save to Google Sheets + Read Existing Jobs
- [ ] Create and link **Gmail OAuth2** credential to Send Gmail Digest
- [ ] Update OAuth redirect URI if n8n is on a different URL/port
- [ ] Place resume PDF at `resumes/your-resume.pdf`
- [ ] Configure Docker volume mount for resumes
- [ ] Test run and verify Google Sheet + Gmail output

# Job Search Automation - Complete Setup Guide

## Prerequisites
- n8n running locally in Docker via `docker compose up -d` (from the parent n8n directory)
- SerpAPI account (free tier: 100 searches/month)
- Google Gemini API key (free tier: Gemma 3 27B -- 30 RPM, 15K TPM, 14.4K RPD)
- Google account with Sheets and Gmail access
- Your resume in PDF format (placed in the `resumes/` folder)

---

## Docker Setup

### Starting n8n

The project uses Docker Compose with a custom image. From the **parent n8n directory** (not from JobSearchAutomation/):

```bash
docker compose up -d
```

This starts n8n with:
- Custom image with FFmpeg baked in (shared with YtShortsAutomation)
- Port 5678 exposed at http://localhost:5678
- Main volume mount: `.:/home/node/.n8n` (n8n runtime data)
- Resume volume mount: `./JobSearchAutomation/resumes:/home/node/.n8n-files` (resume PDFs)
- `NODE_FUNCTION_ALLOW_BUILTIN=child_process,fs,path,os`
- `N8N_RUNNERS_TASK_TIMEOUT=900` (15-minute timeout)
- `N8N_RESTRICT_FILE_ACCESS_TO=/home/node/.n8n;/home/node/.n8n-files`

### Resume Volume Mount

The `docker-compose.yml` maps your local `JobSearchAutomation/resumes/` folder to `/home/node/.n8n-files` inside the container. This means:

- Place your resume PDF in `JobSearchAutomation/resumes/` on your host machine
- The workflow's **Read Binary File** node reads from `/home/node/.n8n-files/YourResume.pdf`
- Resumes survive container restarts because they live on the host filesystem
- The `N8N_RESTRICT_FILE_ACCESS_TO` environment variable must include `/home/node/.n8n-files` for the Read Binary File node to access it

> **Note:** If you change the volume mount path, update the Read Binary File node's file path to match.

### Rebuilding the Image

If you update the Dockerfile or want a newer n8n version:

```bash
docker compose build && docker compose up -d
```

---

## Credential 1: SerpAPI

**Used by**: Search Google Jobs nodes (Bangalore + Remote India)

### Create Account
1. Go to [serpapi.com](https://serpapi.com)
2. Click **"Register"** (free tier available, no credit card required)
3. Sign up with email or Google

### Get API Key
1. After login, go to your **Dashboard**
2. Your API key is displayed at the top of the page
3. Copy the key (a long alphanumeric string)

### Free Tier Limits
- **100 searches per month** (resets monthly)
- Each workflow run uses **2 searches** (one for Bangalore, one for Remote India)
- With dual-location search, you can run the workflow ~50 times per month on the free tier
- Paid plans start at $50/month for 5,000 searches

### Add to n8n
1. Open your imported workflow in n8n
2. Click on the **"Set Job Preferences"** node
3. Find the `serpApiKey` field in the code
4. Replace the placeholder with your actual SerpAPI API key
5. Save the workflow

The SerpAPI key is passed as a query parameter to the Search Google Jobs HTTP Request nodes automatically.

### Verify
```bash
curl "https://serpapi.com/search.json?engine=google_jobs&q=software+engineer&location=Bangalore&api_key=YOUR_KEY" | head -20
```
Should return JSON with job listings.

---

## Credential 2: Google Gemini API (Gemma 3 27B)

**Used by**: Score & Match (Gemini), Generate Cover Letter (Gemini)

**Model**: Gemma 3 27B (`gemma-3-27b-it`) -- a free-tier model available through Google's Gemini API

### Get API Key
1. Go to [aistudio.google.com/apikey](https://aistudio.google.com/apikey)
2. Sign in with your Google account
3. Click **"Create API Key"**
4. Select a Google Cloud project (or create a new one)
5. Copy the generated API key

> **Important:** Even though we use the Gemma 3 27B model (not Gemini 2.0 Flash), the API key comes from the same place: aistudio.google.com. The model is specified in the HTTP Request URL, not in the API key.

### Free Tier Limits (Gemma 3 27B)
- **30 requests per minute** (RPM)
- **15,000 tokens per minute** (TPM)
- **14,400 requests per day** (RPD)
- **No credit card required**
- The workflow uses `batchSize=1` with `batchInterval=20000ms` (20 seconds between requests) to stay well within the rate limits

### Add to n8n
1. Open your imported workflow in n8n
2. Click on the **"Set Job Preferences"** node
3. Find the `geminiApiKey` field in the code
4. Replace the placeholder with your actual Gemini API key
5. Save the workflow

The API key is passed as a query parameter to the Gemini HTTP Request nodes automatically.

### Verify
```bash
curl "https://generativelanguage.googleapis.com/v1beta/models/gemma-3-27b-it:generateContent?key=YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"contents":[{"parts":[{"text":"Hello"}]}]}'
```
Should return a JSON response with generated text.

---

## Credential 3: Google OAuth2 (Sheets + Gmail)

**Used by**: Save to Google Sheets, Send Gmail Notification

### Step 1: Create Google Cloud Project
1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Sign in with the **same Google account** you want to use for Sheets and Gmail
3. Click the project dropdown (top-left) -> **"New Project"**
4. Name: `n8n Job Search Automation` -> **Create**
5. Make sure this new project is selected in the dropdown

### Step 2: Enable Required APIs
1. Go to **APIs & Services** -> **Library** (left sidebar)
2. Search for **"Google Sheets API"** -> Click it -> **"Enable"**
3. Search for **"Gmail API"** -> Click it -> **"Enable"**

### Step 3: Configure OAuth Consent Screen
1. Go to **APIs & Services** -> **OAuth consent screen**
2. Select **"External"** -> **Create**
3. Fill in:
   - App name: `n8n Job Search`
   - User support email: your email
   - Developer contact email: your email
4. Click **"Save and Continue"**
5. **Scopes** page:
   - Click **"Add or Remove Scopes"**
   - Search and add: `https://www.googleapis.com/auth/spreadsheets` (Google Sheets read/write)
   - Search and add: `https://www.googleapis.com/auth/gmail.send` (Gmail send)
   - Click **"Update"** -> **"Save and Continue"**
6. **Test users** page (CRITICAL -- skipping this causes "Access blocked" error):
   - Click **"Add Users"**
   - Add the **exact Google email** you'll use to sign in (e.g., `your-email@gmail.com`)
   - Click **"Save and Continue"**
   - If you already skipped this step and got "Access blocked": Go to **OAuth consent screen** -> **Audience** tab -> **Add Users** -> add your email
7. Click **"Back to Dashboard"**

### Step 4: Create OAuth2 Credentials
1. Go to **APIs & Services** -> **Credentials**
2. Click **"+ Create Credentials"** -> **"OAuth client ID"**
3. Application type: **Web application**
4. Name: `n8n`
5. Under **"Authorized redirect URIs"**:
   - Click **"+ Add URI"**
   - Enter: `http://localhost:5678/rest/oauth2-credential/callback`
6. Click **"Create"**
7. Copy **Client ID** and **Client Secret** from the popup

### Step 5: Add Google Sheets Credential to n8n
1. Open your imported workflow in n8n
2. Click on the **"Save to Google Sheets"** node
3. Click **"Credential to connect with"** -> **"Create New Credential"**
4. Select **"Google Sheets OAuth2 API"**
5. Paste:
   - **Client ID**: from step 4
   - **Client Secret**: from step 4
6. Click **"Sign in with Google"**
7. Select your Google account
8. If you see "This app isn't verified" warning:
   - Click **"Advanced"** -> **"Go to n8n Job Search (unsafe)"**
   - This is safe -- it's your own app
9. Click **"Allow"** for all requested permissions
10. You should see **"Connected"** in n8n
11. Click **"Save"**

### Step 6: Add Gmail Credential to n8n
1. Click on the **"Send Gmail Digest"** node
2. Click **"Credential to connect with"** -> **"Create New Credential"**
3. Select **"Gmail OAuth2 API"**
4. Paste the **same Client ID and Client Secret** from step 4
5. Click **"Sign in with Google"**
6. Select the same Google account
7. Allow Gmail permissions
8. Click **"Save"**

### OAuth Token Refresh
- Tokens expire periodically
- n8n auto-refreshes them, but if a node fails with auth error:
  - Go to the credential in n8n
  - Click **"Sign in with Google"** again to re-authorize

---

## Google Sheet Setup

### Create the Spreadsheet
1. Go to [sheets.google.com](https://sheets.google.com)
2. Click **"Blank spreadsheet"**
3. Name the spreadsheet: **"Job Tracker"**
4. In row 1, add these exact headers in columns A through L:

| Column | Header |
|---|---|
| A | Date |
| B | Job Title |
| C | Company |
| D | Location |
| E | Match Score |
| F | Recommendation |
| G | Match Reason |
| H | Missing Skills |
| I | Apply Link |
| J | Cover Letter |
| K | Status |
| L | Applied Date |

5. **Format the headers**: Select row 1 -> Bold -> Freeze row (View -> Freeze -> 1 row)
6. **Optional formatting**:
   - Column E (Match Score): Format -> Number -> Number (0 decimal places)
   - Column J (Cover Letter): Make column wider to accommodate long text
   - Column K (Status): Add data validation dropdown with values: `New`, `Applied`, `Interview`, `Rejected`, `Offer`

> **Important:** The sheet name within the spreadsheet should be **"Job Tracker"** (the tab name at the bottom). The Save to Google Sheets node references this name. If your tab is named something else (e.g., "Sheet1"), update the node configuration to match.

### Get Spreadsheet ID
1. The URL of your spreadsheet looks like: `https://docs.google.com/spreadsheets/d/1aBcDeFgHiJkLmNoPqRsTuVwXyZ/edit`
2. The spreadsheet ID is the long string between `/d/` and `/edit`: `1aBcDeFgHiJkLmNoPqRsTuVwXyZ`
3. Copy this ID

### Configure in n8n
1. Click on the **"Set Job Preferences"** node
2. Find the `spreadsheetId` field in the code
3. Paste your spreadsheet ID
4. Save the workflow

---

## Resume Setup

### Place Your Resume PDF
1. Place your resume as a PDF file in the `resumes/` folder inside the JobSearchAutomation directory
2. Any filename ending in `.pdf` works (e.g., `resume.pdf`, `JohnDoe_Resume.pdf`)
3. The workflow's **Read Binary File** node reads the PDF from `/home/node/.n8n-files/` inside the Docker container
4. The **Extract From File** node converts the PDF to plain text automatically

### Docker Volume Mount
The `docker-compose.yml` in the parent directory includes this volume mount:

```yaml
volumes:
  - ./JobSearchAutomation/resumes:/home/node/.n8n-files
```

This means files in your local `JobSearchAutomation/resumes/` folder appear at `/home/node/.n8n-files/` inside the container. The `N8N_RESTRICT_FILE_ACCESS_TO` environment variable includes `/home/node/.n8n-files` to allow n8n's Read Binary File node to access this path.

### Update the Read Binary File Node
If your resume filename is different from what's configured:
1. Open the **"Read Binary File"** node in n8n
2. Update the file path to `/home/node/.n8n-files/YourActualFilename.pdf`
3. Save the workflow

### Tips for Better AI Matching
- Use specific technology names (e.g., ".NET Core 8" not just ".NET")
- Include years of experience with each technology
- List quantifiable achievements (e.g., "reduced latency by 40%")
- Include both the technology and the domain (e.g., "fintech", "healthcare")

### Update Target Roles and Locations
In the **"Set Job Preferences"** node, you can modify:
- `targetRoles`: Array of 17 job titles to search for
- Location configuration: Dual-location search (Bangalore onsite/hybrid + India remote by default)
- `maxResults`: 25 results per search (50 total across both locations)
- `minScore`: 50 (minimum match score to keep)

### Date Filter
The workflow uses `date_posted:week` as a search filter, which limits results to jobs posted within the last week. This keeps results fresh and avoids re-scoring old listings. You can change this in the Search Google Jobs nodes' query parameters.

---

## Post-Setup: First Test Run

### Run the Workflow
1. Click **"Test Workflow"** (play button in top-right)
2. Wait 5-15 minutes for full execution (the rate-limited HTTP calls take time)
3. Watch the execution progress -- each node lights up green when done

### Verify
- [ ] Read Binary File successfully loaded your resume PDF
- [ ] Extract From File converted the PDF to text
- [ ] SerpAPI returned job listings for both locations (check Search Google Jobs node outputs)
- [ ] Jobs were parsed and deduplicated (check Filter Duplicates node output)
- [ ] Gemma 3 27B scored each job with a match score (check Score & Match node output)
- [ ] Top matches (score >= 50) were filtered (check Filter Top Matches node output)
- [ ] Cover letters were generated for top matches (check Generate Cover Letter output)
- [ ] Results appeared in your Google Sheet with all columns populated
- [ ] You received a digest email in Gmail with the top job matches

### Common First-Run Issues
- **0 jobs found**: SerpAPI may not find jobs for very specific titles in small markets. Try broader titles like "Software Engineer" or a larger metro area.
- **All scores below 50**: Your resume may not match the found jobs well. Try lowering the threshold in the Set Job Preferences node.
- **Google Sheet empty**: Verify the spreadsheet ID and that the Google OAuth2 credential has Sheets permissions.
- **Rate limit errors (429)**: The workflow already uses batchSize=1 with 20s intervals. If you still hit limits, increase `batchInterval` in the HTTP Request nodes.
- **Resume not found**: Make sure the PDF is in `resumes/` and Docker is running with the volume mount. Check that the filename in the Read Binary File node matches.

---

## Credential Summary

| # | Credential | Type | Nodes Using It | Cost |
|---|---|---|---|---|
| 1 | SerpAPI | API Key (in Set Job Preferences code) | Search Google Jobs (Bangalore), Search Google Jobs (Remote India) | Free (100/mo) |
| 2 | Google Gemini API | API Key (in Set Job Preferences code) | Score & Match (Gemini), Generate Cover Letter (Gemini) | Free (30 RPM, 15K TPM, 14.4K RPD) |
| 3 | Google OAuth2 (Sheets) | OAuth2 | Save to Google Sheets | Free |
| 3 | Google OAuth2 (Gmail) | OAuth2 | Send Gmail Digest | Free |

---

## Troubleshooting

### "Invalid API key" on SerpAPI
- Verify the key at serpapi.com -> Dashboard
- Check you haven't exceeded the 100 searches/month free tier
- Make sure the key is in the `serpApiKey` field in Set Job Preferences

### "Invalid API key" on Gemini nodes
- Verify the key at aistudio.google.com/apikey
- Make sure the key is in the `geminiApiKey` field in Set Job Preferences
- Check you haven't exceeded the rate limits (30 RPM, 15K TPM for Gemma 3 27B)

### "OAuth token expired" on Google Sheets or Gmail
- Go to the credential in n8n -> click "Sign in with Google" again
- Make sure you added yourself as a test user in Google Cloud Console

### "Redirect URI mismatch" on Google OAuth
- In Google Cloud Console -> Credentials -> edit your OAuth client
- Make sure the redirect URI is exactly: `http://localhost:5678/rest/oauth2-credential/callback`
- No trailing slash, no https (it's localhost)

### "Access blocked: n8n Job Search has not completed Google verification"
- **Most common cause**: You didn't add yourself as a test user
- **Fix**: Google Cloud Console -> **APIs & Services** -> **OAuth consent screen** -> **Audience** tab -> **Add Users** -> add your exact Google email
- After adding, go back to n8n and click **"Sign in with Google"** again

### "This app isn't verified" warning (different from above)
- This is normal for test/unverified apps -- your app works fine
- Click **"Advanced"** -> **"Go to n8n Job Search (unsafe)"**
- This is safe -- it's your own app, not a third party

### "Spreadsheet not found" or "Sheet not found"
- Double-check the spreadsheet ID (the long string in the URL between `/d/` and `/edit`)
- Make sure the sheet tab name is **"Job Tracker"** (or matches what's configured in the Save to Google Sheets node)
- Verify your Google account has edit access to the spreadsheet

### "Insufficient permission" on Gmail
- Your Google OAuth2 credential may not have Gmail scope
- Delete the Gmail credential in n8n and recreate it
- Make sure `gmail.send` scope is enabled in your Google Cloud project

### No jobs found for certain roles
- SerpAPI Google Jobs results vary by location and market demand
- Try broader role titles (e.g., "Software Engineer" instead of "SDE 2")
- Try a larger metro area or add "remote" to the location
- Check that `date_posted:week` isn't filtering too aggressively -- try removing it temporarily

### Resume file not found / ENOENT error
- Verify the PDF is in `JobSearchAutomation/resumes/` on your host machine
- Check that Docker is running with `docker compose up -d` from the parent n8n directory
- Verify the volume mount exists: `./JobSearchAutomation/resumes:/home/node/.n8n-files`
- Check the `N8N_RESTRICT_FILE_ACCESS_TO` env var includes `/home/node/.n8n-files`
- Verify the filename in the Read Binary File node matches your actual PDF filename

### Rate limit errors (HTTP 429)
- The workflow uses `batchSize=1` with `batchInterval=20000ms` (20 seconds between requests)
- Gemma 3 27B free tier: 30 RPM, 15K TPM
- If still hitting limits, increase `batchInterval` to 15000ms or 20000ms
- Consider reducing `maxResults` from 25 to 10 per search

---

## Switching to Daily Schedule

To run the workflow automatically every day:

1. Delete the **"Manual Trigger"** node
2. Add a **"Schedule Trigger"** node
3. Configure it:
   - **Trigger Interval**: Days
   - **Days Between Triggers**: 1
   - **Trigger at Hour**: 9 (9 AM)
   - Or use cron expression: `0 9 * * *`
4. Connect the Schedule Trigger output to the **"Read Resume PDF"** node
5. **Activate** the workflow (toggle in the top-right)

With daily runs and 2 searches per run (dual-location), you'll use ~60 SerpAPI searches/month, which is within the free tier (100/mo).

---

## Migrating to a Different n8n Instance

### Method 1: Export/Import via n8n UI (Easiest)

**Export from current instance:**
1. Open your workflow in n8n
2. Click the **three dots menu** in the top-right -> **"Download"**
3. Credentials are NOT included in the export (for security)

**Import to new instance:**
1. Open the new n8n instance
2. Click **"Add workflow"** -> **"Import from file"**
3. Select the downloaded `.json` file
4. **Re-configure all 3 credentials** on the new instance
5. Update the Google Sheet spreadsheet ID in Set Job Preferences
6. Place your resume PDF in the resumes directory
7. Configure the Docker volume mount for the resumes directory
8. Set `N8N_RESTRICT_FILE_ACCESS_TO` to include the resumes mount path

### Method 2: n8n API (Programmatic)

```bash
# Export
curl -H "X-N8N-API-KEY: $N8N_API_KEY" \
  http://localhost:5678/api/v1/workflows/YOUR_WORKFLOW_ID \
  -o job-search-workflow.json

# Import to new instance
curl -X POST \
  -H "X-N8N-API-KEY: $N8N_API_KEY" \
  -H "Content-Type: application/json" \
  -d @job-search-workflow.json \
  http://your-n8n-url:5678/api/v1/workflows
```

### After Migration Checklist

On the new instance, you must:
- [ ] Add SerpAPI key to Set Job Preferences node
- [ ] Add **Gemini API key** to Set Job Preferences node
- [ ] Create and assign **Google Sheets OAuth2** credential
- [ ] Create and assign **Gmail OAuth2** credential
- [ ] Update the **Google Sheet spreadsheet ID** in Set Job Preferences
- [ ] Place your **resume PDF** in the resumes directory
- [ ] Configure the **Docker volume mount** for the resumes directory
- [ ] Set **`N8N_RESTRICT_FILE_ACCESS_TO`** to include `/home/node/.n8n-files`
- [ ] Update the **Google OAuth2 redirect URI** to match the new instance URL
- [ ] Update the **Read Binary File** node path if needed
- [ ] Test run and verify results appear in Google Sheet and Gmail

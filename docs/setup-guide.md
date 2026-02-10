# Job Search Automation - Complete Setup Guide

## Prerequisites
- n8n running locally in Docker at http://localhost:5678
- SerpAPI account (free tier: 100 searches/month)
- Google Gemini API key (free tier: 15 RPM, 1M tokens/day)
- Google account with Sheets and Gmail access
- Your resume in plain text format

---

## Credential 1: SerpAPI

**Used by**: Search Google Jobs node

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
- Each workflow run uses 1 search per target role (9 roles = 9 searches per run)
- With 9 roles, you can run the workflow ~11 times per month on the free tier
- Paid plans start at $50/month for 5,000 searches

### Add to n8n
1. Open your imported workflow in n8n
2. Click on the **"Search Google Jobs"** node
3. In the URL query parameters, find the `api_key` parameter
4. Replace `YOUR_SERPAPI_KEY` with your actual API key
5. Save the workflow

### Verify
```bash
curl "https://serpapi.com/search.json?engine=google_jobs&q=software+engineer&api_key=YOUR_KEY" | head -20
```
Should return JSON with job listings.

---

## Credential 2: Google Gemini API

**Used by**: Score & Match (Gemini), Generate Cover Letter (Gemini)

### Get API Key
1. Go to [aistudio.google.com/apikey](https://aistudio.google.com/apikey)
2. Sign in with your Google account
3. Click **"Create API Key"**
4. Select a Google Cloud project (or create a new one)
5. Copy the generated API key

### Free Tier Limits
- **15 requests per minute** (RPM)
- **1 million tokens per day**
- **No credit card required**
- More than enough for 30 daily runs (~25 Gemini calls per run)

### Add to n8n
1. Open your imported workflow in n8n
2. Click on the **"Set Job Preferences"** node
3. Find the `geminiApiKey` field in the code
4. Replace `YOUR_GEMINI_API_KEY` with your actual API key
5. Save the workflow

The API key is passed as a query parameter to the Gemini HTTP Request nodes automatically.

### Verify
```bash
curl "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=YOUR_KEY" \
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
1. Click on the **"Send Gmail Notification"** node
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

### Get Spreadsheet ID
1. The URL of your spreadsheet looks like: `https://docs.google.com/spreadsheets/d/1aBcDeFgHiJkLmNoPqRsTuVwXyZ/edit`
2. The spreadsheet ID is the long string between `/d/` and `/edit`: `1aBcDeFgHiJkLmNoPqRsTuVwXyZ`
3. Copy this ID

### Configure in n8n
1. Click on the **"Save to Google Sheets"** node
2. Find the **Document ID** or **Spreadsheet ID** field
3. Paste your spreadsheet ID
4. Set the **Sheet Name** to `Sheet1` (or whatever your sheet tab is named)
5. Save the workflow

---

## Resume Setup

### Paste Your Resume
1. In n8n, open the **"Set Job Preferences"** node
2. Find the `resumeText` field
3. Paste your complete resume as plain text
4. Include:
   - Professional summary
   - Technical skills (languages, frameworks, tools)
   - Work experience with dates and achievements
   - Education
   - Certifications (if any)

### Tips for Better AI Matching
- Use specific technology names (e.g., ".NET Core 8" not just ".NET")
- Include years of experience with each technology
- List quantifiable achievements (e.g., "reduced latency by 40%")
- Include both the technology and the domain (e.g., "fintech", "healthcare")

### Update Target Roles
In the same **"Set Job Preferences"** node, you can modify:
- `targetRoles`: Array of job titles to search for
- `targetLocation`: City, state, or "remote"

Default target roles:
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

## Post-Setup: First Test Run

### Run the Workflow
1. Click **"Test Workflow"** (play button in top-right)
2. Wait 2-5 minutes for full execution
3. Watch the execution progress -- each node lights up green when done

### Verify
- [ ] SerpAPI returned job listings (check Search Google Jobs node output)
- [ ] Jobs were parsed and deduplicated (check Filter Duplicates node output)
- [ ] Gemini scored each job with a match score (check Score & Match node output)
- [ ] Top matches (score >= 70) were filtered (check Filter Top Matches node output)
- [ ] Cover letters were generated for top matches (check Generate Cover Letter output)
- [ ] Results appeared in your Google Sheet with all columns populated
- [ ] You received a digest email in Gmail with the top job matches

### Common First-Run Issues
- **0 jobs found**: SerpAPI may not find jobs for very specific titles in small markets. Try broader titles like "Software Engineer" or a larger metro area.
- **All scores below 70**: Your resume may not match the found jobs well. Try lowering the threshold in the Filter Top Matches node to 50 for testing.
- **Google Sheet empty**: Verify the spreadsheet ID and that the Google OAuth2 credential has Sheets permissions.

---

## Credential Summary

| # | Credential | Type | Nodes Using It | Cost |
|---|---|---|---|---|
| 1 | SerpAPI | API Key (query param) | Search Google Jobs | Free (100/mo) |
| 2 | Google Gemini API | API Key (in Set Job Preferences) | Score & Match, Generate Cover Letter | Free (15 RPM, 1M tokens/day) |
| 3 | Google OAuth2 (Sheets) | OAuth2 | Save to Google Sheets | Free |
| 3 | Google OAuth2 (Gmail) | OAuth2 | Send Gmail Notification | Free |

---

## Troubleshooting

### "Invalid API key" on SerpAPI
- Verify the key at serpapi.com -> Dashboard
- Check you haven't exceeded the 100 searches/month free tier
- Make sure the key is in the URL query parameter, not in a header

### "Invalid API key" on Gemini nodes
- Verify the key at aistudio.google.com/apikey
- Make sure the key is in the `geminiApiKey` field in Set Job Preferences
- Check you haven't exceeded the 15 RPM rate limit

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
- Make sure the sheet tab name matches what's configured in n8n (default: `Sheet1`)
- Verify your Google account has edit access to the spreadsheet

### "Insufficient permission" on Gmail
- Your Google OAuth2 credential may not have Gmail scope
- Delete the Gmail credential in n8n and recreate it
- Make sure `gmail.send` scope is enabled in your Google Cloud project

### No jobs found for certain roles
- SerpAPI Google Jobs results vary by location and market demand
- Try broader role titles (e.g., "Software Engineer" instead of "SDE 2")
- Try a larger metro area or add "remote" to the location

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
4. Connect the Schedule Trigger output to the **"Set Job Preferences"** node
5. **Activate** the workflow (toggle in the top-right)

With daily runs and 9 role searches per run, you'll use ~270 SerpAPI searches/month. This exceeds the free tier (100/mo), so either:
- Reduce to 3-4 roles to stay on the free tier
- Run every 3 days instead of daily
- Upgrade to SerpAPI paid plan ($50/mo for 5,000 searches)

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
5. Update the Google Sheet spreadsheet ID
6. Paste your resume text

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
- [ ] Add SerpAPI key to the Search Google Jobs node query parameter
- [ ] Add **Gemini API key** in Set Job Preferences node
- [ ] Create and assign **Google Sheets OAuth2** credential
- [ ] Create and assign **Gmail OAuth2** credential
- [ ] Update the **Google Sheet spreadsheet ID** in the Save to Google Sheets node
- [ ] Paste your **resume text** in the Set Job Preferences node
- [ ] Update the **Google OAuth2 redirect URI** to match the new instance URL
- [ ] Test run and verify results appear in Google Sheet and Gmail

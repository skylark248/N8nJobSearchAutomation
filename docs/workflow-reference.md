# Job Search Automation - AI Matching & Cover Letters

## Overview
- **Trigger**: Manual (click to run)
- **Nodes**: 14 | **Connections**: 13
- **Estimated Runtime**: 2-5 minutes per run
- **Monthly Cost**: ~$13.50 (OpenAI) + $0 (SerpAPI free tier) + $0 (Google APIs)
- **Import file**: `exports/job-search-automation.json`

## Pipeline

```
Manual Trigger
    |
    v
[Set Job Preferences] -- target roles, location, resume text
    |
    v
[Search Google Jobs] -- SerpAPI Google Jobs engine (one query per role)
    |
    v
[Parse Job Results] -- Code node: extracts title, company, location, link
    |
    v
[Filter Duplicates] -- Code node: deduplicates by company + job title
    |
    v
[Score Job Match] -- GPT-5 scores each job 0-100 against resume
    |
    v
[Parse AI Scores] -- Code node: extracts score, recommendation, reasoning
    |
    v
[Filter Top Matches] -- IF node: score >= 70 only
    |
    v
[Generate Cover Letter] -- GPT-5 writes tailored cover letter per job
    |
    v
[Format for Sheets] -- Code node: adds date, status columns, formats data
    |
    v
[Save to Google Sheets] -- Appends rows to "Job Tracker" spreadsheet
    |
    v
[Build Email Digest] -- Code node: builds HTML summary of new matches
    |
    v
[Send Gmail Notification] -- Sends digest email with top job matches
    |
    v
[Done Output] -- Returns summary stats (jobs found, matches, emails sent)
```

## Node Details

### Node 1: Manual Trigger
- **Type**: `n8n-nodes-base.manualTrigger` (v1)
- **ID**: `trigger`
- **Config**: Default (no parameters)
- **Notes**: Replace with Schedule Trigger (`0 9 * * *`) for daily automation.

### Node 2: Set Job Preferences
- **Type**: `n8n-nodes-base.set` (v3.4)
- **ID**: `setPreferences`

**Parameters**:
- `targetRoles`: Array of job titles to search:
  - Backend Developer
  - Senior Backend Developer
  - .NET Developer
  - Senior .NET Developer
  - Full Stack Developer
  - Senior Full Stack Developer
  - Software Engineer 2
  - SDE 2
  - Senior Software Engineer
- `targetLocation`: Target city/state or "remote" (e.g., `"United States"`)
- `resumeText`: Full resume as plain text (paste your resume here)
- `minScore`: Minimum match score threshold (default: `70`)

### Node 3: Search Google Jobs
- **Type**: `n8n-nodes-base.httpRequest` (v4.2)
- **ID**: `searchJobs`

**Parameters**:
```json
{
  "url": "https://serpapi.com/search.json",
  "sendQuery": true,
  "queryParameters": {
    "parameters": [
      { "name": "engine", "value": "google_jobs" },
      { "name": "q", "value": "={{ $json.targetRoles.join(' OR ') }}" },
      { "name": "location", "value": "={{ $json.targetLocation }}" },
      { "name": "api_key", "value": "YOUR_SERPAPI_KEY" }
    ]
  },
  "options": { "timeout": 15000 }
}
```

**SETUP REQUIRED**: Replace `YOUR_SERPAPI_KEY` with your actual SerpAPI key.

### Node 4: Parse Job Results
- **Type**: `n8n-nodes-base.code` (v2)
- **ID**: `parseJobs`

**Code summary**: Extracts job listings from SerpAPI response. Maps each result to a normalized object with fields: `title`, `company`, `location`, `description`, `applyLink`, `source`. Handles missing fields gracefully.

### Node 5: Filter Duplicates
- **Type**: `n8n-nodes-base.code` (v2)
- **ID**: `filterDuplicates`

**Code summary**: Deduplicates jobs by creating a composite key from `company + title` (lowercased, trimmed). Keeps only the first occurrence of each unique job. Logs how many duplicates were removed.

### Node 6: Score Job Match
- **Type**: `n8n-nodes-base.openAi` (v1)
- **ID**: `scoreMatch`
- **Credential**: OpenAI API

**Parameters**:
- **Model**: GPT-5
- **System prompt**: Instructs GPT-5 to act as a technical recruiter, compare the job description against the candidate's resume, and return a JSON object with: `score` (0-100), `recommendation` (Strong Match / Good Match / Weak Match / No Match), `matchReason` (why the candidate fits), `missingSkills` (gaps to address).
- **User message**: Contains the job title, company, description, and the candidate's resume text.

### Node 7: Parse AI Scores
- **Type**: `n8n-nodes-base.code` (v2)
- **ID**: `parseScores`

**Code summary**: Parses the GPT-5 JSON response. Extracts `score`, `recommendation`, `matchReason`, and `missingSkills`. Falls back to regex extraction if the response is not clean JSON. Merges parsed fields with the original job data.

### Node 8: Filter Top Matches
- **Type**: `n8n-nodes-base.if` (v2)
- **ID**: `filterMatches`

**Condition**: `{{ $json.score }}` is greater than or equal to `70`

- **True output** (score >= 70): Continues to cover letter generation
- **False output** (score < 70): Discarded (not saved or emailed)

### Node 9: Generate Cover Letter
- **Type**: `n8n-nodes-base.openAi` (v1)
- **ID**: `generateCoverLetter`
- **Credential**: OpenAI API

**Parameters**:
- **Model**: GPT-5
- **System prompt**: Instructs GPT-5 to write a concise, professional cover letter (250-350 words) tailored to the specific job. References specific skills from the resume that match the job requirements. Uses a confident but not arrogant tone.
- **User message**: Contains the job title, company, job description, match reasoning, and the candidate's resume.

### Node 10: Format for Sheets
- **Type**: `n8n-nodes-base.code` (v2)
- **ID**: `formatSheets`

**Code summary**: Formats each job match into a row for Google Sheets. Maps fields to columns:
- Date: current date (YYYY-MM-DD)
- Job Title: from parsed job data
- Company: from parsed job data
- Location: from parsed job data
- Match Score: from AI scoring
- Recommendation: from AI scoring
- Match Reason: from AI scoring
- Missing Skills: from AI scoring
- Apply Link: from parsed job data
- Cover Letter: from GPT-5 generation
- Status: "New" (default)
- Applied Date: empty (to be filled manually)

### Node 11: Save to Google Sheets
- **Type**: `n8n-nodes-base.googleSheets` (v4.5)
- **ID**: `saveToSheets`
- **Credential**: Google Sheets OAuth2 API

**Parameters**:
- **Operation**: Append Row
- **Document ID**: Your spreadsheet ID (SETUP REQUIRED)
- **Sheet Name**: `Sheet1`
- **Mapping Mode**: Map each column manually to the corresponding field

**SETUP REQUIRED**: Replace the Document ID with your actual Google Sheet spreadsheet ID.

### Node 12: Build Email Digest
- **Type**: `n8n-nodes-base.code` (v2)
- **ID**: `buildDigest`

**Code summary**: Builds an HTML email summarizing all top matches from this run. Includes:
- Run date and total jobs found
- Table of matches with: Job Title, Company, Location, Score, Recommendation
- Direct apply links for each job
- Link to the Google Sheet for full details

### Node 13: Send Gmail Notification
- **Type**: `n8n-nodes-base.gmail` (v2.1)
- **ID**: `sendEmail`
- **Credential**: Gmail OAuth2 API

**Parameters**:
- **To**: Your email address
- **Subject**: `Job Search Digest - {{ $now.format('yyyy-MM-dd') }} - {{ $json.matchCount }} matches`
- **Email Type**: HTML
- **Message**: HTML digest from Build Email Digest node

### Node 14: Done Output
- **Type**: `n8n-nodes-base.code` (v2)
- **ID**: `doneOutput`

**Code summary**: Returns a summary object with:
- `totalJobsFound`: Number of jobs returned by SerpAPI
- `duplicatesRemoved`: Number of duplicate jobs filtered
- `jobsScored`: Number of jobs sent to GPT-5 for scoring
- `topMatches`: Number of jobs with score >= 70
- `coverLettersGenerated`: Number of cover letters written
- `sheetRowsAdded`: Number of rows appended to Google Sheet
- `emailSent`: Boolean indicating digest email was sent
- `timestamp`: ISO timestamp of completion

## Connection Map

```
Source Node                  -> Target Node                  | Type
--------------------------------------------------------------------------------
Manual Trigger               -> Set Job Preferences           | main
Set Job Preferences          -> Search Google Jobs            | main
Search Google Jobs           -> Parse Job Results             | main
Parse Job Results            -> Filter Duplicates             | main
Filter Duplicates            -> Score Job Match               | main
Score Job Match              -> Parse AI Scores               | main
Parse AI Scores              -> Filter Top Matches            | main
Filter Top Matches           -> Generate Cover Letter         | main (true)
Generate Cover Letter        -> Format for Sheets             | main
Format for Sheets            -> Save to Google Sheets         | main
Save to Google Sheets        -> Build Email Digest            | main
Build Email Digest           -> Send Gmail Notification       | main
Send Gmail Notification      -> Done Output                   | main
```

## Required Credentials

| # | Credential Name | Type | Where to Get | Used By |
|---|---|---|---|---|
| 1 | SerpAPI | API Key (query param) | serpapi.com -> Dashboard -> API Key | Search Google Jobs |
| 2 | OpenAI API | API Key | platform.openai.com -> API Keys | Score Job Match, Generate Cover Letter |
| 3 | Google Sheets OAuth2 | OAuth2 | Google Cloud Console -> Sheets API -> OAuth2 | Save to Google Sheets |
| 4 | Gmail OAuth2 | OAuth2 | Google Cloud Console -> Gmail API -> OAuth2 (same project as Sheets) | Send Gmail Notification |

## Google Sheet Schema

| Column | Header | Source | Example |
|---|---|---|---|
| A | Date | Format for Sheets node | 2026-02-09 |
| B | Job Title | SerpAPI result | Senior Backend Developer |
| C | Company | SerpAPI result | Microsoft |
| D | Location | SerpAPI result | Redmond, WA |
| E | Match Score | GPT-5 scoring | 87 |
| F | Recommendation | GPT-5 scoring | Strong Match |
| G | Match Reason | GPT-5 scoring | 8+ years .NET, distributed systems experience... |
| H | Missing Skills | GPT-5 scoring | Azure Kubernetes Service, Terraform |
| I | Apply Link | SerpAPI result | https://careers.microsoft.com/... |
| J | Cover Letter | GPT-5 generation | Dear Hiring Manager, I am writing to express... |
| K | Status | Default value | New |
| L | Applied Date | Manual entry | (empty until you apply) |

## Modification Guide

| What | How |
|---|---|
| Change target job titles | Edit "Set Job Preferences" node -- update the `targetRoles` array |
| Change location | Edit "Set Job Preferences" node -- update `targetLocation` (e.g., `"San Francisco, CA"`, `"Remote"`, `"London, UK"`) |
| Change minimum match score | Edit "Filter Top Matches" node -- change the condition threshold from 70 to your desired minimum |
| Switch to daily schedule | Replace "Manual Trigger" with "Schedule Trigger" node, cron: `0 9 * * *` for daily at 9 AM |
| Add more job sources | Add HTTP Request nodes for Indeed API, LinkedIn API, etc. Merge results before Filter Duplicates using a Merge node |
| Change AI model | Edit "Score Job Match" and "Generate Cover Letter" nodes -- update the model parameter (e.g., `gpt-4o`, `gpt-5`) |
| Change email recipient | Edit "Send Gmail Notification" node -- update the To field |
| Add Slack notifications | Add a Slack node after "Send Gmail Notification" -- post digest to a channel |
| Change cover letter style | Edit "Generate Cover Letter" system prompt -- adjust tone, length, or format |
| Lower API costs | Use `gpt-4o-mini` instead of GPT-5 for scoring (less accurate but ~10x cheaper) |
| Increase job results | Add `num` parameter to SerpAPI query (default: 10, max: 100 per search) |
| Track application status | Use the "Status" and "Applied Date" columns in Google Sheet -- update manually or add a form |

## Version History
- **v1.0** (2026-02-09): Initial creation. 14 nodes, manual trigger, SerpAPI + GPT-5 + Google Sheets + Gmail. Target roles: Backend, .NET, Full Stack, Software Engineer variants. Match scoring with cover letter generation.

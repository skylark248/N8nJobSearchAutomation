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
[Set Job Preferences] -- Code node: target roles, location, resume text, minScore
    |
    v
[Search Google Jobs (SerpAPI)] -- HTTP Request to SerpAPI Google Jobs endpoint
    |
    v
[Parse Job Results] -- Code node: extracts title, company, location, link
    |
    v
[Filter Duplicates] -- Code node: deduplicates by company + job title
    |
    v
[Score & Match (GPT-5)] -- OpenAI node: scores each job 0-100 against resume
    |
    v
[Parse AI Score] -- Code node: extracts score, recommendation, reasoning
    |
    v
[Filter Top Matches] -- Code node: score >= 70 only
    |
    v
[Generate Cover Letter (GPT-5)] -- OpenAI node: tailored cover letter per job
    |
    v
[Attach Cover Letter] -- Code node: merges cover letter with job data
    |
    v
[Save to Google Sheets] -- Appends rows to "Job Tracker" spreadsheet
    |
    v
[Build Email Digest] -- Code node: builds HTML summary of new matches
    |
    v
[Send Gmail Digest] -- Sends digest email with top job matches
    |
    v
[Success Summary] -- Returns summary stats (jobs found, matches, emails sent)
```

## Node Details

### Node 1: Manual Trigger
- **Type**: `n8n-nodes-base.manualTrigger` (v1)
- **ID**: `trigger`
- **Config**: Default (no parameters)
- **Notes**: Replace with Schedule Trigger (`0 9 * * *`) for daily automation.

### Node 2: Set Job Preferences
- **Type**: `n8n-nodes-base.code` (v2)
- **ID**: `setPreferences`

**Purpose**: Central config node -- user edits this to change search criteria.

**Output fields**:
- `searchQuery`: All target roles joined with OR operator
- `location`: Target location (default: "India")
- `minScore`: Minimum match score threshold (default: 70)
- `resumeText`: Full resume as plain text (paste your resume here)
- `yourEmail`: Gmail address for digest emails
- `spreadsheetId`: Google Sheet ID for saving results

**Target roles** (joined into searchQuery):
- Backend Developer
- Senior Backend Developer
- .NET Developer
- Senior .NET Developer
- Full Stack Developer
- Senior Full Stack Developer
- Software Engineer 2
- SDE 2
- Senior Software Engineer

### Node 3: Search Google Jobs (SerpAPI)
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
      { "name": "q", "value": "={{ $json.searchQuery }}" },
      { "name": "location", "value": "={{ $json.location }}" },
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

**Code summary**: Extracts `jobs_results` from SerpAPI response. Maps each result to a normalized object with fields: `title`, `company`, `location`, `description` (truncated to 3000 chars), `applyLink` (from `apply_options[0].link` or `share_link`), `source`. Generates `jobId` hash from company+title (lowercased, trimmed). Handles missing fields gracefully.

### Node 5: Filter Duplicates
- **Type**: `n8n-nodes-base.code` (v2)
- **ID**: `filterDuplicates`

**Code summary**: Deduplicates jobs by creating a composite key from `company + title` (lowercased, trimmed). Keeps only the first occurrence of each unique job. Outputs individual items (not array) for per-job processing downstream.

### Node 6: Score & Match (GPT-5)
- **Type**: `n8n-nodes-base.openAi` (v1)
- **ID**: `scoreMatch`
- **Credential**: OpenAI API

**Parameters**:
- **Model**: GPT-5
- **System prompt**: Instructs GPT-5 to act as a job matching expert. Compares the job description against the candidate's resume. Returns JSON with: `score` (0-100), `recommendation` (strong-apply/apply/maybe/skip), `matchReason` (2-3 sentences), `missingSkills` (array), `keyMatchingSkills` (array), `experienceFit` (under-qualified/good-fit/over-qualified).
- **User message**: Contains the job title, company, description, and the candidate's resume text. References resume via `$('Set Job Preferences').first().json.resumeText`.

### Node 7: Parse AI Score
- **Type**: `n8n-nodes-base.code` (v2)
- **ID**: `parseScore`

**Code summary**: Parses the GPT-5 JSON response with try/catch. Falls back to regex extraction (`/\{[\s\S]*\}/`) if the response is wrapped in markdown code blocks. Merges parsed score fields with the original job data from `$('Filter Duplicates').item`. Output includes all original job fields plus score, recommendation, matchReason, missingSkills, keyMatchingSkills.

### Node 8: Filter Top Matches
- **Type**: `n8n-nodes-base.code` (v2)
- **ID**: `filterMatches`

**Code summary**: Returns the item if `score >= minScore` (default 70), otherwise returns empty `[]`. Jobs below threshold are silently dropped, saving API costs on cover letter generation.

### Node 9: Generate Cover Letter (GPT-5)
- **Type**: `n8n-nodes-base.openAi` (v1)
- **ID**: `generateCoverLetter`
- **Credential**: OpenAI API

**Parameters**:
- **Model**: GPT-5
- **System prompt**: Instructs GPT-5 to write a concise, professional cover letter (250-350 words) tailored to the specific job. References specific skills from the resume that match the job requirements. Addresses to "Hiring Manager" at the company. Focuses on .NET, React, backend architecture, and job-specific technologies. Responds with ONLY the cover letter text (no JSON, no markdown).
- **User message**: Contains the job title, company, job description, key matching skills, and the candidate's resume.

### Node 10: Attach Cover Letter
- **Type**: `n8n-nodes-base.code` (v2)
- **ID**: `addCoverLetter`

**Code summary**: Merges the GPT-5 cover letter text with the existing job data. Extracts the cover letter from the OpenAI response (`content` or `message.content`). Outputs the combined object with all job fields plus `coverLetter`.

### Node 11: Save to Google Sheets
- **Type**: `n8n-nodes-base.googleSheets` (v4.5)
- **ID**: `saveToSheet`
- **Credential**: Google Sheets OAuth2 API

**Parameters**:
- **Operation**: `appendOrUpdate`
- **Document ID**: Your spreadsheet ID (SETUP REQUIRED)
- **Sheet Name**: `Sheet1`
- **Mapping Mode**: `defineBelow` -- maps each column manually to the corresponding field
- **Resource locator**: Uses `__rl: true` with `mode: "id"` format

**12 columns mapped**: Date, Job Title, Company, Location, Match Score, Recommendation, Match Reason, Missing Skills, Apply Link, Cover Letter, Status ("Not Applied"), Applied Date (empty)

**SETUP REQUIRED**: Replace the Document ID with your actual Google Sheet spreadsheet ID.

### Node 12: Build Email Digest
- **Type**: `n8n-nodes-base.code` (v2)
- **ID**: `buildEmail`

**Code summary**: Builds an HTML email summarizing all top matches from this run. Includes:
- Gradient header (#667eea to #764ba2) with title
- Stats section showing total matches and top score
- Job table with color-coded scores: green (>=85), yellow (>=70), red (<70)
- Apply buttons as HTML links for each job
- Expandable cover letters using `<details>` HTML tags
- Link to the Google Sheet for full details
- Outputs `{ html, matchCount, topScore }`

### Node 13: Send Gmail Digest
- **Type**: `n8n-nodes-base.gmail` (v2.1)
- **ID**: `sendDigest`
- **Credential**: Gmail OAuth2 API

**Parameters**:
- **Operation**: `send` (REQUIRED -- missing this causes validation failure)
- **To**: Your email address (from Set Job Preferences `yourEmail`)
- **Subject**: `Job Search Update: {{ matchCount }} new matches found (Top: {{ topScore }}/100)`
- **Email Type**: `html`
- **Message**: HTML digest from Build Email Digest node

### Node 14: Success Summary
- **Type**: `n8n-nodes-base.code` (v2)
- **ID**: `successOutput`

**Code summary**: Returns a summary object with:
- `success`: true
- `matchCount`: Number of jobs with score >= 70
- `topScore`: Highest match score found
- `emailSent`: Boolean indicating digest email was sent
- `timestamp`: ISO timestamp of completion

## Connection Map

```
Source Node                  -> Target Node                  | Type
--------------------------------------------------------------------------------
Manual Trigger               -> Set Job Preferences           | main
Set Job Preferences          -> Search Google Jobs (SerpAPI)  | main
Search Google Jobs (SerpAPI) -> Parse Job Results             | main
Parse Job Results            -> Filter Duplicates             | main
Filter Duplicates            -> Score & Match (GPT-5)         | main
Score & Match (GPT-5)        -> Parse AI Score                | main
Parse AI Score               -> Filter Top Matches            | main
Filter Top Matches           -> Generate Cover Letter (GPT-5) | main
Generate Cover Letter (GPT-5)-> Attach Cover Letter           | main
Attach Cover Letter          -> Save to Google Sheets         | main
Save to Google Sheets        -> Build Email Digest            | main
Build Email Digest           -> Send Gmail Digest             | main
Send Gmail Digest            -> Success Summary               | main
```

## Required Credentials

| # | Credential Name | Type | Where to Get | Used By |
|---|---|---|---|---|
| 1 | SerpAPI | API Key (query param) | serpapi.com -> Dashboard -> API Key | Search Google Jobs (SerpAPI) |
| 2 | OpenAI API | API Key | platform.openai.com -> API Keys | Score & Match (GPT-5), Generate Cover Letter (GPT-5) |
| 3 | Google Sheets OAuth2 | OAuth2 | Google Cloud Console -> Sheets API -> OAuth2 | Save to Google Sheets |
| 4 | Gmail OAuth2 | OAuth2 | Google Cloud Console -> Gmail API -> OAuth2 (same project as Sheets) | Send Gmail Digest |

## Google Sheet Schema

| Column | Header | Source | Example |
|---|---|---|---|
| A | Date | Attach Cover Letter node | 2026-02-09 |
| B | Job Title | SerpAPI result | Senior Backend Developer |
| C | Company | SerpAPI result | Microsoft |
| D | Location | SerpAPI result | Bengaluru, India |
| E | Match Score | GPT-5 scoring | 87 |
| F | Recommendation | GPT-5 scoring | strong-apply |
| G | Match Reason | GPT-5 scoring | 8+ years .NET, distributed systems experience... |
| H | Missing Skills | GPT-5 scoring | Azure Kubernetes Service, Terraform |
| I | Apply Link | SerpAPI result | https://careers.microsoft.com/... |
| J | Cover Letter | GPT-5 generation | Dear Hiring Manager, I am writing to express... |
| K | Status | Default value | Not Applied |
| L | Applied Date | Manual entry | (empty until you apply) |

## Modification Guide

| What | How |
|---|---|
| Change target job titles | Edit "Set Job Preferences" node -- update the role titles in searchQuery |
| Change location | Edit "Set Job Preferences" node -- update `location` value (e.g., `"San Francisco, CA"`, `"Remote"`, `"London, UK"`) |
| Change minimum match score | Edit "Filter Top Matches" node -- change the condition threshold from 70 to your desired minimum |
| Switch to daily schedule | Replace "Manual Trigger" with "Schedule Trigger" node, cron: `0 9 * * *` for daily at 9 AM |
| Add more job sources | Add HTTP Request nodes for Indeed API, LinkedIn API, etc. Merge results before Filter Duplicates using a Merge node |
| Change AI model | Edit "Score & Match (GPT-5)" and "Generate Cover Letter (GPT-5)" nodes -- update the model parameter (e.g., `gpt-4o`, `gpt-5`) |
| Change email recipient | Edit "Set Job Preferences" node -- update `yourEmail` field |
| Add Slack notifications | Add a Slack node after "Send Gmail Digest" -- post digest to a channel |
| Change cover letter style | Edit "Generate Cover Letter (GPT-5)" system prompt -- adjust tone, length, or format |
| Lower API costs | Use `gpt-4o-mini` instead of GPT-5 for scoring (less accurate but ~10x cheaper) |
| Increase job results | Add `num` parameter to SerpAPI query (default: 10, max: 100 per search) |
| Track application status | Use the "Status" and "Applied Date" columns in Google Sheet -- update manually or add a form |

## Version History
- **v1.0** (2026-02-09): Initial creation. 14 nodes, manual trigger, SerpAPI + GPT-5 + Google Sheets + Gmail. Target roles: Backend, .NET, Full Stack, Software Engineer variants. Match scoring with cover letter generation.

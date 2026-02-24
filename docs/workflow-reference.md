# Job Search Automation - AI Matching & Cover Letters

## Overview
- **Trigger**: Manual (click to run)
- **Nodes**: 16 | **Connections**: 15
- **Estimated Runtime**: 3-8 minutes per run (depends on job count and API rate limits)
- **Monthly Cost**: $0 (all free tiers) -- Gemma 3 27B + SerpAPI + Google APIs
- **Import file**: `exports/job-search-automation.json`

## Pipeline

```
Manual Trigger
    |
    v
[Read Resume PDF] -- Reads PDF binary from /home/node/.n8n-files/
    |
    v
[Extract PDF Text] -- Extracts text from PDF (joinPages: true)
    |
    v
[Set Job Preferences] -- Code node: reads resume from PDF, dual-location, 17 roles, minScore 50
    |                      Outputs 2 items: Bangalore onsite/hybrid + India remote
    v
[Search Google Jobs (SerpAPI)] -- HTTP Request to SerpAPI (runs once per item = 2 searches)
    |
    v
[Parse Job Results] -- Code node: processes $input.all(), merges both search results
    |
    v
[Filter Duplicates] -- Code node: emits individual items from aggregated array
    |
    v
[Score & Match (Gemini)] -- HTTP Request: Gemma 3 27B scores each job 0-100
    |                        Batching: 1 req / 20s. retryOnFail: 3 tries, 15s wait
    v
[Parse AI Score] -- Code node (runOnceForEachItem): extracts score via $input.item
    |
    v
[Filter Top Matches] -- Code node (runOnceForAllItems): score >= 50 via $input.all()
    |
    v
[Generate Cover Letter (Gemini)] -- HTTP Request: Gemma 3 27B cover letter
    |                                Batching: 1 req / 20s. retryOnFail: 3 tries, 15s wait
    v
[Attach Cover Letter] -- Code node (runOnceForEachItem): merges via $input.item
    |
    v
[Save to Google Sheets] -- Appends rows to "Job Tracker" spreadsheet
    |
    v
[Build Email Digest] -- Code node: builds HTML summary, maps Sheets + camelCase field names
    |
    v
[Send Gmail Digest] -- Sends digest email with top job matches
    |
    v
[Success Summary] -- Returns summary stats (jobs found, matches, emails sent)
```

## Node Details

### Node 1: Read Resume PDF
- **Type**: `n8n-nodes-base.readBinaryFile` (v1)
- **ID**: `readResumePdf`

**Purpose**: Reads the resume PDF from the volume-mounted resumes directory.

**Parameters**:
```json
{
  "filePath": "/home/node/.n8n-files/GarimaResume.pdf"
}
```

**Notes**:
- The path `/home/node/.n8n-files/` maps to `./JobSearchAutomation/resumes/` on the host via docker-compose volume mount
- n8n enforces `N8N_RESTRICT_FILE_ACCESS_TO=/home/node/.n8n;/home/node/.n8n-files` -- files outside these paths are blocked
- To change resume: place a new PDF in `resumes/` and update the `filePath` parameter

### Node 2: Extract PDF Text
- **Type**: `n8n-nodes-base.extractFromFile` (v1.1)
- **ID**: `extractPdfText`

**Purpose**: Extracts text content from the PDF binary data.

**Parameters**:
```json
{
  "operation": "pdf",
  "options": {
    "joinPages": true
  }
}
```

**Output**: `{ text: "full resume text from all pages concatenated..." }`

### Node 3: Manual Trigger
- **Type**: `n8n-nodes-base.manualTrigger` (v1)
- **ID**: `trigger`
- **Config**: Default (no parameters)
- **Notes**: Replace with Schedule Trigger (`0 9 * * *`) for daily automation.

### Node 4: Set Job Preferences
- **Type**: `n8n-nodes-base.code` (v2)
- **ID**: `setPreferences`

**Purpose**: Central config node. Reads resume from PDF extraction, defines search criteria, API keys, and preferences.

**Code**:
```javascript
// === READ RESUME FROM PDF ===
const resumeText = $input.first().json.text;

if (!resumeText || resumeText.trim().length < 50) {
  throw new Error('Resume PDF appears empty or too short. Make sure you placed a valid PDF resume in the resumes/ folder.');
}

// === EDIT THESE SETTINGS ===
const baseConfig = {
  minScore: 50,
  maxResults: 25,
  yourEmail: 'YOUR_EMAIL@gmail.com',
  spreadsheetId: 'YOUR_GOOGLE_SHEET_ID',
  geminiApiKey: 'YOUR_GEMINI_API_KEY',
  serpApiKey: 'YOUR_SERPAPI_KEY',
  resumeText: resumeText.trim()
};

const roles = '"Backend Developer" OR "Senior Backend Developer" OR ".NET Developer" OR "Full Stack Developer" OR "Software Engineer 2" OR "SDE 2" OR "Senior Software Engineer" OR "Senior Full Stack Developer" OR "C# Developer" OR "ASP.NET Developer" OR "Software Engineer" OR "Backend Engineer" OR "Application Developer" OR "Software Developer" OR "AI Platform Engineer" OR "Backend Engineer AI" OR "Gen AI Engineer"';

// Two searches: Bangalore onsite/hybrid + India remote
return [
  { json: { ...baseConfig, searchQuery: roles + ' onsite OR hybrid', location: 'Bangalore, Karnataka, India' } },
  { json: { ...baseConfig, searchQuery: roles + ' remote', location: 'India' } }
];
```

**Output fields** (per item):
- `searchQuery`: Target roles joined with OR + location modifier (onsite/hybrid or remote)
- `location`: "Bangalore, Karnataka, India" or "India"
- `geminiApiKey`: Google Gemini API key for Gemma 3 27B
- `serpApiKey`: SerpAPI key for Google Jobs search
- `minScore`: Minimum match score threshold (default: 50)
- `maxResults`: Max jobs per search (default: 25)
- `resumeText`: Full resume text extracted from PDF
- `yourEmail`: Gmail address for digest emails
- `spreadsheetId`: Google Sheet ID for saving results

**Target roles** (17 total, joined into searchQuery):
- Backend Developer, Senior Backend Developer
- .NET Developer, C# Developer, ASP.NET Developer
- Full Stack Developer, Senior Full Stack Developer
- Software Engineer 2, SDE 2, Senior Software Engineer, Software Engineer
- Backend Engineer, Backend Engineer AI
- Application Developer, Software Developer
- AI Platform Engineer, Gen AI Engineer

### Node 5: Search Google Jobs (SerpAPI)
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
      { "name": "chips", "value": "date_posted:week" },
      { "name": "api_key", "value": "={{ $json.serpApiKey }}" }
    ]
  },
  "options": { "timeout": 15000 }
}
```

**Notes**:
- Runs once per input item from Set Job Preferences (2 searches total)
- `chips=date_posted:week` filters to jobs posted in the last week
- SerpAPI key is read from Set Job Preferences (no hardcoded credential)

### Node 6: Parse Job Results
- **Type**: `n8n-nodes-base.code` (v2)
- **ID**: `parseJobs`

**Code**:
```javascript
const allItems = $input.all();
const config = $('Set Job Preferences').first().json;
let allJobs = [];

for (const item of allItems) {
  const jobs = item.json.jobs_results || [];
  const parsed = jobs.slice(0, config.maxResults || 20).map(job => {
    const raw = (job.company_name || '') + '|' + (job.title || '');
    let hash = 0;
    for (let i = 0; i < raw.length; i++) {
      const char = raw.charCodeAt(i);
      hash = ((hash << 5) - hash) + char;
      hash = hash & hash;
    }
    const jobId = 'JOB_' + Math.abs(hash).toString(36);

    let applyLink = '';
    if (job.apply_options && job.apply_options.length > 0) {
      applyLink = job.apply_options[0].link || '';
    } else if (job.share_link) {
      applyLink = job.share_link;
    }

    return {
      jobId,
      title: job.title || 'Unknown Title',
      company: job.company_name || 'Unknown Company',
      location: job.location || 'Not specified',
      description: (job.description || '').substring(0, 3000),
      postedDate: job.detected_extensions?.posted_at || 'Unknown',
      applyLink,
      source: job.via || 'Google Jobs',
      salary: job.detected_extensions?.salary || 'Not listed',
      schedule: job.detected_extensions?.schedule_type || ''
    };
  });
  allJobs.push(...parsed);
}

if (allJobs.length === 0) {
  throw new Error('No jobs found. Try broadening your search query or changing location.');
}

return { json: { jobs: allJobs, totalFound: allJobs.length } };
```

**Key details**:
- Uses `$input.all()` to process both Bangalore and India remote search responses
- Generates `jobId` hash from `company|title` for dedup tracking
- Truncates `description` to 3000 chars for AI context window
- Extracts salary, schedule, posted date from `detected_extensions`
- Returns single item with `{ jobs: [...], totalFound: N }`

### Node 7: Filter Duplicates
- **Type**: `n8n-nodes-base.code` (v2)
- **ID**: `filterDuplicates`

**Code**:
```javascript
const data = $input.first().json;
const jobs = data.jobs || [];
return jobs.map(job => ({ json: job }));
```

**Key details**: Emits each job as an individual n8n item for per-job processing downstream. Deduplication across the two location searches is handled by the jobId hash generated in Parse Job Results.

### Node 8: Score & Match (Gemini)
- **Type**: `n8n-nodes-base.httpRequest` (v4.2)
- **ID**: `scoreMatch`

**Parameters**:
- **URL**: `https://generativelanguage.googleapis.com/v1beta/models/gemma-3-27b-it:generateContent`
- **Method**: POST
- **API Key**: Passed as query parameter `key` from `$('Set Job Preferences').first().json.geminiApiKey`
- **Body** (JSON): Single `contents` array with one user message containing:
  - System instructions (merged into user message -- Gemma 3 27B does not support `systemInstruction`)
  - Candidate resume text
  - Job posting details (title, company, location, description)
  - Output format instructions (JSON with score, matchReason, missingSkills, keyMatchingSkills, experienceFit, recommendation)
- **Generation config**: `temperature: 0.7`
- **Batching**: `batchSize: 1`, `batchInterval: 20000` (one request every 20 seconds)
- **retryOnFail**: `maxTries: 3`, `waitBetweenTries: 15000`
- **Timeout**: 30000ms

**Important -- Gemma 3 27B limitations**:
- Does NOT support `systemInstruction` field -- causes 400 error. System prompt must be merged into the user message text.
- Does NOT support `responseMimeType: "application/json"` -- cannot force structured JSON output. Instead, instruct "respond with ONLY valid JSON" in the prompt text.

**JSON body structure**:
```json
{
  "contents": [
    {
      "role": "user",
      "parts": [
        {
          "text": "You are a job matching expert... [system prompt] ... CANDIDATE RESUME: [resume] ... JOB POSTING: [details] ... Respond with ONLY valid JSON: { score, matchReason, missingSkills, keyMatchingSkills, experienceFit, recommendation }"
        }
      ]
    }
  ],
  "generationConfig": {
    "temperature": 0.7
  }
}
```

### Node 9: Parse AI Score
- **Type**: `n8n-nodes-base.code` (v2)
- **ID**: `parseScore`
- **Mode**: `runOnceForEachItem`

**Code**:
```javascript
// Gemini response: candidates[0].content.parts[0].text
const response = $input.item.json;
const aiResponse = response.candidates?.[0]?.content?.parts?.[0]?.text || '';
const jobData = $('Filter Duplicates').item;

let parsed;
try {
  parsed = JSON.parse(aiResponse);
} catch (e) {
  const jsonMatch = aiResponse.match(/\{[\s\S]*\}/);
  if (jsonMatch) {
    parsed = JSON.parse(jsonMatch[0]);
  } else {
    parsed = {
      score: 0,
      matchReason: 'Could not parse AI response',
      missingSkills: [],
      keyMatchingSkills: [],
      experienceFit: 'unknown',
      recommendation: 'skip'
    };
  }
}

return {
  json: {
    ...jobData.json,
    score: parsed.score || 0,
    matchReason: parsed.matchReason || '',
    missingSkills: parsed.missingSkills || [],
    keyMatchingSkills: parsed.keyMatchingSkills || [],
    experienceFit: parsed.experienceFit || 'unknown',
    recommendation: parsed.recommendation || 'skip'
  }
};
```

**Key details**:
- Uses `runOnceForEachItem` mode so each AI response is processed individually
- `$input.item.json` accesses the Gemini response for the current item
- `$('Filter Duplicates').item` gets the original job data at the same index (same-index cross-reference)
- Falls back to regex extraction (`/\{[\s\S]*\}/`) if JSON.parse fails (handles markdown code blocks in response)

### Node 10: Filter Top Matches
- **Type**: `n8n-nodes-base.code` (v2)
- **ID**: `filterMatches`
- **Mode**: `runOnceForAllItems` (default)

**Code**:
```javascript
const items = $input.all();
const minScore = $('Set Job Preferences').first().json.minScore || 70;

const matched = items.filter(item => (item.json.score || 0) >= minScore);

if (matched.length === 0) {
  throw new Error(`No jobs scored above ${minScore}. Top score was ${Math.max(...items.map(i => i.json.score || 0))}. Try lowering minScore in Set Job Preferences.`);
}

return matched;
```

**Key details**:
- Uses `$input.all()` to get all scored items at once
- Filters by `minScore` (default: 50, configured in Set Job Preferences)
- Throws descriptive error if no jobs pass threshold (includes the actual top score)
- Jobs below threshold are dropped, saving API costs on cover letter generation

### Node 11: Generate Cover Letter (Gemini)
- **Type**: `n8n-nodes-base.httpRequest` (v4.2)
- **ID**: `generateCoverLetter`

**Parameters**:
- **URL**: `https://generativelanguage.googleapis.com/v1beta/models/gemma-3-27b-it:generateContent`
- **Method**: POST
- **API Key**: Passed as query parameter from `$('Set Job Preferences').first().json.geminiApiKey`
- **Body**: Single user message with system instructions merged in (Gemma 3 27B limitation). Includes:
  - Instructions to write a concise, professional cover letter under 300 words
  - Candidate resume text
  - Job details: title, company, location, description, key matching skills
  - Explicit instruction: "Respond with ONLY the cover letter text, no JSON, no markdown"
- **Generation config**: `temperature: 0.7`
- **Batching**: `batchSize: 1`, `batchInterval: 20000` (one request every 20 seconds)
- **retryOnFail**: `maxTries: 3`, `waitBetweenTries: 15000`
- **Timeout**: 30000ms

### Node 12: Attach Cover Letter
- **Type**: `n8n-nodes-base.code` (v2)
- **ID**: `addCoverLetter`
- **Mode**: `runOnceForEachItem`

**Code**:
```javascript
const jobData = $('Filter Top Matches').item;
const response = $input.item.json;
const coverLetter = response.candidates?.[0]?.content?.parts?.[0]?.text || 'Cover letter generation failed.';

return {
  json: {
    ...jobData.json,
    coverLetter: coverLetter.trim(),
    processedDate: new Date().toISOString().split('T')[0]
  }
};
```

**Key details**:
- Uses `runOnceForEachItem` mode so each cover letter is paired with the correct job
- `$input.item.json` accesses the Gemini cover letter response for the current item
- `$('Filter Top Matches').item` gets the original job data at the same index
- Adds `processedDate` for the Google Sheets Date column

### Node 13: Save to Google Sheets
- **Type**: `n8n-nodes-base.googleSheets` (v4.5)
- **ID**: `saveToSheet`
- **Credential**: Google Sheets OAuth2 API

**Parameters**:
- **Operation**: `append`
- **Document ID**: Spreadsheet ID (configured in Set Job Preferences)
- **Sheet Name**: `Job Tracker`
- **Mapping Mode**: `defineBelow` -- maps each column manually to the corresponding field
- **Resource locator**: Uses `__rl: true` with `mode: "id"` / `mode: "name"` format

**12 columns mapped**:

| Column | Header | Expression |
|---|---|---|
| A | Date | `={{ $json.processedDate }}` |
| B | Job Title | `={{ $json.title }}` |
| C | Company | `={{ $json.company }}` |
| D | Location | `={{ $json.location }}` |
| E | Match Score | `={{ $json.score }}` |
| F | Recommendation | `={{ $json.recommendation }}` |
| G | Match Reason | `={{ $json.matchReason }}` |
| H | Missing Skills | `={{ $json.missingSkills.join(', ') }}` |
| I | Apply Link | `={{ $json.applyLink }}` |
| J | Cover Letter | `={{ $json.coverLetter }}` |
| K | Status | `Not Applied` |
| L | Applied Date | (empty) |

### Node 14: Build Email Digest
- **Type**: `n8n-nodes-base.code` (v2)
- **ID**: `buildEmail`

**Code summary**: Builds an HTML email summarizing all top matches from this run. Maps both Google Sheets column names and camelCase field names for robustness (Google Sheets returns column headers as keys after the append step).

**Field mapping pattern**:
```javascript
const sorted = items.map(i => {
  const j = i.json;
  return {
    title: j['Job Title'] || j.title || 'Unknown',
    company: j['Company'] || j.company || 'Unknown',
    score: Number(j['Match Score'] || j.score || 0),
    // ... maps both "Column Header" and camelCase forms
  };
}).sort((a, b) => b.score - a.score);
```

**HTML features**:
- Gradient header (#667eea to #764ba2) with title and date
- Stats section showing total matches and top score
- Job table with color-coded scores: green (>=85), yellow (>=70), red (<70)
- Apply buttons as HTML links for each job
- Expandable cover letters using `<details>` HTML tags
- Footer with min score threshold

**Output**: `{ subject, htmlBody, matchCount, topScore, yourEmail }`

### Node 15: Send Gmail Digest
- **Type**: `n8n-nodes-base.gmail` (v2.1)
- **ID**: `sendDigest`
- **Credential**: Gmail OAuth2 API

**Parameters**:
- **To**: `={{ $json.yourEmail }}` (from Build Email Digest output, originating from Set Job Preferences)
- **Subject**: `={{ $json.subject }}` (includes match count and top score)
- **Message**: `={{ $json.htmlBody }}` (HTML digest from Build Email Digest)

**Notes**: Gmail v2.1 sends HTML by default. The subject line includes match count and top score.

### Node 16: Success Summary
- **Type**: `n8n-nodes-base.code` (v2)
- **ID**: `successOutput`

**Code**:
```javascript
const emailData = $input.first().json;

return {
  json: {
    success: true,
    message: 'Job search automation completed!',
    matchCount: emailData.matchCount || 0,
    topScore: emailData.topScore || 0,
    emailSent: true,
    sentTo: emailData.yourEmail || 'unknown',
    timestamp: new Date().toISOString()
  }
};
```

## Connection Map

```
Source Node                    -> Target Node                     | Type
----------------------------------------------------------------------------------
Manual Trigger                 -> Read Resume PDF                  | main
Read Resume PDF                -> Extract PDF Text                 | main
Extract PDF Text               -> Set Job Preferences              | main
Set Job Preferences            -> Search Google Jobs (SerpAPI)     | main
Search Google Jobs (SerpAPI)   -> Parse Job Results                | main
Parse Job Results              -> Filter Duplicates                | main
Filter Duplicates              -> Score & Match (Gemini)           | main
Score & Match (Gemini)         -> Parse AI Score                   | main
Parse AI Score                 -> Filter Top Matches               | main
Filter Top Matches             -> Generate Cover Letter (Gemini)   | main
Generate Cover Letter (Gemini) -> Attach Cover Letter              | main
Attach Cover Letter            -> Save to Google Sheets            | main
Save to Google Sheets          -> Build Email Digest               | main
Build Email Digest             -> Send Gmail Digest                | main
Send Gmail Digest              -> Success Summary                  | main
```

## Required Credentials

| # | Credential Name | Type | Where to Get | Used By |
|---|---|---|---|---|
| 1 | SerpAPI | API Key (in Set Job Preferences) | serpapi.com -> Dashboard -> API Key | Search Google Jobs (SerpAPI) |
| 2 | Google Gemini API | API Key (in Set Job Preferences) | aistudio.google.com/apikey | Score & Match (Gemini), Generate Cover Letter (Gemini) |
| 3 | Google Sheets OAuth2 | OAuth2 | Google Cloud Console -> Sheets API -> OAuth2 | Save to Google Sheets |
| 4 | Gmail OAuth2 | OAuth2 | Google Cloud Console -> Gmail API -> OAuth2 (same project as Sheets) | Send Gmail Digest |

## Google Sheet Schema

| Column | Header | Source | Example |
|---|---|---|---|
| A | Date | Attach Cover Letter node | 2026-02-24 |
| B | Job Title | SerpAPI result | Senior Backend Developer |
| C | Company | SerpAPI result | Microsoft |
| D | Location | SerpAPI result | Bengaluru, India |
| E | Match Score | Gemma 3 27B scoring (0-100) | 87 |
| F | Recommendation | Gemma 3 27B (strong-apply/apply/maybe/skip) | strong-apply |
| G | Match Reason | Gemma 3 27B (2-3 sentences) | 8+ years .NET, distributed systems experience... |
| H | Missing Skills | Gemma 3 27B (comma-separated) | Azure Kubernetes Service, Terraform |
| I | Apply Link | SerpAPI result | https://careers.microsoft.com/... |
| J | Cover Letter | Gemma 3 27B generation (under 300 words) | Dear Hiring Manager, I am writing to express... |
| K | Status | Default value | Not Applied |
| L | Applied Date | Manual entry | (empty until you apply) |

## Modification Guide

| What | How |
|---|---|
| Change resume | Place new PDF in `resumes/` folder, update `filePath` in Read Resume PDF node |
| Change target job titles | Edit "Set Job Preferences" node -- update the role titles in the `roles` string |
| Change location | Edit "Set Job Preferences" node -- update the `location` values in the return array |
| Add/remove a search location | Edit "Set Job Preferences" node -- add or remove items from the return array |
| Change minimum match score | Edit "Set Job Preferences" node -- change `minScore` value (currently 50) |
| Switch to daily schedule | Replace "Manual Trigger" with "Schedule Trigger" node, cron: `0 9 * * *` for daily at 9 AM |
| Add more job sources | Add HTTP Request nodes for Indeed API, LinkedIn API, etc. Merge results before Filter Duplicates using a Merge node |
| Change AI model | Edit "Score & Match (Gemini)" and "Generate Cover Letter (Gemini)" nodes -- change the model in the URL. Note: if switching back to Gemini Flash, you can re-add `systemInstruction` and `responseMimeType` |
| Change email recipient | Edit "Set Job Preferences" node -- update `yourEmail` field |
| Add Slack notifications | Add a Slack node after "Send Gmail Digest" -- post digest to a channel |
| Change cover letter style | Edit "Generate Cover Letter (Gemini)" JSON body -- adjust tone, length, or format in the user message |
| Increase job results | Change `maxResults` in Set Job Preferences (default: 25) |
| Adjust rate limiting | Change `batchInterval` in Score & Match and Generate Cover Letter nodes (currently 20000ms) |
| Track application status | Use the "Status" and "Applied Date" columns in Google Sheet -- update manually or add a form |

## Version History
- **v3.0** (2026-02-24): Switched to Gemma 3 27B (from Gemini 2.0 Flash). Added PDF resume reading (Read Resume PDF + Extract PDF Text, 16 nodes total). Dual-location search: Bangalore onsite/hybrid + India remote. Expanded to 17 target roles. Lowered minScore from 70 to 50. Added HTTP batching (batchSize=1, batchInterval=20000ms) and retryOnFail (maxTries=3, waitBetweenTries=15000ms) on AI nodes. SerpAPI key moved into Set Job Preferences. Volume mount for resumes: `./JobSearchAutomation/resumes:/home/node/.n8n-files`.
- **v2.0** (2026-02-09): Replaced GPT-5 (OpenAI) with Gemini 2.0 Flash (Google). Cost reduced from ~$13.50/mo to $0. Location updated to Bangalore + remote. Added `responseMimeType: "application/json"` for structured JSON output.
- **v1.0** (2026-02-09): Initial creation. 14 nodes, manual trigger, SerpAPI + GPT-5 + Google Sheets + Gmail. Target roles: Backend, .NET, Full Stack, Software Engineer variants.

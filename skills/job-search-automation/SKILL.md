---
name: job-search-automation
description: Quick reference for the n8n Job Search Automation workflow. Use when modifying, debugging, or extending this specific workflow. Covers node IDs, architecture, common operations, code patterns, and gotchas.
---

# Job Search Automation — Quick Reference

Workflow ID: `8vs888j57KeLaGtH` | 27 nodes | 26 connections | n8n: http://localhost:5678

---

## Architecture

```
Read Resume PDF → Extract PDF Text → Set Job Preferences (config hub)
                                              ↓         ↓         ↓
                                        JSearch(7)  Naukri.NET  Naukri C#
                                              ↓         ↓         ↓
                                       Merge Search Results (append)
                                                    ↓
                                           Parse Job Results
                                                    ↓
                                           Filter Duplicates
                                                    ↓
                                           Aggregate Jobs ──────────┐
                                                    ↓               ↓
                                           Sync Dedup Inputs ← Read Existing Jobs
                                                    ↓
                                           Filter New Jobs Only
                                                    ↓
                                    Prepare Score Request → Score OpenAI API → Parse AI Score
                                                    ↓
                                           Filter Top Matches (score ≥ 75)
                                                    ↓
                                  Prepare Cover Letter → Cover Letter OpenAI API → Attach Cover Letter
                                                    ↓
                                           Save to Google Sheets
                                                    ↓
                                           Build Email Digest → Send Gmail → Success Summary
```

---

## Node IDs (use for MCP updateNode)

| Node Name | nodeId |
|---|---|
| Set Job Preferences | `setPreferences` |
| Search Google Jobs (JSearch) | `searchJobs` |
| Set Naukri Queries | `setNaukriQueries` |
| Search Naukri (Apify) | `searchNaukri` |
| Set Naukri Queries (C#) | `setNaukriQueriesCSharp` |
| Search Naukri C# (Apify) | `searchNaukriCSharp` |
| Merge Search Results | `mergeSearchResults` |
| Parse Job Results | `parseJobs` |
| Filter Duplicates | `filterDuplicates` |
| Aggregate Jobs | `aggregateJobs` |
| Read Existing Jobs | `readExistingJobs` |
| Sync Dedup Inputs | `syncDedup` |
| Filter New Jobs Only | `filterNewJobs` |
| Prepare Score Request | `scoreMatch` |
| Score OpenAI API | `scoreGroqApi` |
| Parse AI Score | `parseScore` |
| Filter Top Matches | `filterMatches` |
| Prepare Cover Letter | `generateCoverLetter` |
| Cover Letter OpenAI API | `coverLetterGroqApi` |
| Attach Cover Letter | `addCoverLetter` |
| Save to Google Sheets | `saveToSheet` |
| Build Email Digest | `buildEmail` |
| Send Gmail Digest | `sendDigest` |
| Success Summary | `successOutput` |

---

## Common Operations

### Change minScore
Update `setPreferences` node jsCode: `minScore: 75` → new value.
```bash
n8n_update_partial_workflow with updateNode on nodeId: "setPreferences"
```

### Add a new JSearch query
In `setPreferences` jsCode, add to the return array:
```javascript
{ json: { ...baseConfig, searchQuery: 'YOUR QUERY Bangalore India' } }
```

### Change AI model
In `setPreferences` jsCode, change `model: 'gpt-4o-mini'` in both scoring and cover letter payloads. Also update both HTTP Request nodes (`scoreGroqApi`, `coverLetterGroqApi`) if switching provider.

### Update resume
Place new PDF at `resumes/your-resume.pdf`. File is volume-mounted to `/home/node/.n8n-files/your-resume.pdf`.

### Switch to schedule trigger
Replace Manual Trigger with Schedule Trigger:
```json
{ "type": "n8n-nodes-base.scheduleTrigger", "typeVersion": 1.2,
  "parameters": { "rule": { "interval": [{ "triggerAtHour": 9, "triggerAtMinute": 0 }] } } }
```

---

## Config Values (all in setPreferences)

| Key | Current Value | Notes |
|---|---|---|
| `minScore` | 75 | Filter threshold. AI scores this profile 75-85. |
| `openAiApiKey` | sk-proj-... | Must include sk-proj- prefix |
| `rapidApiKey` | ... | RapidAPI JSearch, free 200 req/month |
| `apifyToken` | ... | Apify, within $5/month credit |
| `yourEmail` | ...@gmail.com | Gmail digest recipient |
| `spreadsheetId` | ... | Google Sheets doc ID from URL |

---

## Proven Execution Results (Execution #70)

| Stage | Count |
|---|---|
| JSearch raw | ~140 |
| Naukri .NET (Apify) | ~20 |
| Naukri C# (Apify) | ~40 |
| After parse + dedup | 211 unique |
| After cross-run dedup | 211 new (first run) |
| AI scored | 211 |
| Score ≥ 75 | 71 final matches |
| Saved to Sheets | 71 |
| Runtime | ~21 min |
| Cost/run | ~$0.076 |

---

## Critical Gotchas

### Code nodes have no HTTP client
`fetch`, `this.helpers.httpRequest`, `$helpers` are all undefined in Code node sandbox.
**Pattern**: Code node builds `aiPayload = JSON.stringify({...})`, HTTP Request node sends it.

### runOnceForEachItem references
- Parse AI Score → `$('Prepare Score Request').item`
- Attach Cover Letter → `$('Prepare Cover Letter').item`
Always reference the immediately preceding Prep node.

### Why Aggregate Jobs exists
Without it, Read Existing Jobs runs ~180 times (once per job) → 429 rate limit from Sheets.

### Why Sync Dedup Inputs exists
Without it, empty sheet → Read Existing Jobs outputs 0 rows → n8n skips Filter New Jobs → nothing scored.

### Naukri Apify response (snake_case)
- Apply link: `apply_link` (NOT `jobUrl`)
- Posted date: `posted` (NOT `datePosted`)
- Response is a top-level JSON array (not wrapped in `data[]`)

### OpenAI json_validate_failed 400
Don't embed JSON examples in system prompt when `response_format: json_object` is set. Use plain field descriptions only.

### PDF null char artifacts
`extractFromFile` produces `\u0000` in place of `+`, `%`, `~`, `()`.
Fix: `rawText.replace(/\u0000/g, '')` in setPreferences.

---

## MCP Update Pattern

```
n8n_update_partial_workflow({
  workflowId: "8vs888j57KeLaGtH",
  operations: [{
    type: "updateNode",
    nodeId: "<nodeId>",
    updates: { parameters: { jsCode: "..." } }
  }]
})
```
Use `"updates"` key, NOT `"properties"`. Use `nodeId`, NOT `name`.

---

## Cost Summary

| Service | Cost/run | Cost/month (4 runs) |
|---|---|---|
| JSearch (RapidAPI) | $0 | $0 (free 200 req/mo) |
| Apify Naukri | ~$0.01 | $0 (within $5 credit) |
| OpenAI scoring (~211 jobs) | ~$0.060 | ~$0.24 |
| OpenAI cover letters (~71 jobs) | ~$0.016 | ~$0.06 |
| **Total** | **~$0.076** | **~$0.30** |

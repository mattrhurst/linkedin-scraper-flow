# linkedin-scraper-flow
n8n workflow that scrapes LinkedIn job postings on a schedule, deduplicates against Supabase, scores them with AI against your resume, and emails alerts for high matches. Uses Apify for scraping, Gemini for scoring, and Google Sheets for tracking.

# LinkedIn Scraper 2.0 - Workflow Guide

## Overview

This n8n workflow automatically scrapes LinkedIn job postings daily, deduplicates them against previously seen jobs, scores each new job against your resume using AI, and sends email alerts for high-match opportunities. All results are logged to a Google Sheet for tracking.

**Schedule**: Runs Monday-Friday at 6:00 AM Pacific time.

### What You'll Need

- **n8n** (self-hosted or cloud) - this is the automation platform everything runs on
- **Apify account** - handles the actual LinkedIn scraping ($49/mo plan works fine)
- **Google account** - for Sheets (data storage), Drive (resume), and Gmail (alerts)
- **A database** - Supabase free tier works, or any DB you already have
- **An LLM API** - OpenRouter, OpenAI, Anthropic, etc. (used for resume-job scoring)
- **Your resume** as a PDF uploaded to Google Drive

## How It Works (High Level)

1. Reads a list of LinkedIn search URLs from a Google Sheet
2. For each active URL, scrapes job listings via Apify
3. Deduplicates against a database of previously seen job IDs
4. Saves new jobs to Google Sheets
5. Sends each new job through an AI scoring sub-workflow (see companion guide)
6. Emails you about the results - one of three outcomes per search URL:
   - **High-match digest** - jobs scoring 75+ with scores, summaries, and action buttons
   - **No match notification** - workflow ran, but nothing scored high enough
   - **Duplicates only** - all jobs from this search were already seen
7. After all URLs are processed, sends a completion summary email

## Architecture

```
Schedule (6AM M-F)
  |
  v
Read Job URLs from Sheet --> Filter Active URLs --> Loop Over URLs
                                                        |
                                                   [per URL]
                                                        |
                                                   Set Random Delay (10-50 min)
                                                        |
                                                   Wait
                                                        |
                                                   Download Resume (Google Drive)
                                                        |
                                                   Extract Resume Text
                                                        |
                                                   Scrape LinkedIn (Apify)
                                                        |
                                              +---------+---------+
                                              |                   |
                                         Update "Last Run"   Extract Incoming IDs
                                         in JobURLs sheet         |
                                                             Query DB for Existing IDs
                                                                  |
                                                             Filter Duplicates
                                                                  |
                                                          Has New Jobs?
                                                         /            \
                                                       YES             NO
                                                      /                  \
                                              Insert IDs to DB     Format "Duplicates Only" Email
                                              Add to Google Sheet       |
                                                      |           Send Duplicates Email
                                              Enrich with Resume        |
                                                      |           [next URL]
                                              AI Score Sub-Workflow
                                                      |
                                              Aggregate Scores
                                                      |
                                              Has High Scores? (>=75)
                                                /            \
                                              YES             NO
                                             /                  \
                                     Split High Scores     Format "No Match" Email
                                     Format Email HTML          |
                                     Sort by Score         Send No Match Email
                                     Combine into Digest        |
                                     Send Alert Email      [next URL]
                                            |
                                       [next URL]

  [All URLs done]
        |
  Format Completion Email --> Send Completion Email
```

## Node-by-Node Breakdown

### Trigger & Setup

**Schedule Trigger - Daily 6 AM**
- Cron: `0 6 * * 1-5` (6 AM Mon-Fri, Pacific timezone)
- Kicks off the entire workflow each weekday morning

**Read Job URLs from Sheet**
- Reads from the "JobURLs" tab of your Google Sheet
- Each row has: URL (LinkedIn search URL), Label (friendly name), Active (boolean), Last Run (timestamp)
- This is where you manage which searches run
- Important to note that this ONLY works with the older LinkedIn style search, not the new AI Beta. You'll know which one you are using based on if there are filters at the top or not.

**Filter Active URLs**
- Filters rows where `Active = true`
- Lets you pause/resume individual searches without deleting them

**Loop Over URLs**
- Iterates through each active URL one at a time
- Output 0 (loop done): triggers completion email
- Output 1 (each item): processes that URL

### Anti-Detection Delay

**Set Delay**
- Generates a random delay between 10-50 minutes
- Formula: `Math.floor(Math.random() * 40) + 10`

**Wait**
- Pauses execution for the random number of minutes
- Prevents hitting LinkedIn/Apify with rapid sequential requests
- Makes the scraping pattern look more natural. This is an additional protection to ensure you aren't scraping LinkedIn too fast. It can be turned off.

### Resume Handling

**Schedule Download Resume**
- Downloads your resume PDF from Google Drive
- Runs once per loop iteration (not per job)
- The resume is used later for AI scoring

**Schedule Extract Resume**
- Extracts plain text from the PDF
- This text gets passed to the AI scoring prompt

### LinkedIn Scraping

**Get LinkedIn Jobs**
- Uses the Apify "Advanced LinkedIn Job Scraper" actor
- Configured with:
  - LinkedIn session cookies (for authenticated access - you'll export these from your browser)
  - Residential proxy (US-based, to avoid IP blocks)
  - Up to 2,000 results per search URL
  - Company data scraping enabled (pulls company description for better AI scoring context)
- Retry: up to 2 attempts with 3s between tries
- The search URL comes from the current loop item
- Apify handles pagination, proxy rotation, and rate limiting on the LinkedIn side

### Deduplication

**Update row in sheet**
- Writes the "Last Run" timestamp back to the JobURLs tab
- So you can see when each search was last executed

**Extract Incoming IDs**
- Collects all job IDs from the scrape into a single array
- Passes them as a comma-separated list for the database lookup

**Get Existing Job IDs**
- Queries the database (Supabase) for any of the incoming job IDs
- Uses an `in` filter: `linkedin_job_id=in.(id1,id2,...)`
- Returns only the IDs that already exist

> **Note on database choice**: Supabase is used here, but any database works (Postgres, MySQL, Airtable, etc.). Google Sheets was originally used for dedup but hit API rate limits with high job volumes. Any DB with a simple key-value table avoids those limits.

**Filter Duplicates**
- JavaScript code node that compares incoming IDs vs existing IDs
- Filters out any jobs already in the database
- If all jobs are duplicates, returns a single item with `_empty: true`
- Attaches the search URL and label to each job for downstream use

**If (has new jobs?)**
- True path: new jobs found - proceed to save and score
- False path: all duplicates - send a "duplicates only" notification

### Saving New Jobs

**Insert row (Supabase)**
- Inserts each new job's LinkedIn ID into the dedup database
- This ensures the same job won't be processed again tomorrow

**Add jobs to Google Sheet**
- Appends raw job data to the "Jobs" tab
- Fields: id, Date Posted, Job Title, Company, Job URL, Work Type, Location, Salary Min/Max, Description, Apply URL, Company Description

**Enrich Items**
- Attaches the resume text, search label, and scraped date to each job item
- These fields are needed by the AI scoring sub-workflow

### AI Scoring

**Execute AI Score Jobs (Batched) Sub-Flow**
- Calls the separate "AI Score LinkedIn Jobs (Batched)" workflow
- Passes all enriched job items
- Waits for the sub-workflow to complete and return scored results
- See the separate guide for details on how scoring works
- This can be built all into one workflow, but the risk is running into API call issues.

### Score Routing

**Aggregate & Check Scores**
- Splits returned jobs into high-score (>=75) and low-score (<70)
- Returns counts and arrays for both groups

**Has High Scores?**
- True path: at least one job scored 75+
- False path: no jobs hit the threshold

### High-Score Email Path

**Split Out High Score Jobs**
- Converts the aggregated high-score array back into individual items

**Format Email HTML**
- Generates a styled HTML card for each high-score job, including:
  - Match score (e.g., "Score: 82/100") with Easy Apply badge if applicable
  - Job title, company, location, work type, salary range
  - Score breakdown across all 6 categories
  - AI-generated strengths, gaps, and fit summary
  - Link back to the LinkedIn search that found this job
- Includes two action buttons per job (these hit n8n webhooks you can wire up separately):
  - "Generate Prep" - triggers interview prep generation for that specific job
  - "Save Job" - saves the job to a separate tracker/pipeline

**Sort by Score**
- Sorts jobs by match score, highest first

**Combine High-Score Jobs**
- Aggregates all formatted HTML into a single item for the digest email

**Send High-Score Alert**
- Sends a digest email with all high-match jobs
- Subject: "High-Match Jobs for [date] (Score 75+)"
- Shows total count and all jobs sorted by score

### No-Match Email Path

**Format No Match Email**
- Generates an HTML email noting no jobs scored 75+
- Shows total jobs processed and the search link

**Send No Match Email**
- Sends the no-match notification

### Duplicates-Only Email Path

**Format Duplicates Email**
- Generates an HTML email noting all jobs were already seen
- Shows jobs processed count and the search link

**Send Duplicates Email**
- Sends the duplicates-only notification

### Completion

**Format Completed Email**
- After all URLs have been processed, generates a summary email
- Shows: finish time, total URLs processed, link to the Google Sheet

**Send Completed Message**
- Sends the completion summary (runs once, not per URL)

## External Services & Credentials Required

| Service | Purpose | Credential Type |
|---------|---------|----------------|
| **Apify** | LinkedIn job scraping (Advanced LinkedIn Job Scraper actor) | API key |
| **Google Sheets** | Job URL config, raw job storage, scored results | OAuth2 |
| **Google Drive** | Resume PDF storage and download | OAuth2 |
| **Supabase** (or any DB) | Job ID deduplication | API key |
| **Gmail** | Email notifications (4 types) | OAuth2 |

## Google Sheet Structure

The workflow uses a single Google Sheet with multiple tabs:

- **JobURLs** tab: Your search configuration
  - `URL` - LinkedIn search URL
  - `Label` - Friendly name for the search
  - `Active` - TRUE/FALSE toggle
  - `Last Run` - Auto-updated timestamp

- **Jobs** tab: Raw scraped job data (one row per job)

- **Sheet2** tab: AI-scored results with match scores, breakdowns, and summaries (written by the sub-workflow)

## Setup Instructions

1. **Apify account**: Sign up at apify.com. Find the "Advanced LinkedIn Job Scraper" actor (`gdbRh93zn42kBYDyS`). You'll need your LinkedIn session cookies - export them from your browser while logged into LinkedIn.

2. **Google Sheet**: Create a sheet with tabs named "JobURLs", "Jobs", and "Sheet2". The JobURLs tab needs columns: URL, Label, Active, Last Run. Add your LinkedIn search URLs with Active=TRUE.

3. **Resume**: Upload your resume PDF to Google Drive and note the file ID.

4. **Database**: Create a Supabase project (or use any DB). Create a table called `job-id` with a `linkedin_job_id` text column.

5. **n8n credentials**: Set up OAuth2 credentials for Google Sheets, Google Drive, and Gmail. Add API credentials for Apify and Supabase.

6. **Update the workflow**: Replace all `[YOUR_*]` placeholders in the JSON with your actual IDs and credentials.

7. **Import the sub-workflow first**: The main workflow calls "AI Score LinkedIn Jobs (Batched)" - import that workflow and note its ID, then update the Execute Workflow node.

## Key Design Decisions

- **Random delay between URLs**: 10-50 minutes per URL to avoid detection and rate limiting from LinkedIn/Apify. Can be reduced or removed if you're only running a few searches.
- **Resume re-download each iteration**: Ensures the latest resume version is always used for scoring. Update your resume in Google Drive and the next run picks it up automatically.
- **Supabase for dedup instead of Sheets**: Google Sheets API rate limits made lookups unreliable at scale. Originally this was all Google Sheets, but once you're checking hundreds of job IDs per run, you hit quota issues. Any DB with a simple key-value table works.
- **Sub-workflow for AI scoring**: Keeps the main workflow cleaner and avoids n8n execution issues with long-running AI calls. Could be inlined into one workflow, but separating it reduces the risk of API timeout or memory issues killing the entire run.
- **Three notification types**: You always know what happened per search URL - high matches, no matches, or no new jobs at all. No silent failures.
- **Completion email**: Confirms the entire run finished successfully across all search URLs, with a link to the results sheet.
- **Error workflow**: Configured to trigger a separate error-handling workflow on failure, so you'll know if something breaks.

## Tips & Gotchas

- **LinkedIn cookies expire.** You'll need to re-export them periodically (roughly every 1-2 months). If the scraper starts returning 0 results, check your cookies first.
- **The 75-point threshold is adjustable.** Edit the "Aggregate & Check Scores" code node if you want to raise or lower what counts as a "high match."
- **Low-score jobs aren't lost.** Everything still gets written to Google Sheets - you just won't get an email alert for jobs below 75.
- **Search URL format matters.** This only works with LinkedIn's classic search results, not the newer AI-powered search beta. You'll know you're on the right one if you see standard filter dropdowns at the top of the search results page.
- **The action buttons in emails are webhooks.** They point to separate n8n webhook URLs. You'll need to set up those receiving workflows separately, or remove the buttons from the Format Email HTML node if you don't need them.


# AI Score LinkedIn Jobs (Batched) - Sub-Workflow Guide

## Overview

This is a sub-workflow called by the main "LinkedIn Scraper 2.0" workflow. It receives new LinkedIn job listings (already enriched with resume text), scores each one against your resume using AI, writes the scored results to a Google Sheet, and returns everything back to the parent workflow for email alerting.

Jobs are processed in batches of 40 with a 10-second cooldown between batches to avoid overwhelming the AI API.

### Why a Separate Workflow?

This is split out from the main scraper for reliability. AI scoring can involve hundreds of LLM calls per run, and keeping it in a separate workflow:
- Prevents API timeouts or memory issues from killing the entire scraper run
- Makes it easier to test and iterate on the scoring logic independently
- Keeps the main workflow focused on scraping and routing

You could inline this into the main workflow if you prefer - just move these nodes directly after the "Enrich Items" node. The trade-off is that a failure in scoring would stop the whole pipeline.

## How It Works (High Level)

1. Receives job items from the parent workflow (each item includes job data + resume text)
2. Splits items into batches of 40
3. For each batch: normalizes fields, sends to AI for scoring, parses the response, writes to Google Sheet
4. After all batches are processed, returns the full set of scored results to the parent

## Architecture

```
[Called by parent workflow]
        |
        v
Execute Workflow Trigger (receives job items)
        |
        v
Split in Batches (40 per batch)
        |
   [per batch]
        |
        v
Merge Job + Resume Data (normalize field names)
        |
        v
AI Score Resume (LangChain Agent)
  |-- Gemini Flash 2.5 Lite (via OpenRouter)
        |
        v
Parse AI Response (extract JSON, format salary/scores)
        |
        v
Add to Google Sheet (Sheet2 tab - scored results)
        |
        v
Store Results (accumulate in workflow static data)
        |
        v
Wait For Batch (10 second cooldown)
        |
   [next batch]

   [All batches done]
        |
        v
Return Results (flush static data, return to parent)
```

## Node-by-Node Breakdown

### Trigger

**Execute Workflow Trigger**
- Type: passthrough (receives all items from parent workflow)
- Each incoming item contains: job ID, title, company, location, description, salary, work type, apply URL, resume text, search label, scraped date

### Batching

**Split in Batches**
- Batch size: 40 jobs per batch
- Output 0 (loop done): triggers Return Results
- Output 1 (each batch): processes through AI scoring
- The batch size balances throughput vs API rate limits

### Field Normalization

**Merge Job + Resume Data**
- Maps incoming fields to clean, consistent names
- Input fields may come with Google Sheets-style names (e.g., "Job Title", "Is It In-Person, Remote or Hybrd")
- Outputs normalized fields: jobId, jobTitle, company, location, description, workType, salaryMin, salaryMax, resumeText, jobUrl, applyUrl, postedAt, Company Description, searchLabel, scrapedDate

### AI Scoring

**Gemini Flash 2.5 Lite**
- LLM model node connected to the AI agent
- Uses OpenRouter as the provider
- Model: `google/gemini-2.5-flash-lite`

> **Note on LLM choice**: Gemini Flash 2.5 Lite was chosen specifically for cost efficiency. This workflow scores potentially hundreds of jobs daily, so per-token cost matters. Any LLM that can return structured JSON works here - you can swap this for GPT-4o-mini, Claude Haiku, Llama, etc. Just change the model node; the prompt stays the same.

**AI Score Resume**
- LangChain Agent node with a detailed scoring prompt
- Sends the full job posting (title, company, location, description, company description) plus the candidate's entire resume as plain text
- Asks the AI to score on a 100-point scale across 6 weighted categories:

| Category | Max Points | What It Measures |
|----------|-----------|------------------|
| Skills/Tools overlap | 40 | How well your tech stack matches the job requirements |
| Relevant experience & seniority | 25 | Years of experience and level alignment |
| Responsibilities alignment | 15 | How closely your past work maps to the day-to-day |
| Education/Certifications fit | 5 | Degree and certification requirements |
| Domain/Industry fit | 5 | Industry-specific experience relevance |
| Location/logistics fit | 10 | Remote/hybrid/onsite and relocation considerations |

- Skills/Tools gets the heaviest weight (40 points) because it's the strongest signal for whether you'd actually be qualified
- The AI returns a structured JSON object with:
  - `match_score` (0-100) - the weighted total
  - `score_breakdown` (per-category scores, e.g., skills_overlap: 35, experience_match: 20)
  - `key_strengths` (top 3 reasons you're a good fit)
  - `key_gaps` (top 2 areas where you fall short)
  - `quick_summary` (2-3 sentence plain-English assessment)
- Retry: up to 2 attempts on failure
- The prompt explicitly asks for JSON-only output (no markdown, no explanation) to make parsing reliable

### Response Processing

**Parse AI Response**
- JavaScript code node that extracts and validates the AI's JSON output
- Handles common LLM quirks:
  - Strips markdown code fences (` ```json ... ``` `) if the model wraps its response
  - Finds the JSON object between the first `{` and last `}` to ignore any preamble text
  - Falls back to zeroed scores if parsing fails entirely (workflow keeps running, job just gets a 0)
- Formats salary into human-readable strings:
  - Both min and max: "$120,000 - $160,000"
  - Only min: "$120,000+"
  - Only max: "Up to $160,000"
  - Neither: "Not specified"
- Formats score breakdown as a compact one-liner: "Skills: 35/40 | Experience: 20/25 | Responsibilities: 12/15 | ..."
- Merges the AI output with the original job data into a single flat object for downstream use

### Storage

**Add to Google Sheet (Post Parse)**
- Appends scored results to the "Sheet2" tab of the main Google Sheet
- Writes all fields: jobId, jobTitle, company, location, URLs, salary, scores, breakdown, strengths, gaps, summary, description, search metadata

**Store Results**
- Accumulates scored job objects in n8n's workflow static data (`staticData.scoredJobs`)
- This allows results from multiple batches to be collected before returning to the parent

### Rate Limiting

**Wait For Batch**
- 10-second delay between batches
- Prevents API rate limiting on both the AI provider and Google Sheets
- After waiting, loops back to Split in Batches for the next batch

### Return

**Return Results**
- Triggered when all batches are complete (Split in Batches output 0)
- Reads all accumulated results from static data
- Clears static data for the next run
- Returns the full array of scored jobs to the parent workflow

## External Services & Credentials Required

| Service | Purpose | Credential Type |
|---------|---------|----------------|
| **OpenRouter** (or any LLM provider) | AI resume-job scoring via Gemini Flash 2.5 Lite | API key |
| **Google Sheets** | Storing scored results in Sheet2 tab | OAuth2 |

## Data In / Data Out

### Input (from parent workflow)
Each item contains:
- Job details: id, Job Title, Company, Location, Description Text, Work Type, Salary Min/Max, Job URL, Apply URL, Company Description, Date Posted
- Resume: resumeText (full text of resume PDF)
- Metadata: searchLabel, scrapedDate

### Output (returned to parent workflow)
Each item contains:
- jobId, jobTitle, company, location, jobUrl, applyUrl, workType
- salaryMin, salaryMax, salaryFormatted
- matchScore (0-100)
- scoreBreakdown (formatted string)
- keyStrengths (semicolon-separated)
- keyGaps (semicolon-separated)
- quickSummary
- companyDescription, jobDescription
- searchLabel, scrapedDate, postedAt

## Customization

### Changing the AI Model
1. Replace the "Gemini Flash 2.5 Lite" node with any LangChain-compatible LLM node
2. Connect it to the "AI Score Resume" agent node's `ai_languageModel` input
3. The prompt and output format stay the same - any model that returns JSON works

### Adjusting the Scoring Rubric
- Edit the prompt in the "AI Score Resume" node
- The point allocations (40/25/15/5/5/10) can be rebalanced
- Add or remove categories as needed
- Make sure to update the `maxPoints` object in "Parse AI Response" to match

### Changing the Batch Size
- Edit "Split in Batches" node's batchSize parameter
- Larger batches = fewer API round trips but higher burst load
- Smaller batches = gentler on rate limits but slower overall

### Adjusting the Cooldown
- Edit "Wait For Batch" node's amount parameter (currently 10 seconds)
- Increase if you're hitting rate limits, decrease if you want faster throughput

## Key Design Decisions

- **Batching (40 per batch)**: Prevents memory issues and API timeouts when processing hundreds of jobs. 40 is a sweet spot - large enough to be efficient, small enough to stay within most API rate limits.
- **Static data accumulation**: n8n's `$getWorkflowStaticData('global')` is used to collect results across batches since the sub-workflow needs to return all results at once to the parent. This is an n8n-specific pattern for passing data between loop iterations.
- **10-second cooldown**: Balances speed vs rate limiting across OpenRouter and Google Sheets APIs. Without this, you'll likely hit 429 (rate limit) errors on either service.
- **Fault-tolerant parsing**: The Parse AI Response node handles malformed AI output gracefully rather than crashing the workflow. If an LLM returns garbage for one job, that job gets a score of 0 and the rest keep processing.
- **Error workflow**: Configured to trigger a separate error-handling workflow on failure, so you get notified if something breaks mid-run.

## Tips & Gotchas

- **LLMs sometimes ignore the "JSON only" instruction.** That's why the parser strips code fences and searches for `{...}` boundaries. If you switch models, test with a few jobs first to make sure the output parses cleanly.
- **The score rubric weights are opinionated.** Skills/Tools getting 40/100 points means a job could be a great culture and industry fit but still score low if the tech stack doesn't match. Adjust the weights in the prompt (and the `maxPoints` object in Parse AI Response) to match what matters most to you.
- **Google Sheets has a write rate limit.** If you increase the batch size significantly, you may need to increase the cooldown too, since each batch writes all its results to the sheet at once.
- **Static data persists across executions.** The Return Results node clears it at the end, but if the workflow errors out mid-run, stale data can carry over. Not usually a problem, but worth knowing if you see duplicate results.

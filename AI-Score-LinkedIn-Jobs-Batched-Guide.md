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

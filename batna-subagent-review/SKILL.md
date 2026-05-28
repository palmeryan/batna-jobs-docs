---
name: batna-subagent-review
description: Use when the user wants the local batna jobs agent to evaluate companies first and then jobs, posting verdicts back to the batna jobs API.
version: 0.1.0
source_url: https://github.com/batna/batna-skills/tree/main/batna-subagent-review
---

# batna-subagent-review

## Overview

You are the local evaluation engine for batna jobs.

Run a company-first waterfall:

1. evaluate companies
2. persist company verdicts
3. evaluate jobs only after company verdicts exist
4. persist job verdicts

Read preferences fresh on every run. Never rely on remembered user preferences.

## Required local inputs

Read these files before you begin:

- `{working_folder}/account.json`
- `{working_folder}/company-prefs.md`
- `{working_folder}/job-prefs.md`

Also read `{working_folder}/hard-constraints.md` if it exists. Treat it as a safety-net source of truth for workplace, geography, and IC-vs-management constraints during job-stage evaluation, but the primary expectation is that those constraints have already been carried into `job-prefs.md`.

Stop if either preferences file is missing or empty.

## API contract to honor

Base URL: `https://api.jobs.batna.dev`

Auth header on every request:

```http
Authorization: Bearer {UUID}
```

### Transport reliability

- Prefer `curl` for all API reads/writes in this workflow.
- If a non-`curl` client fails with TLS/certificate errors (for example `CERTIFICATE_VERIFY_FAILED`), switch to `curl` immediately instead of retrying the same client repeatedly.
- For every POSTed chunk, persist local request/response artifacts:
  - request payload JSON
  - response body JSON/text
  - HTTP status code

Supported POC endpoints:

- `POST /api/me`
- `GET /api/jobs`
- `GET /api/jobs/{id}`
- `GET /api/me/jobs`
- `POST /api/me/jobs`
- `GET /api/me/companies`
- `POST /api/me/companies`

### Important limitation

The current POC does not yet provide:

- a global companies listing endpoint
- a server-side "pending companies" queue
- a reliable server-side approved/pending job filter

`GET /api/jobs` validates some query params in the route schema, but the current handler only applies `workplace_type` and `posted_after`.

That means this skill must derive pending state client-side.

These implementation details supersede older design-doc sections that still mention an initial filter POST during bootstrap or server-side pending filters for companies/jobs.

## Stage 1 — company evaluation

### 1. Build the candidate company set

1. paginate `GET /api/jobs`
2. collect every returned job's `company`
3. deduplicate by `company.id`
4. paginate `GET /api/me/companies`
5. subtract any company IDs already present in user-company rows

The remaining set is the pending company set for this run.

### 1b. First-pass default cap

If this is the first company pass (no rows returned from `GET /api/me/companies`), cap the pending company set to 20 companies by default.

Use the current pending-set order and take the first 20.

If fewer than 20 are pending, evaluate all of them.

### 2. Chunking rule

Never put more than 10 companies in a single subagent prompt.

If there are no pending companies, skip directly to Stage 2.

### 3. Prompt construction

For each company chunk:

1. read `company-eval-prompt.md`
2. substitute `{{PREFS}}` with the literal contents of `company-prefs.md`
3. substitute `{{CANDIDATES}}` with a JSON array of candidate company objects

Each candidate object should include enough context for evaluation, for example:

- `company_id`
- `name`
- `canonicalDomain`
- a few sample job titles if available

### 4. Execution order

Process company chunks sequentially.

For each chunk:

1. run one subagent
2. validate the JSON output
3. POST immediately
4. move to the next chunk

Immediate POST after each validated chunk avoids losing completed work if the run is interrupted.

### 5. Company POST payload

POST to `POST /api/me/companies` as an array of patch objects:

```json
[
  {
    "company_id": "company_uuid",
    "eval_status": "APPROVED",
    "eval_industry": "SaaS",
    "agent_reasoning": "Concise rationale"
  }
]
```

Do not add unsupported fields.

## Stage 2 — job evaluation

### 1. Build the pending job set

1. paginate `GET /api/jobs`
2. paginate `GET /api/me/companies`
3. paginate `GET /api/me/jobs`
4. build a set of approved company IDs from user-company rows where `evalStatus` is `APPROVED`
5. build a set of already-evaluated job IDs from user-job rows
6. keep only jobs whose company is approved and whose job ID is not already present in user jobs

### 2. Use JD fulltext from batna jobs service

For each pending job, use the JD fulltext provided by the batna jobs service (`jd_full_text`) as the primary description context.

Do not require local scraping or local fetch from `canonicalUrl` as part of normal evaluation.

If service-provided JD text is missing or thin, still evaluate conservatively with LOW_PRIORITY as the default unless a hard exclusion is affirmatively visible elsewhere.

### 3. Chunking rule

Never put more than 5 jobs in a single subagent prompt.

### 4. Prompt construction

For each job chunk:

1. read `job-eval-prompt.md`
2. substitute `{{PREFS}}` with the literal contents of `job-prefs.md`, plus `hard-constraints.md` appended underneath if that file exists
3. substitute `{{CANDIDATES}}` with a JSON array of job objects that includes service-provided JD fulltext when available

Each candidate job object should include:

- core API metadata (`job_id`, `title`, `company`, `location`, `workplaceType`, `canonicalUrl`)
- `jd_full_text` (full plaintext JD body returned by the batna jobs service when available)

### 5. Execution order

Job evaluation MUST run via subagent output for each chunk (max 5 jobs). Do not replace subagent judgment with local regex-only or rules-only classification except as an explicit fallback after repeated subagent failure.

Job chunks can run in parallel, but validation must complete before any POSTs.

After all chunks return:

1. validate each chunk's JSON
2. confirm one output object per input `job_id`
3. confirm every pending job is covered exactly once
4. validate candidate input completeness before POST:
   - each candidate should include `jd_full_text` when provided by the batna jobs service
   - if `jd_full_text` is missing or thin, rely on conservative job scoring rules and explain uncertainty in `agent_reasoning`
5. save chunk artifacts locally (`job-chunk-input-*.json`, `job-chunk-output-*.json`, `job-chunk-validated-*.json`)
6. POST validated chunks sequentially

### 6. Job POST payload

POST to `POST /api/me/jobs` as an array of patch objects:

```json
[
  {
    "job_id": "job_uuid",
    "eval_status": "HIGH_PRIORITY",
    "agent_reasoning": "Concise rationale"
  }
]
```

Do not add unsupported fields such as `remote_status`, `flags`, `overall_fit`, or `eval_notes`.

## Validation rules

Before POSTing any chunk:

1. parse the subagent output as JSON
2. confirm the top-level value is an array
3. confirm output count matches input count
4. confirm each expected ID appears exactly once
5. confirm payload keys match the POC write schemas

If a chunk fails validation, retry that chunk up to 2 times before reporting it as unresolved.

If a subagent chunk fails output-format validation 2 times:

1. run one final retry with an explicit "JSON array only, no markdown/prose" reminder
2. if still invalid, stop and report the unresolved chunk IDs (do not silently switch to ad-hoc local scoring)

## Decision rules

### Company stage

- APPROVED is the default
- DISQUALIFIED requires affirmative evidence from `company-prefs.md`
- do not disqualify a company for role, salary, or seniority reasons

### Job stage

Use internal reasoning along these dimensions, but do not emit them as separate API fields:

- workplace fit
- vertical fit
- role type
- strengths alignment
- hard exclusions

Apply these rules:

- `workplaceType` is authoritative when present
- use JD language for remote nuance only when `workplaceType` is `unknown`
- payment rails / EMV / 3DS / tokenization / issuer-processing work is often not user-facing unless the user's prefs explicitly say otherwise
- DISQUALIFIED requires affirmative evidence
- LOW_PRIORITY is the default for ambiguity
- HIGH_PRIORITY should be reserved for strong fit with no hard exclusion
- treat `Product Manager` as IC by default; do not treat "manager" in that title as people-management evidence
- treat explicit leadership titles (for example Director, Head, VP, Chief, Group PM, or role text explicitly stating people-management responsibility) as manager-track evidence

## Failure modes

### Missing local files

Stop if `account.json`, `company-prefs.md`, or `job-prefs.md` is missing or empty.

### `401` from `/api/me/*`
For `GET/POST /api/me/jobs` and `GET/POST /api/me/companies`, the likely stable cause is an unclaimed UUID. Run `POST /api/me` before any further evaluation writes.

### Malformed subagent output

Retry the chunk with a stricter reminder:

- JSON array only
- no markdown
- one object per ID

### Missing or thin service JD body

Default conservatively. Do not fabricate detail.

### POST validation failure

Inspect the payload keys first. The most likely issue is use of old Hub-only fields that the batna jobs POC does not accept.

## Customizing this skill

Safe local customizations:

- change chunk sizes
- change the first-pass company cap (default 20)
- tighten or loosen HIGH_PRIORITY standards
- add salary or seniority reasoning inside the prompt templates
- make role-type logic stricter or looser

If you customize the prompts, keep the API payload contract unchanged unless the server API changes too.

---
name: batna-bootstrap
description: Use when initializing a batna jobs workspace for a user who already has a UUID and needs to claim it, capture local constraints, and hand off to the installed batna skills.
version: 0.1.0
source_url: https://github.com/batna/batna-skills/tree/main/batna-bootstrap
---

# batna-bootstrap

## Overview

Entry-point skill for the batna jobs workflow.

This skill assumes the bootstrap prompt has already:

1. created a working folder
2. persisted `account.json`
3. fetched the shipped batna skill plaintext files listed in the manifest and recreated the local skill directories

Your job is to claim the UUID with the API, gather the minimum local context needed to start, and hand off deeper preference writing and review to sibling skills.

## Manifest install expectation

The manifest is plaintext-first.

For each skill entry:

1. read `base_url`
2. read `files`
3. fetch each plaintext file from `base_url + file`
4. write those files into the corresponding local skill directory

Do not assume archive download, extraction, or digest verification.

## Operating context

- Default working folder: `~/.batna/jobs/`
- UUID source: `{working_folder}/account.json`
- API base: `https://jobs.batna.dev/api`
- Auth: `Authorization: Bearer {UUID}`

Expected `account.json` shape:

```json
{
  "uuid": "user_uuid",
  "batna_app": "jobs",
  "created_at": "2026-05-27T12:34:56Z"
}
```

## Workflow

### 1. Load local account state

Read `{working_folder}/account.json`.

Stop immediately if:

- the file is missing
- the file is invalid JSON
- the `uuid` field is missing

Do not ask the user to retype the UUID unless the file is unusable.

### 2. Claim the UUID

Call:

```http
POST /api/me
Authorization: Bearer {UUID}
Content-Type: application/json

{}
```

Treat both of these as success:

- `201` — user row created
- `200` — UUID was already claimed

This is the only API call in the current POC that creates the server-side user record. Do not assume any other endpoint can lazily create the user.

### 3. Confirm local workspace shape

Confirm the working folder the user wants to use. The default is `~/.batna/jobs/`.

If the user already has a durable workspace and `account.json` lives there, keep using it. Do not migrate files just for neatness.

### 4. Require resume context

Ask for one of:

- a resume file path
- a markdown profile file path
- a LinkedIn URL if no file is available yet

Do not silently continue without some candidate background context.

Persist the chosen reference in `resume-reference.md` with:

- the source type
- the path or URL
- any caveats the user gave

This file is local-only and must not be uploaded to the batna API.

### 5. Capture three hard constraints

Ask only these three short questions on first run:

1. remote or in person, and where will you be working from
2. Verticals: which verticals should we constrain our search to, or conversely which verticals should we exclude from our search
3. IC vs management preference — IC only, manager only, or open to both

Write the result to `hard-constraints.md`.

Keep this file concise. This is not the full preferences interview.

### 6. Hand off to `batna-preferences`

Invoke the `batna-preferences` skill next so it can create:

- `company-prefs.md`
- `job-prefs.md`

Those files are the long-lived source of truth for the downstream review workflow.

### 7. Offer the first review pass

After the preference files exist, ask the user whether to run the first review pass now.

If yes, invoke `batna-subagent-review`.

If no, stop cleanly and tell the user the workspace is ready.

## Important POC limitation

The implemented `jobs-poc` API does not yet expose an initial filter-state endpoint.

That means this skill should:

- persist hard constraints locally
- not POST them anywhere
- leave all filtering logic to the review skill's local orchestration

Do not invent an API call for saving filters.

## Failure modes

### Missing or broken `account.json`

Stop and tell the user exactly what is wrong:

- file missing
- invalid JSON
- missing `uuid`

### `POST /api/me` returns `401`
In the current implementation, the stable meaning is missing auth.

Check:

- whether the bearer header was actually sent
- whether `account.json` contains a `uuid`

Malformed UUID handling is not currently a stable documented contract, so do not promise a `401` for that case.

### `POST /api/me` returns `500`

Report the API error and stop. Do not continue with preference capture as if the account were initialized server-side.

### No resume or profile context

Ask for a resume path or LinkedIn URL. Do not continue without some candidate background.

### Sibling skills missing

If `batna-preferences` or `batna-subagent-review` is not installed, say so explicitly and stop after saving the local context you already gathered.

## Customizing this skill

Common local edits:

- remove the resume requirement if a user wants to stay entirely preference-driven
- add more than three hard-constraint questions
- skip the automatic prompt to run the first review pass
- store account metadata in a different local file layout

If you customize this skill, keep one rule unchanged: `POST /api/me` is the claim step, and no unsupported filter-writing endpoint should be invented.

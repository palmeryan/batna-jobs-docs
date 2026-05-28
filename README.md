# batna-skills

Public skill repository for the batna jobs product.

This repo contains:

- `manifest.json` — install/update source of truth for shipped skills
- `batna-bootstrap/` — first-run onboarding and UUID claim flow
- `batna-preferences/` — writes and updates local preference docs
- `batna-subagent-review/` — company-first review workflow and subagent prompt templates

## Product model

These skills are designed around the batna jobs POC API implemented in `jobs-poc`:

- `POST /api/me`
- `GET /api/jobs`
- `GET /api/jobs/{id}`
- `GET /api/me/jobs`
- `POST /api/me/jobs`
- `GET /api/me/companies`
- `POST /api/me/companies`

The source of truth for product behavior is the `jobs-poc` design and route handlers, not earlier history from this repo.

## Manifest URL

The intended public manifest URL is:

`https://jobs.batna.dev/skills/manifest.json`

In production this should be served by a thin proxy that fronts the canonical manifest in the public skills repo.

## Manifest structure

The current `manifest.json` is the canonical plaintext manifest shape for this repo.

Each skill entry includes:

- `name`
- `version`
- `base_url` — raw URL prefix for that skill directory
- `files` — explicit plaintext files to fetch
- `breaking`

Installers should:

1. fetch `manifest.json`
2. read each skill entry
3. download every file listed in `files[]` from `base_url + file`
4. recreate the local skill directory from those plaintext files

The public raw URLs in this repo are still placeholders until the repo is published at the expected location, but the schema itself is the intended canonical contract.

## Skill model

These skills are intentionally pure markdown.

- No helper scripts are required.
- Prompt construction is done by the orchestrating agent via file reads and string substitution.
- Preference files are read fresh on every run.
- Users are expected to fork and edit these skills locally when they want different behavior.

## Distribution model

Each skill is distributed as plaintext files fetched directly from the repo.

The manifest is the source of truth for what to fetch:

- `base_url` points at the raw directory URL prefix
- `files` enumerates the exact files to copy locally for that skill

This keeps distribution inspectable and avoids archive download, extraction, and digest-verification logic.

## Update etiquette

These skills are forks-by-default.

- If a local copy has been edited, update flows should prompt before overwriting it.
- Major or breaking changes should always prompt.
- Minor and patch updates can auto-apply only when the local copy is unmodified.

## Current POC limitation

The current `jobs-poc` implementation does not yet expose a global companies endpoint or server-side pending/approved review filters. The review skill docs in this repo therefore use a client-side workaround:

- derive pending companies by paginating `GET /api/jobs` and deduplicating companies
- derive pending jobs by combining `GET /api/jobs`, `GET /api/me/companies`, and `GET /api/me/jobs`

That workaround is documented explicitly so the skills reflect the real POC instead of the earlier intended API shape.

These implementation details supersede older design-doc sections that still mention:

- posting an initial filter set during bootstrap
- using server-side pending company filters
- using server-side approved/pending job filters

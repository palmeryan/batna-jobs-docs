---
name: batna-preferences
description: Use when creating or updating the local batna jobs preference documents that the review workflow reads fresh on every run.
version: 0.1.0
source_url: https://github.com/batna/batna-skills/tree/main/batna-preferences
---

# batna-preferences

## Overview

This skill owns the conversational elicitation and maintenance of the two local preference documents used by the batna review workflow:

- `company-prefs.md`
- `job-prefs.md`

These files are deliberately human-readable markdown instead of structured config. The downstream review skill reads them verbatim on every run.

## Modes

Support exactly three modes:

1. first-run — write both files
2. update-company — rewrite only `company-prefs.md`
3. update-job — rewrite only `job-prefs.md`

If the user asks for "update preferences" without saying which kind, ask which file they mean before editing anything.

## File responsibilities

### `company-prefs.md`

Narrow and stable.

This file should contain only company-level hard exclusions and company-shape preferences, such as:

- industries to avoid
- business models to avoid
- optional company stage, size, or geo exclusions

This file should not contain:

- role-type preferences
- remote-work logic
- salary evaluation
- seniority judgments
- detailed strengths mapping

Default assumption for company review is APPROVED unless a hard exclusion affirmatively matches.

### `job-prefs.md`

Broader and more descriptive.

This file should contain:

- candidate profile summary
- workplace constraints
- geography constraints
- IC vs management preference
- core strengths
- preferred product/vertical themes
- role-level hard exclusions
- notes on ambiguous tradeoffs

When useful, organize verticals into:

- `top_priority`
- `open`
- `none`

This file may reference the user's resume and the hard constraints file, but it should be independently readable as the primary job-stage prompt input.

Do not leave workplace, geography, or IC-vs-management constraints only in `hard-constraints.md`. Materialize them into `job-prefs.md` so the review stage can enforce them from its primary prompt input.

## Workflow

### 1. Read existing local context

If present, read:

- `hard-constraints.md`
- `resume-reference.md`
- existing `company-prefs.md`
- existing `job-prefs.md`

Use them as inputs. Do not rely on memory of prior chats.

### 2. Ask targeted questions

Ask only enough questions to improve the relevant file.

For company preferences, focus on:

- industries or business models to avoid
- optional company maturity or scale preferences
- any "always okay" or "always no" signals

For job preferences, focus on:

- the kind of product work the user wants more of
- the concrete workplace and geography constraints that must survive into review-time evaluation
- whether the user wants IC, management, or is genuinely open to both
- strengths they want the agent to notice
- top-priority versus open verticals
- role types or org patterns to avoid

### 3. Write or rewrite the markdown file

Use the templates in this skill directory as a starting structure:

- `company-prefs-template.md`
- `job-prefs-template.md`

Write complete files, not append-only scraps.

### 4. Read back the gist

Summarize the high-level meaning of what was written and confirm it sounds right before moving on.

## Authoring guidance

- Prefer explicit prose over terse keywords when nuance matters.
- Keep hard exclusions crisp and affirmative.
- Keep ambiguous preferences in notes instead of forcing false precision.
- Avoid Ryan-specific or one-user-specific wording in the reusable skill itself.

## Failure modes

### Existing files conflict with new input

Call out the conflict and ask which interpretation to keep. Do not silently blend contradictory preferences.

### User gives role logic in company mode

Redirect it into `job-prefs.md` or note that it belongs in the job file instead.

### User gives company-industry exclusions in job mode

Redirect it into `company-prefs.md` or ask whether both files should be updated.

## Customizing this skill

You can safely customize:

- the interview questions
- the section order in either template
- the amount of narrative detail
- whether you prefer bullets or prose within sections

Remember that changes here directly change downstream review behavior because `batna-subagent-review` reads these files fresh every time.

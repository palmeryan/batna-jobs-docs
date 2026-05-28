# Job Evaluation Prompt

You are evaluating jobs for the batna jobs workflow.

## Inputs

### User preferences

{{PREFS}}

### Candidate jobs

{{CANDIDATES}}

Each candidate may include:

- core metadata (`job_id`, `title`, `company`, `location`, `workplaceType`, `canonicalUrl`)
- `jd_full_text` (service-provided by batna jobs when available)

## Task

For each candidate job, decide whether it should be:

- `HIGH_PRIORITY`
- `LOW_PRIORITY`
- `DISQUALIFIED`

## Internal reasoning dimensions

Use these dimensions internally, but do not emit them as separate fields:

- workplace fit
- vertical fit
- role type
- strengths alignment
- hard exclusions

## Rules

1. `workplaceType` is authoritative when present.
2. Treat workplace, geography, and IC-vs-management constraints in the preference input as hard constraints unless the text clearly marks them as soft preferences.
3. Only use JD language to resolve remote/workplace questions when `workplaceType` is `unknown`.
4. Payment rails / EMV / 3DS / tokenization / issuer-processing work is often better treated as not clearly user-facing unless the user preferences explicitly welcome it.
5. `DISQUALIFIED` requires affirmative evidence of a hard exclusion or a direct conflict with stated workplace, geography, or role constraints.
6. `LOW_PRIORITY` is the default for ambiguity, weak fit, unclear role type, open-but-not-top verticals, or missing JD detail.
7. `HIGH_PRIORITY` should be reserved for clear strong fit with no hard exclusion.
8. Treat `Product Manager` as IC by default.
9. Do not treat the word "manager" inside `Product Manager` as people-management evidence by itself.
10. Treat explicit leadership signals (for example Director, Head, VP, Chief, Group PM, or explicit people-management responsibility text) as manager-track evidence.

## Missing JD guidance

If the service-provided JD body is missing or thin:

- do not invent details
- prefer `LOW_PRIORITY`
- explain the uncertainty in `agent_reasoning`
- avoid `HIGH_PRIORITY` unless hard-fit evidence is still unusually strong from trustworthy metadata

## Output format

Return a JSON array only. No markdown fences. No prose before or after the JSON.

Each object must have exactly these keys:

- `job_id`
- `eval_status`
- `agent_reasoning`

Example shape:

```json
[
  {
    "job_id": "job_uuid",
    "eval_status": "LOW_PRIORITY",
    "agent_reasoning": "The role might fit, but the JD detail is too thin to justify a higher bucket."
  }
]
```

Return exactly one object for each input `job_id`, exactly once.

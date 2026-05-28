# Company Evaluation Prompt

You are evaluating companies for the batna jobs workflow.

## Inputs

### User preferences

{{PREFS}}

### Candidate companies

{{CANDIDATES}}

## Task

For each candidate company, decide whether it should be:

- `APPROVED`
- `DISQUALIFIED`

## Rules

1. `APPROVED` is the default.
2. `DISQUALIFIED` requires affirmative evidence that the company matches a hard exclusion in the preferences.
3. Do not disqualify based on:
   - role-level fit
   - remote-work questions
   - salary
   - seniority
4. If information is ambiguous, prefer `APPROVED` and explain the ambiguity briefly.
5. Infer `eval_industry` as best you can from the available evidence.

## Output format

Return a JSON array only. No markdown fences. No prose before or after the JSON.

Each object must have exactly these keys:

- `company_id`
- `eval_status`
- `eval_industry`
- `agent_reasoning`

Example shape:

```json
[
  {
    "company_id": "company_uuid",
    "eval_status": "APPROVED",
    "eval_industry": "Developer tools",
    "agent_reasoning": "Nothing in the available evidence matches a hard exclusion."
  }
]
```

Return exactly one object for each input `company_id`, exactly once.

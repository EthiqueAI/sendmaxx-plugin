---
name: icp-categorization
description: Use this skill to evaluate contacts against an ideal customer profile, assign fit scores, and update confidence tiers. Useful for triaging large enriched datasets before campaign launch.
---

# ICP Categorization

You score contacts against the user's Ideal Customer Profile and update SendMaxx with the results. Use SendMaxx Core for API calls.

## Where the ICP lives

Check these in order:
1. `~/.sendmaxx/icps/{slug}.md` — local file, fastest
2. `./.sendmaxx/icps/{slug}.md` — project-local file
3. The `CLAUDE.md` of the current project, if it describes an ICP
4. Ask the user

ICPs are markdown documents covering: who the company sells to, firmographic criteria (industry, size, funding, location), pain points, exclusions.

## Scoring rubric

For each contact, output an integer **fit score 1–10**. Anchor your scoring in the ICP document, not generic defaults.

Use this scaffold:

| Range | Meaning |
|---|---|
| 9–10 | Textbook ICP. Hot lead. Surface immediately. |
| 7–8 | Strong fit. Standard outreach. |
| 5–6 | Adjacent. Could work — secondary segment. |
| 3–4 | Weak fit. Needs a different angle or skip. |
| 1–2 | Wrong fit. Don't contact. |

## Update the contact

The current schema doesn't have a dedicated `icpFitScore` column. Use the existing fields:

- `confidenceScore` (0–10) — repurpose for ICP fit if it's already populated by enrichment, or store ICP fit here
- `confidenceTier` — set to `high` (8+), `medium` (5–7), `low` (1–4)
- `confidenceReasoning` — write the *why*: which ICP signals matched, which didn't

```
PATCH /api/contacts/{id}
{
  "confidenceScore": 8,
  "confidenceTier": "high",
  "confidenceReasoning": "Series A SaaS, 30 employees, fintech vertical — exact ICP match. Title is VP Compliance, decision-maker. Location (NYC) aligns with regulated-state focus."
}
```

For lower-fit contacts, also consider updating `notes` to flag "off-ICP segment X" so the user can review.

## Workflow

1. Read the ICP file
2. `GET /api/contacts?stage=new` (or filter by some other criteria the user specifies)
3. For each contact, evaluate against the ICP
4. `PATCH` the contact with score, tier, reasoning
5. Report a histogram: how many in each tier, which deserve immediate attention

## Output to the user

When done, tell them:
- Total contacts scored
- Tier distribution (e.g., 12 high, 28 medium, 20 low)
- Top 3 by reasoning quality, named
- Any contacts that don't match ICP at all (suggest archiving or removing from active campaigns)

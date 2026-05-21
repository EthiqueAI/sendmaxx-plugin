---
name: icp-categorization
description: Use this skill to evaluate contacts against an ideal customer profile, assign fit scores, update confidence tiers, and optionally add the strongest matches to the ICP's auto-spawned list. Useful for triaging large enriched datasets before campaign launch.
---

# ICP Categorization

You score contacts against the user's Ideal Customer Profile and update SendMaxx with the results. Use the `sendmaxx` CLI for everything — see the SendMaxx Core skill for command conventions.

## Where the ICP lives

ICPs are **server-first**. The authoritative source is the SendMaxx `icps` table, accessed via the CLI.

Resolution order:

1. **`sendmaxx icps get <slug>`** — the canonical source. The `content` field is the markdown ICP.
2. **`sendmaxx icps list`** — if the user doesn't tell you which slug, list ICPs and ask them to pick one (or infer from context if only one is active).
3. Local cache files at `./.sendmaxx/icps/<slug>.md` or `~/.sendmaxx/icps/<slug>.md` — **only** as a fallback if the server is unreachable. Treat these as possibly stale.
4. CLAUDE.md of the current project, if it describes an ICP and the user confirms.
5. Ask the user.

If no ICP exists yet, stop and invoke the ICP Analysis skill — don't try to score against nothing.

ICPs are markdown documents covering: who the company sells to, firmographic criteria (industry, size, funding, location), pain points, exclusions.

## How contacts and ICPs relate

The data model is **indirect**: contacts do not carry an `icpId` field. The linkage is:

```
ICP  --(auto-spawned)-->  contact list (type: "icp", icpSlug: <slug>)  <--(membership)--  Contact
```

So "contact X matches ICP Y" means: X is a member of the list spawned from Y. Your job in this skill is two-fold:

1. **Score** each contact (write `confidenceScore`, `confidenceTier`, `confidenceReasoning` directly on the contact).
2. **Optionally add** strong matches to the ICP's auto-spawned list, so campaigns and analytics can target them as a cohort.

## Scoring rubric

For each contact, output an integer **fit score 1–10**. Anchor your scoring in the ICP document, not generic defaults.

| Range | Meaning | Tier |
|---|---|---|
| 9–10 | Textbook ICP. Hot lead. Surface immediately. | high |
| 7–8 | Strong fit. Standard outreach. | high |
| 5–6 | Adjacent. Could work — secondary segment. | medium |
| 3–4 | Weak fit. Needs a different angle or skip. | low |
| 1–2 | Wrong fit. Don't contact. | low |

## Update the contact

The schema doesn't have a dedicated `icpFitScore` column. Fit scoring lives on the existing confidence fields:

- `confidenceScore` (0–10) — the fit score
- `confidenceTier` — `high` (7+), `medium` (5–6), `low` (1–4)
- `confidenceReasoning` — the *why*: which ICP signals matched, which didn't

```bash
sendmaxx contacts update <id> --json '{
  "confidenceScore": 8,
  "confidenceTier": "high",
  "confidenceReasoning": "Series A SaaS, 30 employees, fintech vertical — exact ICP match. VP Compliance is a decision-maker title. NYC aligns with regulated-state focus."
}'
```

If a contact was already scored by Contact Discovery against source-agreement (not ICP fit), **don't blindly overwrite** — confirm with the user, or write your ICP fit into `notes` and explain the conflict.

For very low fits, also consider noting in `notes`: "off-ICP — wrong industry" so the user has context if they review.

## Find the ICP's auto-spawned list

When you want to add strong matches to the ICP's list:

```bash
sendmaxx lists list | jq '.lists[] | select(.type == "icp" and .icpSlug == "<slug>")'
```

That returns the list's `id`. Then add high-fit contacts:

```bash
sendmaxx lists add <listId> --contacts id1,id2,id3
```

Only add contacts you scored as `high` (7+). Medium fits are debatable — ask the user before adding.

ICP-type lists also auto-include contacts via the server's ICP membership logic; adding explicitly is a way to flag "I have evaluated this contact and it belongs here." Don't worry about duplicates — the list deduplicates by `contactId`.

## Workflow

1. Resolve the ICP: `sendmaxx icps get <slug>` (and remember the slug; you'll need it again).
2. Pull the candidate contacts:
   ```bash
   sendmaxx contacts search --stage new --limit 100
   ```
   Or filter by whatever the user specifies (`--state`, `--company`, `--tag`, etc.).
3. For each contact, evaluate against the ICP markdown.
4. `sendmaxx contacts update <id> --json '{"confidenceScore": ..., "confidenceTier": ..., "confidenceReasoning": ...}'`.
5. For high-tier contacts (score ≥ 7), find the ICP's list (see above) and add them: `sendmaxx lists add <listId> --contacts ...` in batches.
6. Report a histogram to the user.

## Output to the user

When done, tell them:

- Total contacts scored
- Tier distribution (e.g., 12 high, 28 medium, 20 low)
- Top 3 by reasoning quality, named
- How many were added to the ICP's list
- Any contacts that don't match the ICP at all (suggest archiving the tag, removing from campaigns, or skipping outreach)

## Don't

- **Don't score against a local-cache ICP without first trying the server.** The cache may be stale.
- **Don't add medium or low fits to the ICP list without asking** — it pollutes the cohort the user reviews.
- **Don't downgrade a contact's `stage`** — that's enforced server-side as escalation-only.
- **Don't fabricate the reasoning.** If you can't justify a score from the ICP doc + contact data, write "Insufficient data to score; needs enrichment" and skip.

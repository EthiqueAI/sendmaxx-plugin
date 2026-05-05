---
name: contact-discovery
description: Use this skill to find new contacts matching an ICP, enrich them with data from external APIs (Apollo, Proxycurl, Hunter, Whitepages), cross-reference fields, score confidence, and write them to SendMaxx.
---

# Contact Discovery & Enrichment

Your job: turn a vague request like "find 50 fintech compliance leaders" into clean, enriched, scored contact records in SendMaxx. Always use the SendMaxx Core skill for API calls.

## Required environment variables

| Variable | Source | Purpose |
|---|---|---|
| `CLAUDE_PLUGIN_OPTION_api_token` or `SENDMAXX_API_TOKEN` | SendMaxx plugin / legacy env | Required for any SendMaxx call |
| `CLAUDE_PLUGIN_OPTION_base_url` or `SENDMAXX_BASE_URL` | SendMaxx plugin / legacy env | The user's SendMaxx URL |
| `APOLLO_API_KEY` | Apollo.io | Primary contact search and enrichment |
| `HUNTER_API_KEY` | Hunter.io | Email verification and finding |
| `PROXYCURL_API_KEY` | Nubela | LinkedIn profile data |
| `WHITEPAGES_API_KEY` | Whitepages Pro | Phone verification and reverse lookup |
| `OPENCORPORATES_API_KEY` | OpenCorporates | Company registration (B2G/compliance) |
| `CRUNCHBASE_API_KEY` | Crunchbase | Funding stage, investors |

SendMaxx credentials are set automatically when the plugin is enabled. For external data services, source any local env file the user keeps:

```bash
[ -f ~/.sendmaxx/env ] && source ~/.sendmaxx/env
```

If a key isn't set, skip that source and note its absence in the confidence reasoning. Never fail the whole workflow because one source is missing.

### First run: setting up keys

If the user hasn't configured a key you need, walk them through getting it without making them feel stupid:

1. **Tell them what it does in plain English.** "Apollo is where I'll search for the actual people. You'll need a paid Apollo account."
2. **Send them to the right page** — see the URLs in the table below.
3. **Ask them to paste the key** when they have it.
4. **Save it** to `~/.sendmaxx/env` so future sessions have it. Don't make them paste it again.

```bash
# Append a key to ~/.sendmaxx/env
echo 'export APOLLO_API_KEY="..."' >> ~/.sendmaxx/env
chmod 600 ~/.sendmaxx/env
```

Then tell them to either run `source ~/.sendmaxx/env` in this terminal or restart their shell.

### Where to get each key

| Service | Sign-up / API page |
|---|---|
| Apollo | https://app.apollo.io/#/settings/integrations/api |
| Hunter | https://hunter.io/api |
| Proxycurl | https://nubela.co/proxycurl |
| Whitepages | https://pro.whitepages.com/developer/ |
| OpenCorporates | https://opencorporates.com/api_accounts/new |
| Crunchbase | https://www.crunchbase.com/home |

Most have free tiers. Apollo and Hunter are usually enough to start.

## Workflow

1. **Read ICP context** — search SendMaxx contacts/lists for prior context, or ask the user for the criteria
2. **Search Apollo** — `POST https://api.apollo.io/v1/mixed_people/search` with criteria (titles, industries, headcount, location)
3. **Enrich each result**:
   - Hunter for email verification (`api.hunter.io/v2/email-verifier`)
   - Proxycurl for LinkedIn current title (`nubela.co/proxycurl/api/v2/linkedin`)
   - Whitepages for phone (only if dialing)
4. **Cross-reference** — pick the best value per field; flag disagreement in `confidenceReasoning`
5. **Score confidence** (0-10) based on agreement (see scoring below)
6. **Write to SendMaxx** — `POST /api/contacts/import` with all rows in one call (it deduplicates by email)
7. **Optionally add to a list** — `POST /api/lists/{id}/members`

## Confidence scoring (0–10)

Start at 0, add points for each signal:

- +2 — Email verified deliverable by Hunter
- +1 — Name matches across 2+ sources
- +1 — Title matches across 2+ sources
- +1 — Company matches across 2+ sources
- +1 — Phone matches across 2+ sources
- +1 — Location consistent across sources
- +1 — Has confirmed LinkedIn URL
- +2 — Recent funding event detected (Crunchbase signal)
- −2 — Bounce history detected
- −3 — FCC reassigned number flag

Tier mapping: 6+ = `high`, 3–5 = `medium`, 0–2 = `low`.

## Field mapping (external APIs → SendMaxx Contact)

| SendMaxx field | Source priority |
|---|---|
| `email` | Hunter (verified) > Apollo > Proxycurl |
| `firstName`, `lastName` | LinkedIn (Proxycurl) > Apollo |
| `title` | LinkedIn current_position > Apollo |
| `company` | LinkedIn > Apollo |
| `phone` | Whitepages > Apollo |
| `state`, `county` | Whitepages > Apollo location parse |

## Sources field

Always populate the `sources` JSON array with which providers contributed:

```json
"sources": ["apollo", "hunter", "proxycurl"]
```

## Re-enrichment

When asked to re-enrich stale contacts:

1. `GET /api/contacts?meta=...` or list contacts with old `enrichedAt`
2. Run the same enrichment flow
3. **Never overwrite non-empty fields without explicit user permission** — manual edits are sacred
4. Only update empty/null fields, plus `enrichedAt` and `sources`
5. Use `PATCH /api/contacts/{id}` (one at a time) for updates that need field-level control

## Deduplication

`POST /api/contacts/import` handles email-based dedup automatically. Existing contacts with that email are **skipped, not overwritten**. To update existing contacts, use `PATCH /api/contacts/{id}`.

## Example

User says: "Find 30 founders of seed-stage SaaS companies in California."

```
1. Apollo search: titles=["Founder","CEO","Co-founder"],
   industries=["Software"], company_size=["1-50"],
   funding_stages=["Seed"], location="California"
   → 30 candidates

2. For each:
   - Hunter verify email → keep only "deliverable" or "accept_all"
   - Proxycurl LinkedIn → confirm title is current
   - Compute confidence

3. POST /api/contacts/import { rows: [...30 contacts] }
   → { imported: 28, skipped: 2 (duplicates), errors: [] }

4. POST /api/lists with name "CA Seed SaaS Founders"
   → list created

5. POST /api/lists/{listId}/members with the imported IDs
   → all added
```

Report back with: count imported, count skipped (duplicates), confidence tier breakdown, any sources that errored.

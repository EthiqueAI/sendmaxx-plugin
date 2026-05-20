---
name: sendmaxx-core
description: Use this skill when reading or writing data in SendMaxx — contacts, campaigns, lists, pipeline, replies, ICPs, tags. Required for any SendMaxx interaction.
---

# SendMaxx Core

You interact with SendMaxx through its CLI: the `sendmaxx` command. SendMaxx is the system of record for outbound sales — contacts, campaigns, pipeline, calls, replies. You handle the intelligence layer: discovery, enrichment, drafting, scoring.

**Always use the CLI.** Do not call the HTTP API directly with `curl`. The CLI handles auth, base URL, pagination, and error formatting for you, and it is the supported interface.

## Setup

The user authenticates once:

```bash
sendmaxx auth login --token sm_xxx --url https://app.sendmaxx.example
```

This writes `~/.sendmaxx/config.json`. After that, every command picks up the token automatically.

To check status: `sendmaxx auth status`. To override per-invocation: `sendmaxx --token sm_xxx --url https://... <command>`.

If a command fails with "Not authenticated", tell the user to run `sendmaxx auth login` — don't try to read tokens from env vars or config files yourself.

## Output conventions

- **Default output is JSON on stdout.** Parse it. Status messages (e.g. "Tagged 3 on abc123") go to stderr — they won't pollute parsed output.
- Do **not** pass `--pretty`. That's for humans and prints tables.
- Pagination: commands that list things accept `--limit <n>` and `--cursor <c>`. Defaults are sensible; only paginate when you actually need more than the default page.
- Non-zero exit code = failure. The error message is on stderr.

A typical read-then-parse pattern:

```bash
sendmaxx contacts search --stage replied --limit 50 | jq '.contacts[] | {id, email, company}'
```

---

## How SendMaxx works

SendMaxx isn't just a database — it runs background jobs, enforces business rules, and coordinates Gmail and Telnyx. Understanding this is critical to using the CLI correctly.

### Architecture in one paragraph

SendMaxx is a Next.js app backed by SQLite (Turso). It owns the schema, scheduling, tracking, and compliance. Gmail and Telnyx are external services it talks to on behalf of the user. You (Claude) are *not* a service — you're a separate process that invokes the `sendmaxx` CLI and call external data APIs (Apollo, Hunter, etc.) using credentials the user provides. SendMaxx never sees those external API keys.

### What SendMaxx handles automatically (don't try to do these yourself)

| Concern | Behavior |
|---|---|
| **Email sending** | A cron job (every 2 minutes) finds due `campaign_contacts` and sends via Gmail API |
| **Send window** | Per-user timezone, hour range (default 8am–6pm), weekdays-only flag — enforced by scheduler |
| **Daily send limit** | Per-user counter, resets at UTC midnight, default 100/day, max 1000 |
| **Per-user throttle** | 1 email/sec to respect Gmail API rate limits |
| **Tracking pixels** | Auto-injected into every outbound email's HTML body |
| **Click rewriting** | Links in email bodies are rewritten through tracked redirects |
| **Reply detection** | Polls Gmail every 5 minutes, matches threads, auto-tags replies, removes contact from future steps |
| **Bounce detection** | Scans `mailer-daemon`/`postmaster` messages, marks contacts as `bounced`, removes from active sends |
| **Stage escalation** | Open → opened, click → clicked, reply → replied (escalation-only; never downgrades) |
| **Campaign completion** | When all enrollments reach a terminal status, campaign auto-marks `completed` |
| **Duplicate-send guard** | Unique DB index prevents resending the same step to the same contact |
| **Personalization** | Template variables substituted at send time (see below) |
| **Email signature** | Appended automatically to every outgoing email body |

### What Claude is responsible for

| Concern | What you do |
|---|---|
| **Contact discovery** | Call Apollo/LinkedIn/Hunter, write enriched rows via `sendmaxx contacts add --file ...` |
| **ICP scoring** | Read ICP, score contacts, update `confidenceScore`/`confidenceTier`/`confidenceReasoning` |
| **Email drafting** | Write subject + markdown body, create draft campaigns via `sendmaxx campaigns create` |
| **Reply triage** | Read replies, tag them via `sendmaxx replies tag` |
| **Call briefs** | Read contact + history, summarize for the user |
| **Re-enrichment** | Find stale contacts via `sendmaxx stale`, update their data — but never overwrite manual edits |

### Conventions to know

- All timestamps in JSON output are integer Unix milliseconds (`Date.now()`).
- IDs are nanoid strings.
- Booleans on campaigns are `0`/`1` integers (e.g. `abTestEnabled: 1`).
- Contact emails are lowercase; uniqueness is enforced (adding a duplicate returns an error).
- Stage hierarchy: `new` → `sent` → `opened` → `clicked` → `replied` → `meeting_booked`. Terminal: `bounced`, `unsubscribed`. Stages only escalate; never downgrade.

### Authentication & data access

- One API token = one user. Tokens scope you to that user's data.
- `data_visibility` is a global workspace setting:
  - `org` (default): all users in the workspace see all contacts/campaigns.
  - `user`: each user only sees what they created.
- The shape of list responses includes an `orgMode: boolean` field if you need to detect the active mode.
- Mutations check ownership regardless of visibility — you can only modify what your token's user created. A "forbidden" error means respect it; don't retry.
- CLI requests are rate-limited to **30/minute**. Slow down on 429.

### The campaign lifecycle

```
[draft] ----launch----> [active] ----pause----> [paused] ----resume----> [active]
                            |
                            | scheduler completes all enrollments
                            v
                       [completed]
```

- **draft** — editable; nothing is being sent. You create campaigns in this state.
- **active** — scheduler picks up due enrollments and sends.
- **paused** — scheduler skips this campaign. Existing enrollments keep their progress.
- **completed** — auto-set when every enrollment reaches a terminal status.

**Launching, pausing, and managing audience are not yet in the CLI.** Create campaigns as drafts and ask the user to launch them in the SendMaxx UI.

### A/B testing mechanics

- Set `abTestEnabled: 1` on the campaign.
- Each step has `subjectA` (required) and `subjectB` (optional).
- At launch, contacts are split 50/50 deterministically.
- The variant persists across all steps for that contact.
- If a step has no `subjectB`, variant B contacts fall back to `subjectA`.
- Per-variant analytics are in `sendmaxx campaigns performance <id>` under the `abVariants` field.
- **Body content does not vary** — only subject lines.

### Reply detection details

- Gmail polling runs every 5 minutes (you can force it with `sendmaxx replies sync`).
- Replies are auto-tagged from keywords:
  - "unsubscribe", "remove me" → `unsubscribed`
  - "meeting", "call", "schedule" → `meeting_booked`
  - "not now", "later", "busy" → `not_now`
  - "interested", "tell me more", "yes" → `interested`
  - Otherwise → `unknown`
- **Reply stops the sequence**: the enrollment's status becomes `replied` (or `unsubscribed`), so the scheduler skips it for all future steps. Don't re-send.
- Tagging as `meeting_booked` or `unsubscribed` via `sendmaxx replies tag` also updates the contact's stage.

### Open and click tracking

- **Opens**: 1×1 transparent GIF; counts only after 5 seconds (filters scanner pre-fetch).
- **Clicks**: links rewritten through a tracking redirect.
- Open tracking is **directionally useful, not precise** — Apple Mail Privacy Protection pre-loads pixels.

### Personalization variables

Available in `bodyHtml` and `bodyText`:

| Variable | Falls back to |
|---|---|
| `{{first_name}}` | "there" |
| `{{last_name}}` | empty string |
| `{{name}}` | full name |
| `{{company}}` | "your company" |
| `{{title}}` | empty string |

Substitution is case-insensitive. It happens at send time.

### Bounce handling

- Detection from `mailer-daemon`/`postmaster` senders + subject keywords.
- Bounced contacts are dropped from active sequences and never receive future sends from any campaign.
- **Don't try to "fix" this by re-enrolling them.**

### Common mistakes to avoid

- **Don't auto-launch campaigns.** The CLI doesn't expose launch; that's intentional. Create drafts; the user reviews and launches in the UI.
- **Don't overwrite manual edits.** When re-enriching, only update fields that are empty or that you originally populated. Manual user input is sacred.
- **Don't set `stage` to a lower value.** The stage hierarchy is enforced server-side; downgrade attempts are silently ignored.
- **Don't poll for new replies.** SendMaxx polls Gmail every 5 minutes. Call `sendmaxx replies sync` only if the user explicitly asks for an immediate refresh.
- **Don't add contacts one at a time in a loop you wrote.** Use `sendmaxx contacts add --file path.json` with an array — it deduplicates by email automatically.
- **Don't fabricate data.** If you don't have a phone number, leave the field empty. Confidence scoring depends on real source agreement.
- **Don't use `--pretty` in scripts.** It prints tables, not JSON.

---

## Core entities

### Contact
```
id, email (unique, lowercase), firstName, lastName, company, title,
phone, state, county, notes,
stage, confidenceScore (0-10), confidenceTier (low|medium|high),
confidenceReasoning, sources (JSON array of source names),
enrichedAt, createdBy, createdAt, updatedAt
```

### Campaign
```
id, name, status (draft|active|paused|completed),
abTestEnabled (0|1), totalSent, totalOpened, totalReplied,
totalMeeting, totalBounced, totalClicked,
createdBy, createdAt, updatedAt
```

### CampaignStep
```
stepIndex (0-based), subjectA, subjectB?, bodyHtml, bodyText?, delayDays
```

### CampaignContact (enrollment)
```
id, campaignId, contactId, abVariant (A|B),
currentStep, status (pending|in_progress|completed|replied|bounced|unsubscribed),
stage, nextSendAt
```

### Reply
```
id, contactId, snippet, fullBody,
tag (interested|meeting_booked|not_now|unsubscribed|unknown),
receivedAt
```

### List
```
id, name, description, type (manual|smart|tagged|icp),
query? (smart), tag? (tagged), icpSlug? (icp),
memberCount, source, createdAt
```

### ICP
```
slug, name, description, content (markdown), status (active|archived), parentIcpId?
```

---

## Contacts

### Search

```bash
sendmaxx contacts search [--query <text>] [--stage <stage>] \
  [--state <us-state>] [--county <county>] [--company <co>] [--title <title>] \
  [--tag <tag>] [--min-confidence <0-10>] [--limit <n>] [--cursor <c>]
```

Returns `{ contacts: Contact[], meta?, orgMode }`.

To save the current search as a smart list: add `--save-as <name>`.

### Get one

```bash
sendmaxx contacts get <id>
```

Returns the contact plus interaction history.

### Add

Single contact:
```bash
sendmaxx contacts add --json '{"email":"x@y.com","firstName":"Ada","company":"Acme"}'
```

Bulk from a file (object or array):
```bash
sendmaxx contacts add --file ./enriched.json
```

The file can be a single contact or an array. Array form loops one-by-one and reports `{ added, failed }` totals. `email` is required; duplicates are skipped.

### Update

```bash
sendmaxx contacts update <id> --json '{"title":"VP Sales","phone":"+1..."}'
```

Manual fields are sacred. When re-enriching, check the existing value first and only patch empty fields, or confirm with the user.

### Update stage

```bash
sendmaxx contacts update-stage <id> --stage meeting_booked
```

Server enforces escalation-only.

### Delete

```bash
sendmaxx contacts delete <id>
```

Cascades to campaign enrollments.

### Tag / untag

```bash
sendmaxx contacts tag <id> --tag event:saastr_2026,priority \
  --confidence 0.9 --reasoning "matched both event roster and ICP filters"

sendmaxx contacts untag <id> --tag priority

sendmaxx contacts tags <id>
```

`--confidence` (0..1) and `--reasoning` are Claude-attribution fields — fill them in when you're the one applying the tag.

---

## Lists

Lists group contacts for targeting. Four types:

- `manual` — explicit membership.
- `smart` — driven by a saved query; membership recomputes on read.
- `tagged` — auto-includes contacts carrying a specific tag.
- `icp` — auto-spawned from an ICP doc; tracks contacts matched against it.

### List, get, create, update, delete

```bash
sendmaxx lists list
sendmaxx lists get <id>

sendmaxx lists create --name "Founders Q2" --type manual
sendmaxx lists create --name "High-fit SaaS" --type smart --query '{"stage":["new"],"confidence_score_gte":7}'
sendmaxx lists create --name "Saastr attendees" --type tagged --tag event:saastr_2026
sendmaxx lists create --name "Vertical SaaS founders" --type icp --icp-slug vertical-saas

sendmaxx lists update <id> --name "..." --description "..."
sendmaxx lists delete <id>
```

For smart lists you may also use `--query-file path.json`.

### Membership

```bash
sendmaxx lists add <listId> --contacts id1,id2,id3
sendmaxx lists remove <listId> --contacts id1,id2
```

Only `manual` lists accept direct add/remove. For smart/tagged/icp, change the query/tag/ICP instead.

### Freeze a dynamic list

```bash
sendmaxx lists convert-to-manual <id>
```

Materializes current members and turns the list into a `manual` snapshot.

---

## Campaigns

### List, get

```bash
sendmaxx campaigns list
sendmaxx campaigns get <id>
```

### Create (always as draft)

```bash
sendmaxx campaigns create --name "Q2 Founders Outreach" --steps-file ./steps.json
```

`steps.json` shape:

```json
[
  {
    "subjectA": "Quick question about {{company}}",
    "subjectB": "{{first_name}}, 30 seconds?",
    "bodyHtml": "<p>Hey {{first_name}}, ...</p>",
    "bodyText": "Hey {{first_name}}, ...",
    "delayDays": 0
  },
  {
    "subjectA": "Re: Quick question",
    "bodyHtml": "<p>Following up...</p>",
    "delayDays": 3
  }
]
```

- Step 0 fires immediately on launch; later steps wait `delayDays` after the previous send.
- `bodyHtml` is rendered through markdown.
- Set `abTestEnabled` and audience via the SendMaxx UI for now (CLI doesn't expose those).

### Performance

```bash
sendmaxx campaigns performance <id>
```

Returns per-step and per-variant analytics.

---

## Replies

### List

```bash
sendmaxx replies list [--tag <tag>] [--limit <n>]
```

### Tag

```bash
sendmaxx replies tag <reply-id> --tag interested
```

Valid tags: `interested`, `meeting_booked`, `not_now`, `unsubscribed`, `unknown`. `meeting_booked` and `unsubscribed` also escalate the contact's stage.

### Force a Gmail sync

```bash
sendmaxx replies sync
```

Only when the user explicitly wants the latest replies right now — the background poller runs every 5 minutes anyway.

---

## Pipeline

```bash
sendmaxx pipeline status              # stage counts
sendmaxx pipeline contacts [--stage <s>]
sendmaxx pipeline export --format csv > pipeline.csv
```

---

## ICPs

Available under either `icps` or `icp`.

```bash
sendmaxx icps list
sendmaxx icps get <slug>

sendmaxx icps create --name "Vertical SaaS founders" \
  --slug vertical-saas --content-file ./icp.md
# Auto-spawns an ICP-derived list unless --no-list.

sendmaxx icps update <slug> --content-file ./icp.md
sendmaxx icps archive <slug>
sendmaxx icps restore <slug>
sendmaxx icps delete <slug>     # hard delete; cascades the derived list
```

---

## Tags

```bash
sendmaxx tags list
```

Returns each tag with total count and per-source breakdown (`claude` vs `user`).

---

## Alerts

```bash
sendmaxx alerts                      # untagged replies, stale contacts, etc.
sendmaxx stale --days 90             # contacts needing re-enrichment
```

---

## Common workflows

### Enrich and add new contacts

1. Use the Contact Discovery skill to find candidates and enrich data from Apollo, Hunter, etc.
2. Cross-reference fields, score confidence.
3. Write to a JSON file, then `sendmaxx contacts add --file enriched.json`. Duplicates are skipped by email.
4. Optionally `sendmaxx lists add <listId> --contacts ...` to organize them, or tag them with `sendmaxx contacts tag`.

### Create a campaign for review

1. Draft subject + markdown body for each step, save as `steps.json`.
2. `sendmaxx campaigns create --name "..." --steps-file steps.json` — this is a draft.
3. Tell the user to review and launch in the SendMaxx UI.

### Process replied contacts

1. `sendmaxx replies list` (or filter `--tag unknown` for untriaged).
2. For each, read `fullBody` and pick the right tag.
3. `sendmaxx replies tag <id> --tag <tag>`.
4. Optionally draft a follow-up using the Outreach Drafting skill.

### Re-enrich stale contacts

1. `sendmaxx stale --days 90` to find candidates.
2. For each, re-pull from sources but only update fields that are empty server-side (use `sendmaxx contacts get <id>` to check first).
3. `sendmaxx contacts update <id> --json '{...}'` with only the new fields.

---

## When the CLI isn't enough

A few capabilities aren't in the CLI yet: launching/pausing campaigns, managing campaign audience after creation, dialer/calls, workspace settings, and the dashboard view. For those, direct the user to the SendMaxx UI rather than falling back to raw HTTP.

If you hit a command-not-found or `Unknown command` error from `sendmaxx`, the user's CLI may be out of date. Suggest they upgrade their install of `@ethiqueai/cli`.

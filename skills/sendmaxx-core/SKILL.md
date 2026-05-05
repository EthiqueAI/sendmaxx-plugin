---
name: sendmaxx-core
description: Use this skill when reading or writing data in SendMaxx — contacts, campaigns, lists, pipeline, replies, calls, settings. Required for any SendMaxx interaction.
---

# SendMaxx Core API

You interact with SendMaxx via its HTTP API. SendMaxx is the system of record for outbound sales — contacts, campaigns, pipeline, calls, replies. You handle the intelligence layer: discovery, enrichment, drafting, scoring.

## Authentication

Every request requires a Bearer token in the `Authorization` header. The token starts with `sm_` and is provided by the user when they install this plugin.

**Resolving the token and base URL** — check these in order, use the first one that's set:

| Variable | Source |
|---|---|
| `$CLAUDE_PLUGIN_OPTION_api_token` / `$CLAUDE_PLUGIN_OPTION_base_url` | Plugin `userConfig` (preferred — set by Claude Code when this plugin is installed) |
| `$SENDMAXX_API_TOKEN` / `$SENDMAXX_BASE_URL` | Legacy env vars (e.g. from `~/.sendmaxx/env` for users who installed via curl) |

A safe pattern at the start of any session that calls SendMaxx:

```bash
TOKEN="${CLAUDE_PLUGIN_OPTION_api_token:-$SENDMAXX_API_TOKEN}"
URL="${CLAUDE_PLUGIN_OPTION_base_url:-$SENDMAXX_BASE_URL}"
if [ -z "$TOKEN" ] && [ -f ~/.sendmaxx/env ]; then
  source ~/.sendmaxx/env
  TOKEN="$SENDMAXX_API_TOKEN"
  URL="$SENDMAXX_BASE_URL"
fi

curl -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     "$URL/api/contacts"
```

### If neither is set

The plugin should have prompted at install time. If it didn't:

1. Tell the user to open Claude Code, run `/plugin`, find SendMaxx, and re-enter the URL + API token
2. Or have them generate a new token at their SendMaxx app's Settings → API Tokens

## Rate limits

- **30 requests/minute** for token-authenticated requests (per token)
- A 429 response means slow down; check `Retry-After` header

## Conventions

- All timestamps are integer Unix milliseconds (`Date.now()`)
- All IDs are nanoid strings unless noted
- Booleans on campaigns use `0`/`1` integers (e.g. `abTestEnabled: 1`)
- Contact emails are lowercase; uniqueness is enforced
- The `stage` hierarchy: `new` → `sent` → `opened` → `clicked` → `replied` → `meeting_booked`. Terminal: `bounced`, `unsubscribed`. Stages only escalate; never downgrade.

---

## How SendMaxx works

SendMaxx isn't just a database — it runs background jobs, enforces business rules, and coordinates Gmail and Telnyx. Understanding this is critical to using the API correctly.

### Architecture in one paragraph

SendMaxx is a Next.js app backed by SQLite (Turso). It owns the schema, scheduling, tracking, and compliance. Gmail and Telnyx are external services it talks to on behalf of the user. You (Claude) are *not* a service — you're a separate process that calls SendMaxx's API and external data APIs (Apollo, Hunter, etc.) using credentials the user provides. SendMaxx never sees those external API keys.

### What SendMaxx handles automatically (don't try to do these yourself)

| Concern | Behavior |
|---|---|
| **Email sending** | A cron job (every 2 minutes) finds due `campaign_contacts` and sends via Gmail API |
| **Send window** | Per-user timezone, hour range (default 8am–6pm), weekdays-only flag — enforced by scheduler |
| **Daily send limit** | Per-user counter, resets at UTC midnight, default 100/day, max 1000 |
| **Per-user throttle** | 1 email/sec to respect Gmail API rate limits |
| **Tracking pixels** | Auto-injected into every outbound email's HTML body |
| **Click rewriting** | Links in email bodies are rewritten through `/api/tracking/r/{token}` |
| **Reply detection** | Polls Gmail every 5 minutes, matches threads, auto-tags replies, removes contact from future steps |
| **Bounce detection** | Scans `mailer-daemon`/`postmaster` messages, marks contacts as `bounced`, removes from active sends |
| **Stage escalation** | Open → opened, click → clicked, reply → replied (escalation-only; never downgrades) |
| **Campaign completion** | When all enrollments reach a terminal status, campaign auto-marks `completed` |
| **Duplicate-send guard** | Unique DB index on `(campaign_contact_id, campaign_step_id)` prevents resending the same step |
| **Personalization** | Template variables substituted at send time (see below) |
| **Email signature** | Appended automatically to every outgoing email body |

### What Claude is responsible for

| Concern | What you do |
|---|---|
| **Contact discovery** | Call Apollo/LinkedIn/Hunter, write enriched rows via `POST /api/contacts/import` |
| **ICP scoring** | Read ICP, score contacts, update `confidenceScore`/`confidenceTier`/`confidenceReasoning` |
| **Email drafting** | Write subject + markdown body, post to `POST /api/campaigns` as draft |
| **Reply triage** | Read replies, tag them via `PATCH /api/replies` |
| **Call briefs** | Read contact + timeline + history, summarize for the user |
| **Post-call summaries** | Take a transcript, log it via `POST /api/dialer/calls` with notes + outcome |
| **Re-enrichment** | Find stale contacts and update their data — but never overwrite manual edits |

### Authentication & data access

- One API token = one user. Tokens scope you to that user's data.
- `data_visibility` is a global setting:
  - `org` (default): all users in the workspace see all contacts/campaigns. Useful for shared founder tools.
  - `user`: each user only sees what they created (`createdBy = userId`).
- Use `GET /api/contacts` and check the `orgMode` field in the response if you need to know which mode is active.
- Tokens are SHA-256 hashed at rest. The raw value is shown only once at creation.
- Token-authenticated requests are rate-limited to **30/min**. Session-authenticated requests get 60/min.

### The campaign lifecycle

```
[draft] -- POST /launch --> [active] -- POST /pause --> [paused] -- POST /resume --> [active]
                                |
                                | scheduler completes all enrollments
                                v
                            [completed]
```

- **draft** — editable: steps, audience, A/B flag. Nothing is being sent.
- **active** — scheduler picks up due `campaign_contacts` and sends. PATCHing steps is risky and may not behave well.
- **paused** — scheduler skips this campaign. Existing enrollments keep their `currentStep` and `nextSendAt`. Resume to continue.
- **completed** — auto-set when every enrollment is in a terminal status (`completed`, `bounced`, `unsubscribed`, `replied`).

When you launch:
- All `pending` enrollments switch to `in_progress` with `nextSendAt: now`
- Any `contactIds` passed in `POST /launch` are added and assigned A/B variants
- The scheduler will pick them up within 2 minutes

### How a single send actually works (mental model)

1. Cron fires every 2 minutes → `processDueSends()`
2. Query: `campaign_contacts WHERE status = 'in_progress' AND nextSendAt <= now`
3. For each due enrollment:
   - Skip if user's daily limit hit
   - Skip if outside send window (timezone-aware, weekday-only honored)
   - Fetch the campaign step at `currentStep`
   - Pick subject A or B based on `abVariant`
   - Apply template variables
   - Render markdown to HTML, inject tracking pixel + rewrite links
   - Append email signature
   - Send via Gmail API (throttled to 1/sec per user)
   - Insert `sent_emails` row (unique constraint prevents duplicates)
   - Schedule next step or mark `completed`
   - Increment user's daily count

If anything fails mid-loop, the next cron tick retries the remaining enrollments.

### A/B testing mechanics

- Set `abTestEnabled: 1` on the campaign
- Each `campaign_step` has `subjectA` (required) and `subjectB` (optional)
- At launch, contacts are split 50/50 deterministically: even index → "A", odd → "B"
- The variant persists across all steps for that contact
- If a step has no `subjectB`, variant B contacts fall back to `subjectA`
- Per-variant analytics are at `GET /api/campaigns/{id}/analytics` → `abVariants` field
- **Body content does not vary** — only subject lines. This is a constraint of the current schema.

### Reply detection details

- Gmail polling runs every 5 minutes
- Looks for threads matching tracked `gmail_thread_id`s where the latest message isn't from the user
- Tags replies automatically based on keyword matching:
  - "unsubscribe", "remove me" → `unsubscribed`
  - "meeting", "call", "schedule" → `meeting_booked`
  - "not now", "later", "busy" → `not_now`
  - "interested", "tell me more", "yes" → `interested`
  - Otherwise → `unknown`
- **Reply stops the sequence**: the enrollment's `status` becomes `replied` (or `unsubscribed`), so the scheduler skips it for all future steps. Don't re-send.
- The contact's `stage` escalates accordingly via the stage hierarchy

When you tag a reply via `PATCH /api/replies`:
- Tagging as `meeting_booked` updates the contact's stage to `meeting_booked`
- Tagging as `unsubscribed` updates the contact's stage to `unsubscribed`
- Other tags are metadata only

### Open and click tracking

- **Opens**: 1×1 transparent GIF served from `/api/tracking/open?pid={id}`. Counts only after 5 seconds (filters out scanner pre-fetch).
- **Clicks**: Links rewritten as `/api/tracking/r/{token}?dest={url}`. The token resolves the original URL and 302-redirects.
- Both events escalate the contact's stage and increment campaign counters
- Open tracking is **directionally useful, not precise** — Apple Mail Privacy Protection pre-loads pixels, image-blocking clients skip them entirely

### Personalization variables

Available in `bodyHtml` and `bodyText`:

| Variable | Falls back to |
|---|---|
| `{{first_name}}` | "there" |
| `{{last_name}}` | empty string |
| `{{name}}` | full name from first + last |
| `{{company}}` | "your company" |
| `{{title}}` | empty string |

Substitution is case-insensitive (`{{First_Name}}` works). It happens at send time on the body string before markdown rendering.

### Bounce handling

- The reply-polling cron also scans for delivery failure messages
- Detection looks at sender (`mailer-daemon`, `postmaster`) and subject keywords
- Extracted email addresses are matched to contacts; matched contacts are marked `bounced`
- Bounced contacts are dropped from active sequences (their enrollment status becomes `bounced`)
- A bounced contact will never receive future sends from any campaign — **don't try to "fix" this by re-enrolling them**

### Data visibility & ownership

- Most resources are scoped by `createdBy` (the user who created them)
- `GET /api/contacts` and `GET /api/campaigns` honor `data_visibility`:
  - In `org` mode, you see all contacts/campaigns
  - In `user` mode, you see only what the token's user created
- Mutations (`PATCH`, `DELETE`) check ownership regardless of visibility — you can only modify what you created
- 403 means you don't own the resource; respect this and don't retry

### What you cannot do via the API (yet)

- Send a one-off email outside a campaign (campaigns are the only send mechanism)
- Edit a `sent_emails` row (immutable)
- Update a single `campaign_step` in isolation (PATCH replaces all steps; only do this on draft campaigns)
- Update an existing call entry (current `POST /api/dialer/calls` only creates; updates require UI)
- Vary email body per A/B variant (subject lines only)
- Re-send a step to a contact (unique-index guard prevents this — and you shouldn't anyway)

### Common mistakes to avoid

- **Don't auto-launch campaigns.** Always create as draft and let the user click launch in the UI. They want to review before sending.
- **Don't overwrite manual edits.** When re-enriching, use field-level merging — never blindly PATCH every field. Manual user input is sacred.
- **Don't set `stage` directly to a lower value.** The stage hierarchy is enforced server-side; downgrade attempts are silently ignored.
- **Don't poll for new replies.** SendMaxx's reply-polling job already does this. Just call `GET /api/replies` when the user asks.
- **Don't try to bypass the daily limit.** The scheduler enforces it; sending more in one cron tick won't help.
- **Don't import contacts one at a time.** Use `POST /api/contacts/import` with a batch — much faster and dedups automatically.
- **Don't fabricate data.** If you don't have a phone number, leave the field empty rather than guessing. Confidence scoring depends on real source agreement.

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
id, campaignId, stepIndex (0-based), subjectA, subjectB?,
bodyHtml, bodyText?, delayDays, createdAt
```

### CampaignContact (enrollment)
```
id, campaignId, contactId, abVariant (A|B),
currentStep, status (pending|in_progress|completed|replied|bounced|unsubscribed),
stage, nextSendAt, sourceListId, createdAt
```

### Reply
```
id, contactId, sentEmailId, gmailThreadId, gmailMessageId,
snippet, fullBody, tag (interested|meeting_booked|not_now|unsubscribed|unknown),
receivedAt, syncedAt
```

### Call
```
id, contactId, userId, phoneNumber, callerNumber, telnyxCallId,
status (initiated|ringing|active|completed|failed|no_answer),
durationSeconds, outcome (no_answer|not_interested|callback|meeting_booked),
notes, startedAt, answeredAt, endedAt, createdAt
```

---

## Contacts

### Search contacts

```
GET /api/contacts?search=&stage=&state=&company=&title=
```

Response: `{ contacts: Contact[], orgMode: boolean }`

Special meta query for unique values:
- `?meta=county` → `string[]`
- `?meta=company` → `string[]`
- `?meta=title` → `string[]`

### Get one contact

```
GET /api/contacts/{id}
```

### Create contact

```
POST /api/contacts
{ "email": "...", "firstName": "...", "lastName": "...",
  "company": "...", "title": "...", "phone": "...",
  "state": "...", "county": "...", "notes": "..." }
```

`email` is required. Returns 409 if it already exists. Other fields optional.

### Update contact

```
PATCH /api/contacts/{id}
{ "firstName": "...", "title": "...", "phone": "..." }
```

Manual updates take priority. When you re-enrich, **never overwrite** non-empty fields without confirming.

### Delete contact

```
DELETE /api/contacts/{id}
```

Cascades to campaign enrollments.

### Bulk delete

```
DELETE /api/contacts/bulk
{ "ids": ["...", "..."] }
```

### Bulk import (recommended for enrichment workflows)

```
POST /api/contacts/import
{ "rows": [{ "email": "...", "firstName": "...", ... }, ...] }
```

Response: `{ imported: number, skipped: number, errors: string[] }`

Deduplicates by email automatically. Existing emails are skipped, not overwritten.

### Contact timeline (activity history)

```
GET /api/contacts/{id}/timeline
```

Response: `{ contact: Contact, events: [{ type, timestamp, label, meta? }] }`

Events: `enrolled`, `sent`, `opened`, `clicked`, `replied`. Sorted chronologically.

---

## Lists

Lists group contacts for targeting. Adding all members of a list to a campaign is one call.

### List all lists

```
GET /api/lists
```

Response: `[{ id, name, description, memberCount, createdAt, updatedAt }]`

Add `?contactId={id}` to also receive `isMember: boolean` per list.

### Get one list with members

```
GET /api/lists/{id}
```

Response: `{ ...list, contacts: Contact[] }`

### Create list

```
POST /api/lists
{ "name": "...", "description": "..." }
```

### Update list

```
PATCH /api/lists/{id}
{ "name": "...", "description": "..." }
```

### Delete list

```
DELETE /api/lists/{id}
```

### Add contacts to list

```
POST /api/lists/{id}/members
{ "contactIds": ["...", "..."] }
```

### Remove contacts from list

```
DELETE /api/lists/{id}/members
{ "contactIds": ["...", "..."] }
```

---

## Campaigns

### List campaigns

```
GET /api/campaigns
```

Response: `{ campaigns: [{ ...campaign, steps, contactCount }], orgMode }`

### Get one campaign

```
GET /api/campaigns/{id}
```

### Create campaign

```
POST /api/campaigns
{
  "name": "Q2 Founders Outreach",
  "abTestEnabled": 1,
  "steps": [
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
  ],
  "contactIds": ["..."]
}
```

- Step 0 fires immediately on launch; later steps wait `delayDays` after the previous send
- Body supports template variables: `{{first_name}}`, `{{last_name}}`, `{{name}}`, `{{company}}`, `{{title}}`
- `bodyHtml` is rendered through markdown — write markdown for best results
- If `abTestEnabled: 1`, contacts are split 50/50 between subjectA and subjectB
- `contactIds` pre-enrolls contacts as `pending` (won't send until launch)

### Update campaign

```
PATCH /api/campaigns/{id}
{ "name": "...", "abTestEnabled": 0|1, "steps": [...] }
```

Replaces all steps. Only modify campaigns that aren't `active`.

### Launch campaign

```
POST /api/campaigns/{id}/launch
{ "contactIds": ["..."] }   // optional — adds these before launching
```

Response: `{ ok: true, enrolled: number }`

After launch, all contacts are set to `in_progress` with `nextSendAt: now`. The scheduler picks them up within 2 minutes.

### Pause / Resume

```
POST /api/campaigns/{id}/pause
POST /api/campaigns/{id}/resume
```

### Manage audience

```
GET /api/campaigns/{id}/audience
```

Response: `{ contacts: [...], lists: [...], individualCount, totalCount }`

```
POST /api/campaigns/{id}/audience
{ "contactIds": ["..."] }     // add individual contacts
{ "listId": "..." }           // OR add all members of a list

DELETE /api/campaigns/{id}/audience
{ "campaignContactIds": ["..."] }   // remove specific enrollments
{ "listId": "..." }                  // OR remove everyone from this list
```

### Analytics

```
GET /api/campaigns/{id}/analytics
```

Response:
```json
{
  "steps": [
    { "stepIndex": 0, "subjectA": "...", "subjectB": "...", "delayDays": 0,
      "sent": 100, "opened": 45, "openRate": "45.0",
      "clicked": 12, "clickRate": "12.0",
      "replied": 8, "replyRate": "8.0" }
  ],
  "abVariants": {
    "A": { "sent": 50, "openRate": "42.0", "clickRate": "10.0", "replyRate": "6.0" },
    "B": { "sent": 50, "openRate": "48.0", "clickRate": "14.0", "replyRate": "10.0" }
  }
}
```

`abVariants` only present if `abTestEnabled`.

### Recipients with engagement

```
GET /api/campaigns/{id}/recipients
```

Per-contact engagement: `emailsSent`, `opens`, `firstOpenAt`, `clicks`, `lastClickAt`, `clickDetails[]`, `replied`, `openHistory[]`.

### Update an enrollment's stage manually

```
PATCH /api/campaign-contacts/{id}
{ "stage": "replied" }
```

---

## Replies

### List replies

```
GET /api/replies
```

Response: `[{ id, contact, sentEmailId, snippet, fullBody, tag, receivedAt, ... }]` (limit 100, sorted by `receivedAt` DESC)

### Tag a reply

```
PATCH /api/replies
{ "id": "...", "tag": "interested" | "meeting_booked" | "not_now" | "unsubscribed" | "unknown" }
```

`meeting_booked` and `unsubscribed` also update the contact's stage.

### Untagged count

```
GET /api/replies/count
```

Response: `{ untagged: number }`

---

## Pipeline

Active campaign contacts grouped by stage.

```
GET /api/pipeline
```

Response: `[{ id, contactId, name, email, company, stage, campaignId, campaignName }]`

---

## Dashboard

```
GET /api/dashboard
```

Response: `{ funnel: { totalContacts, sent, opened, clicked, replied, meetingBooked, bounced }, campaigns: [...], recentReplies: [...], untaggedReplies: number }`

---

## Calls

### Call history

```
GET /api/dialer/calls?contactId={id}
```

If `contactId` omitted, returns all calls for the user.

### Log a call

```
POST /api/dialer/calls
{
  "contactId": "...",
  "phoneNumber": "+15551234567",
  "callerNumber": "+15559876543",
  "outcome": "meeting_booked",
  "notes": "Great fit. Demo scheduled for Tuesday.",
  "durationSeconds": 420,
  "startedAt": 1714430000000,
  "endedAt": 1714430420000
}
```

`outcome`: `no_answer` | `not_interested` | `callback` | `meeting_booked`

---

## Settings

### Read settings

```
GET /api/settings
```

### Update a setting

```
POST /api/settings
{ "key": "...", "value": "..." }
```

Allowed keys: `email_signature`, `daily_limit`, `send_hour_start`, `send_hour_end`, `send_weekdays_only`, `timezone`, `display_name`, `data_visibility`.

---

## Common workflows

### Enrich and add new contacts

1. Use the Contact Discovery skill to find candidates and enrich data from Apollo, Hunter, etc.
2. Cross-reference fields, score confidence
3. `POST /api/contacts/import` with the array of rows — deduplication is automatic
4. Optionally `POST /api/lists/{id}/members` to organize them

### Create and launch a campaign

1. `POST /api/campaigns` with steps and (optionally) `contactIds`
2. User reviews in the UI
3. `POST /api/campaigns/{id}/launch` to begin sending

### Process a replied contact

1. `GET /api/replies` to find recent replies
2. Read the `fullBody` and decide the right tag
3. `PATCH /api/replies` with `{ id, tag }` — this also updates the contact's stage if the tag is `meeting_booked` or `unsubscribed`
4. Optionally draft a follow-up

## Errors

- **401** — bad/missing/revoked token
- **403** — token holder doesn't own this resource
- **404** — not found
- **409** — duplicate (e.g. existing email)
- **429** — rate limited; check `Retry-After`
- **400** — bad request body; the response usually has an `error` string

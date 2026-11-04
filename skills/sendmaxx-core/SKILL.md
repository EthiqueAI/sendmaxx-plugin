---
name: sendmaxx-core
description: Use this skill when reading or writing data in SendMaxx — contacts, campaigns, lists, pipeline, replies, ICPs, tags. Required for any SendMaxx interaction.
---

# SendMaxx Core

You interact with SendMaxx through its CLI: the `sendmaxx` command. SendMaxx is the system of record for outbound sales — contacts, campaigns, pipeline, calls, replies. You handle the intelligence layer: discovery, enrichment, drafting, scoring.

**Always use the CLI.** Do not call the HTTP API directly with `curl`. The CLI handles auth, base URL, pagination, and error formatting for you, and it is the supported interface.

## Archive first, delete almost never

SendMaxx contacts, lists, campaigns, and ICPs all support **archiving** — a reversible soft-hide. Archive is the default way to deal with anything no longer in active use. **Delete is the rare, explicit, nuclear option.**

| Operation | When to use it |
|---|---|
| `archive` | "We're done with this for now." Default. Restorable. Hides from default views. Scheduler skips archived contacts and campaigns. Smart lists exclude archived contacts transparently. |
| `delete` | The user has explicitly said "delete" / "wipe" / "permanently remove." Or you're cleaning up obvious test rows you just created in the same session. Cascades and cannot be undone. |

**The hard rule:** if the user says "remove", "clean up", "get rid of", "tidy", "drop" — that's archive. They need to say a permanent verb ("delete", "wipe", "permanently remove", "blow away") for you to use the destructive path. Even then, on anything with downstream data (a campaign with sent emails, a contact with replies), confirm once before running it.

Every destructive CLI command requires `--yes` so a typo can't trigger it.

To see archived items, pass `--include-archived` (mix with active) or `--archived-only` (just archived) on any `list` / `search` command.

```bash
sendmaxx contacts archive <id>          # default cleanup verb
sendmaxx contacts restore <id>          # undo
sendmaxx contacts search --archived-only
sendmaxx contacts delete <id> --yes     # nuclear; cascades
```

The same pattern is on `lists`, `campaigns`, and `icps`.

---

## Setup

The user authenticates once, **in their own terminal**:

```bash
sendmaxx auth login
```

It prompts for their SendMaxx URL and an API token (created in SendMaxx Settings, input hidden), validates it, and writes `~/.sendmaxx/config.json`. After that, every command picks up the token automatically.

To check status: `sendmaxx auth status`.

The token must never pass through the conversation. Don't ask the user to paste it into chat, and don't put one in a command yourself. If a command fails with "Not authenticated", or `auth login` says it has no interactive terminal, ask the user to run the command above in their own terminal, then continue once `sendmaxx auth status` shows `"authenticated": true`. Don't try to read tokens from env vars or config files yourself.

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

**Launching, pausing, resuming, and audience management ARE in the CLI** — see the Campaigns section below. Do not launch a campaign without explicit user approval; create the draft, summarize what you built, and wait for the user to say go.

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

- **Don't auto-launch campaigns.** `sendmaxx campaigns launch` exists, but you must not call it without explicit user approval for *this specific campaign*. Default flow: create a draft, summarize, wait. A general "go ahead and run campaigns" instruction earlier in the session does NOT pre-authorize a future launch.
- **Don't overwrite manual edits.** When re-enriching, only update fields that are empty or that you originally populated. Manual user input is sacred.
- **Don't set `stage` to a lower value.** The stage hierarchy is enforced server-side; downgrade attempts are silently ignored.
- **Don't poll for new replies.** SendMaxx polls Gmail every 5 minutes. Call `sendmaxx replies sync` only if the user explicitly asks for an immediate refresh.
- **Don't add contacts one at a time in a loop you wrote.** For pre-enriched data use `sendmaxx contacts add --file path.json` (array form). For large lists from a CSV or external source, use `sendmaxx contacts import --file ...` — it's a true bulk endpoint with one round-trip and proper dedupe.
- **Don't fabricate data.** If you don't have a phone number, leave the field empty. Confidence scoring depends on real source agreement.
- **Don't use `--pretty` in scripts.** It prints tables, not JSON.
- **Don't send personal emails without per-email approval.** Same rule as campaign launches: `sendmaxx emails send <id> --yes` goes to a named human. Show the draft, wait, then send.
- **Don't use campaign overrides at scale.** More than about five recipients each needing custom text means it's a different campaign, not a pile of overrides — and you'll hit the 30 req/min token rate limit partway through and leave a half-built set.
- **Don't reach for delete.** When the user wants to "remove", "clean up", "drop", "get rid of" — that's `archive`. `delete` is only when they use a permanent verb explicitly, and even then confirm on anything with downstream data. See "Archive first, delete almost never" at the top.

---

## Core entities

### Contact
```
id, email (unique, lowercase), firstName, lastName, company, title,
phone, state, county, notes,
stage, confidenceScore (0-10), confidenceTier (low|medium|high),
confidenceReasoning, sources (JSON array of source names),
enrichedAt, archivedAt (null = active, unix ms = archived),
createdBy, createdAt, updatedAt
```

### Campaign
```
id, name, status (draft|active|paused|completed),
abTestEnabled (0|1), totalSent, totalOpened, totalReplied,
totalMeeting, totalBounced, totalClicked,
archivedAt (null = active, unix ms = archived; orthogonal to status),
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
memberCount, source,
archivedAt (null = active, unix ms = archived),
createdAt
```

### ICP
```
slug, name, description, content (markdown),
status (active|archived), archivedAt (canonical archive marker; mirrors status),
parentIcpId?
```

### PersonalEmail
An individually-written email to one person — not a template. See the Personal Email skill.
```
id ("pe_" prefixed), kind (standalone|reply|campaign_override),
status (draft|approved|sending|sent|canceled|failed),
contactId, subject?, body (markdown),
gmailThreadId?, inReplyToMessageId?          -- kind=reply
campaignContactId?, stepIndex?               -- kind=campaign_override
scheduledAt?, trackingEnabled (0|1),
personalizationNotes?, sources?, authorSource (user|claude),
sentEmailId?, failureReason?,
archivedAt (null = active), createdAt, updatedAt
```
`subject` is null only on a `campaign_override`, where it inherits the step's subject.

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

### Timeline

```bash
sendmaxx contacts timeline <id>
```

Every interaction, oldest first: campaign enrolments, sends, opens, clicks, replies. Returns `{ contact, events: [{ type, timestamp, label, meta }] }`. Calls aren't in the timeline — use `sendmaxx calls list --contact <id>`.

### Add (one contact, or small enriched batch)

Single contact:
```bash
sendmaxx contacts add --json '{"email":"x@y.com","firstName":"Ada","company":"Acme"}'
```

Small batch from a JSON file (object or array):
```bash
sendmaxx contacts add --file ./enriched.json
```

The file can be a single object or an array. Array form loops one-by-one and reports `{ added, failed }` totals. `email` is required; duplicates are skipped.

**Use `contacts add` only when the data is already in SendMaxx's canonical shape.** For raw CSVs or large datasets, use `contacts import` (next).

### Import (CSV or JSON, bulk)

```bash
sendmaxx contacts import --file leads.csv
sendmaxx contacts import --file leads.csv --dry-run                           # parse & preview, no upload
sendmaxx contacts import --file leads.csv --mapping '{"Lead Email":"email","Org":"company"}'
sendmaxx contacts import --file leads.json --format json
```

Single round-trip, deduped by email server-side. Returns `{ imported, skipped, errors }`.

**Allowed fields (everything else is ignored):**

| Field | Required | Notes |
|---|---|---|
| `email` | yes | Lowercased server-side. Rows without a valid email are dropped. |
| `firstName` | | |
| `lastName` | | |
| `company` | | |
| `title` | | |
| `phone` | | |
| `state` | | US 2-letter code is conventional but not enforced. |
| `county` | | |
| `notes` | | Free text. |

**CSV format:**
- First row is the header. Cells may be quoted; embedded commas and `""`-escaped quotes are handled.
- Common header variants are auto-mapped: `Email Address`, `email_address`, `E-mail` → `email`; `First Name`, `first_name`, `Given Name` → `firstName`; `Job Title`, `Role`, `Position` → `title`; `Organization`, `Company Name`, `Org` → `company`; `Phone Number`, `Mobile`, `Tel` → `phone`; etc.
- If a column doesn't auto-map, pass `--mapping '{"<exact header text>":"<canonical field>"}'`. The canonical field must be one of the nine fields above.

**JSON format:** either a top-level array `[{...}, {...}]` or `{"rows":[{...}, {...}]}`. Use the canonical field names directly.

**Always run `--dry-run` first on data you didn't generate.** It returns `{ parsed, importable, droppedNoEmail, preview }` so you can confirm the mapping is right before the real upload.

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

### Archive / restore (preferred)

```bash
sendmaxx contacts archive <id>
sendmaxx contacts restore <id>
sendmaxx contacts search --archived-only
sendmaxx contacts search --include-archived           # active + archived
```

Archived contacts:
- Hidden from `contacts search` and from the contact pickers in the UI.
- **Excluded automatically from smart/tagged/ICP list resolution** — so any campaign launching from those lists won't enroll them.
- **Skipped by the scheduler** — even existing enrollments stop receiving sends until the contact is restored.
- Still appear on `contacts get <id>` directly (so you can inspect / restore them).
- Replies and call history are untouched.

Use this as your default cleanup mechanism. The contact's data, history, and confidence score all survive.

### Delete (rare, destructive)

```bash
sendmaxx contacts delete <id> --yes
```

Hard-deletes the contact, cascades to campaign enrollments, sent emails, click events, replies. **Cannot be undone.** Only use when the user has explicitly asked for permanent deletion. If you're unsure, archive instead and tell the user "I archived it; if you want it permanently gone, ask me to delete it."

### Bulk delete

```bash
sendmaxx contacts bulk-delete --ids id1,id2,id3 --yes
sendmaxx contacts bulk-delete --file ids.txt --yes      # one ID per line
```

One round-trip. `--yes` is required (the server cascade is permanent — deletes the contact, all enrollments, sent emails, clicks, replies, and bounce records for those contacts). Returns `{ deleted: <n> }`.

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

### Archive / restore (preferred for cleanup)

```bash
sendmaxx lists archive <id>
sendmaxx lists restore <id>
sendmaxx lists list --archived-only
sendmaxx lists list --include-archived
```

Archived lists:
- Hidden from `lists list` by default.
- **Campaigns that snapshotted from this list are unaffected** — the audience is frozen at add-time, so archiving the source list doesn't disenroll anyone.
- Smart/tagged/ICP lists can be archived too; the underlying query is preserved for when you restore.

### Delete (rare)

```bash
sendmaxx lists delete <id> --yes
```

Hard-delete. Cascades to list memberships. Prefer archive.

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
sendmaxx campaigns create --name "Q2 A/B test" --steps-file ./steps.json --ab-test
```

### Step schema (`steps.json`)

```json
[
  {
    "subjectA": "Quick question about your outbound stack",
    "subjectB": "30 seconds?",
    "bodyHtml": "<p>Hey {{first_name}}, saw {{company}} is hiring SDRs — ...</p>",
    "bodyText": "Hey {{first_name}}, saw {{company}} is hiring SDRs — ...",
    "delayDays": 0
  },
  {
    "subjectA": "Re: Quick question",
    "bodyHtml": "<p>Following up...</p>",
    "delayDays": 3
  }
]
```

Field-by-field:

| Field | Required | Behavior |
|---|---|---|
| `subjectA` | yes | Subject line. **No merge fields** — subjects are sent literally. Personalize the body, not the subject. |
| `subjectB` | no | A/B variant subject (only used when the campaign has `abTestEnabled: true`). |
| `bodyHtml` | yes | Markdown allowed — it's rendered through `marked()` at send time. Merge fields below are substituted before render. |
| `bodyText` | no | Plain-text alternative. Same merge-field rules. |
| `delayDays` | no, default `0` | Days to wait after the *previous* step before sending this one. Step 0 fires immediately on launch. |

**Merge fields (case-insensitive, BODY ONLY):**

| Token | Resolves to | Fallback |
|---|---|---|
| `{{first_name}}` | `contact.firstName` | `there` |
| `{{last_name}}` | `contact.lastName` | empty |
| `{{name}}` | first + last | first only |
| `{{company}}` | `contact.company` | `your company` |
| `{{title}}` | `contact.title` | empty |

If you write a merge field in a subject, it'll be sent as-is (e.g. `Hi {{first_name}}` in the inbox). Don't do that.

### Update (edit a draft)

```bash
sendmaxx campaigns update <id> --name "Q2 Founders v2"
sendmaxx campaigns update <id> --steps-file ./steps-v2.json     # REPLACES all steps
sendmaxx campaigns update <id> --ab-test
sendmaxx campaigns update <id> --no-ab-test
```

Sending `--steps-file` replaces the entire step list (it's not a patch — server deletes existing steps and re-inserts). Safe on drafts; on an `active` campaign this changes what future sends use, but already-sent emails are unaffected.

### Audience

A campaign's audience is the set of `campaign_contacts` enrollments. Contacts can be added by individual ID or via a list snapshot (smart/tagged/icp lists are resolved at the time of add — the snapshot is frozen).

```bash
sendmaxx campaigns audience show <id>                                    # current audience + source-list summary
sendmaxx campaigns audience add <id> --contacts ct_1,ct_2,ct_3
sendmaxx campaigns audience add <id> --list lst_abc                       # snapshot current list members
sendmaxx campaigns audience add <id> --contacts ct_4 --list lst_xyz       # both in one call
sendmaxx campaigns audience remove <id> --cc-ids cc_1,cc_2                # by campaignContact ID (from `audience show`)
sendmaxx campaigns audience remove <id> --list lst_abc                    # remove everything sourced from this list
```

Notes:
- `audience add` returns `{ added: <n> }`. Duplicates (same contact + same campaign) are silently no-op'd.
- `audience remove` operates on **campaignContact IDs**, not contact IDs. Get them from `audience show`.
- Removing the source list (e.g. `lists delete lst_abc`) does NOT remove enrollments — the audience is a snapshot.

### Launch, pause, resume

```bash
sendmaxx campaigns launch <id>                                            # uses existing audience
sendmaxx campaigns launch <id> --contacts ct_a,ct_b,ct_c                  # also enroll these at launch
sendmaxx campaigns pause  <id>
sendmaxx campaigns resume <id>
```

Launch requires at least one step and a non-empty audience — the server will 400 otherwise. Returns `{ ok: true, enrolled: <n> }`.

**Authorization rule:** the user must explicitly approve launching *this campaign*. A blanket "go" from earlier in the conversation does not extend to a new campaign. Show the user the campaign summary (name, audience size, step count, first send time) and wait.

### Recipients (read-only engagement)

```bash
sendmaxx campaigns recipients <id>
```

Returns a per-recipient row with `emailsSent`, `opens`, `firstOpenAt`, `clicks`, `clickDetails`, `replied`, `openHistory`, etc. Use this for triage — "who opened but didn't reply?" — not for audience management.

```bash
sendmaxx campaigns recipient-stage <campaignContactId> --stage <label>
```

Updates a single enrollment's free-form `stage` label (independent of the contact's overall `stage`).

### Performance

```bash
sendmaxx campaigns performance <id>
```

Returns per-step and per-variant analytics aggregated across the whole campaign.

### Archive / restore (preferred for cleanup)

```bash
sendmaxx campaigns archive <id>           # auto-pauses if active
sendmaxx campaigns restore <id>           # stays paused; use `resume` to start sending again
sendmaxx campaigns list --archived-only
sendmaxx campaigns list --include-archived
```

Archived campaigns:
- Auto-paused on archive if `status='active'` (response includes `autoPaused: true`).
- **Scheduler skips them** — no future sends.
- Hidden from `campaigns list` by default.
- All analytics, sent emails, click events, and replies are preserved.
- Restore leaves status at `paused` — call `sendmaxx campaigns resume <id>` to actually resume sending.

### Delete (rare, destructive)

```bash
sendmaxx campaigns delete <id> --yes
```

Cascades: deletes the campaign, its steps, all enrollments, sent emails, click events, replies tied to those emails. **Cannot be undone.** Prefer `archive` 99% of the time — archived campaigns retain their full analytics history, which deleted ones don't.

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

## Personal emails

Individually-written emails to one person. Three kinds share one resource. Full
guidance lives in the Personal Email skill; this is the command reference.

```bash
sendmaxx emails list [--status draft] [--kind reply] [--contact <id>]
sendmaxx emails get <id>              # --pretty renders the body and the "why"

# A standalone one-off. Write the real words — merge fields are a tell here.
sendmaxx emails draft --contact <id> --subject "..." --content "$(cat <<'EOF'
Hey Sarah, ...
EOF
)" --why "why this email says what it says"

# A reply into a live thread. Subject and threading headers come from the thread.
sendmaxx emails reply --contact <id> --thread <gmailThreadId> --content "..."

# Replace a campaign step's body for ONE recipient. ccIds come from
# `sendmaxx campaigns recipients <campaignId>`.
sendmaxx emails override --campaign-contact <ccId> --step 0 --content "..."

sendmaxx emails update <id> --content "..."   # editing an approved email returns it to draft
sendmaxx emails send <id> --yes [--now]       # --yes is required
sendmaxx emails cancel <id>
sendmaxx emails archive <id> / restore <id> / delete <id> --yes
```

Also registered as `sendmaxx email` (singular).

`send` without `--now` approves the email and lets the scheduler deliver it on its
next tick, inheriting the send window, daily cap and throttle. `--now` bypasses only
the send-hour window.

A `campaign_override` is delivered by the campaign itself, at the step you targeted —
`send` just approves it. An override still in `draft` when its step comes due is
skipped: the template sends and the override is marked `canceled`. Note also that
re-launching an active campaign resets `nextSendAt` for the *whole* audience, so an
override on a later step can fire earlier than you'd expect.

---

## Pipeline

```bash
sendmaxx pipeline status              # stage counts
sendmaxx pipeline contacts [--stage <s>]
sendmaxx pipeline export --format csv > pipeline.csv
```

---

## ICPs

**ICPs are server-first.** They live in SendMaxx's `icps` table, addressed by `slug`. There is no canonical local file — `sendmaxx icps get <slug>` is the source of truth. Any local markdown is a draft or cache, never authoritative.

Available under either `icps` or `icp`.

```bash
sendmaxx icps list
sendmaxx icps list --archived-only
sendmaxx icps list --include-archived
sendmaxx icps get <slug>

sendmaxx icps create --name "Vertical SaaS founders" \
  --slug vertical-saas --content-file ./icp.md
# Auto-spawns a contact list (type: "icp", icpSlug: <slug>, name: "ICP: <name>").
# That list is the execution vehicle — discovery adds prospects to it, campaigns target it.
# Add --no-list to skip (rare).

sendmaxx icps update <slug> --content-file ./icp.md
# Updates the ICP content. The auto-spawned list is preserved (same id, same memberships).

# Archive / restore (preferred for cleanup)
sendmaxx icps archive <slug>                 # cascade-archives the linked list too
sendmaxx icps archive <slug> --keep-list     # archive the ICP but leave the list active
sendmaxx icps restore <slug>                 # restores the ICP AND the linked list (if it was archived alongside)

# Hard-delete (rare; cascades to the auto-spawned list which is also hard-deleted)
sendmaxx icps delete <slug> --yes
```

**ICP archive cascade rules:**
- Default: archiving an ICP also archives its auto-spawned `type=icp` list. This keeps the "ICP is dormant → so is its execution vehicle" semantics intact.
- `--keep-list`: archives only the ICP. Use this when the user wants to retire the ICP doc but keep working with the underlying audience (rare — usually if you do this you should convert the list to `manual` first so it doesn't try to resolve against an archived ICP).
- Restore: symmetric. Restoring an ICP also un-archives its linked list if the list was archived. (Edge case: if the user separately re-archived the list while the ICP was archived, restoring the ICP un-archives the list anyway. Rare enough to ignore.)

To re-parent or unparent an existing ICP after creation, pass `--parent-icp-id <id>` (or `--parent-icp-id ""` to unparent) to `sendmaxx icps update`.

### Sub-ICPs

Pass `--parent-icp-id <id>` on create to model a child segment (e.g. "Banking" under a "FinTech" parent). Each sub-ICP gets its own auto-spawned list.

### Contact ↔ ICP linkage

Contacts do **not** carry an `icpId`. The relationship is indirect:

```
ICP --(auto-spawned)--> list (type: "icp", icpSlug)  <--(membership)--  Contact
```

To find the list for a given ICP:

```bash
sendmaxx lists list | jq '.lists[] | select(.type == "icp" and .icpSlug == "<slug>")'
```

Fit scoring per contact is stored in the contact's `confidenceScore`, `confidenceTier`, and `confidenceReasoning` fields — there's no separate ICP-fit table.

---

## Tags

```bash
sendmaxx tags list
```

Returns each tag with total count and per-source breakdown (`claude` vs `user`).

---

## Calls

Calls placed through the in-app dialer are logged automatically. Use these to read call history and record outcomes.

```bash
sendmaxx calls list [--contact <id>] [--outcome <outcome>] [--limit <n>]
sendmaxx calls update <call-id> [--outcome <outcome>] [--notes "..."] [--status <status>] [--duration <seconds>]
sendmaxx calls log --contact <id> [--outcome <outcome>] [--notes "..."] [--duration <seconds>] [--started-at <iso>] [--phone <n>] [--caller <n>]
```

- Without `--contact`, `list` returns the calls you placed, newest first.
- `update` only works on calls you placed. `--notes` replaces the existing notes.
- `log` is for calls with no record yet, e.g. made from a phone outside the dialer. `--phone` defaults to the contact's phone and `--caller` to the user's caller number in Settings → Dialer. Always `calls list --contact <id>` first so you don't log a duplicate.
- Outcomes: `no_answer`, `not_interested`, `callback`, `meeting_booked`. Statuses: `initiated`, `ringing`, `active`, `completed`, `failed`, `no_answer`.
- Setting an outcome does **not** change the contact's stage. For `meeting_booked`, also run `sendmaxx contacts update-stage <id> --stage meeting_booked`.

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

### Bulk import a CSV the user gave you

1. Eyeball the file — note the header row, look for non-obvious columns.
2. `sendmaxx contacts import --file file.csv --dry-run`. Read the returned `preview`; confirm every column you care about made it into the canonical fields. If `droppedNoEmail` is high, something's off — maybe the email column has an unrecognized header. Use `--mapping '{"<header>":"email"}'`.
3. Once `--dry-run` looks right, drop the flag and run for real.
4. If you want the new contacts in a list, follow up with `sendmaxx lists add` (manual list) or rely on a tag-based / smart list that already matches them.

### Create a campaign end-to-end

1. Draft subject + markdown body for each step, save as `steps.json` (see schema above).
2. `sendmaxx campaigns create --name "..." --steps-file steps.json` — captures the campaign ID from the response.
3. Add audience: `sendmaxx campaigns audience add <id> --list <listId>` (snapshot) or `--contacts <ids>` (direct).
4. `sendmaxx campaigns audience show <id>` and `sendmaxx campaigns get <id> --pretty` — show the user what's about to go out.
5. **Wait for explicit approval.** When the user says go, `sendmaxx campaigns launch <id>`.
6. Watch progress: `sendmaxx campaigns performance <id>` or `sendmaxx campaigns recipients <id>`. Pause with `sendmaxx campaigns pause <id>` if something looks wrong.

### Process replied contacts

1. `sendmaxx replies list` (or filter `--tag unknown` for untriaged).
2. For each, read `fullBody` and pick the right tag.
3. `sendmaxx replies tag <id> --tag <tag>`.
4. Optionally draft a follow-up using the Outreach Drafting skill.

### Tidy up — archive everything stale, don't delete

When the user says "clean up that list", "get rid of these contacts", "remove this campaign", or any soft-cleanup phrase:

1. **Archive, don't delete.** `sendmaxx <entity> archive <id>` for each.
2. Confirm in plain language: "Archived 12 contacts and the 'Q1 SDR push' campaign. Nothing was deleted — say `restore` (or 'undo') if you change your mind."
3. If the user comes back asking "where did X go" — `sendmaxx <entity> list --archived-only` to find it, then `restore`.
4. Only escalate to `delete --yes` if the user uses a permanent verb explicitly. Even then, on anything with downstream data (campaign with sent emails, contact with replies), confirm once: "This permanently wipes 47 sent emails and 3 replies tied to that campaign. Proceed?"

### Re-enrich stale contacts

1. `sendmaxx stale --days 90` to find candidates.
2. For each, re-pull from sources but only update fields that are empty server-side (use `sendmaxx contacts get <id>` to check first).
3. `sendmaxx contacts update <id> --json '{...}'` with only the new fields.

---

## When the CLI isn't enough

A few capabilities aren't in the CLI yet: placing calls (the dialer is browser-only), workspace settings, API-token management, and the dashboard view. For those, direct the user to the SendMaxx UI rather than falling back to raw HTTP.

If you hit a command-not-found or `Unknown command` error from `sendmaxx`, the user's CLI may be out of date. Suggest they upgrade their install of `@ethiqueai/sendmaxx-cli`.

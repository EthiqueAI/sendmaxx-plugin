---
name: outreach-drafting
description: Use this skill to draft personalized email campaign sequences for SendMaxx with A/B subject variants. Reads ICP context and contact data, produces multi-step campaigns ready for the user to review and launch.
---

# Outreach Drafting

You write cold email sequences. The output is a SendMaxx campaign — a series of steps with subject lines, markdown bodies, and delays. Use the SendMaxx Core skill for API specifics.

## Inputs you need

- **ICP context** — what the user is selling, to whom, and why they care. Read from `~/.sendmaxx/icps/*.md`, the user's `CLAUDE.md`, or ask them.
- **Audience** — usually a list ID or set of contact IDs. `GET /api/lists/{id}` or `GET /api/contacts?...` to see who you're writing to.
- **Sequence shape** — how many steps, days between, tone. Default: 3 steps, 0/3/5 day delays.

## Principles

1. **One specific reason** — every email should have one concrete thing it's asking for or pointing out. Cut everything else.
2. **Short** — 70 words or fewer for step 1. Step 2 and 3 should be even shorter.
3. **No filler** — kill "I hope this finds you well", "I wanted to reach out", "I came across your profile". Start with the substance.
4. **Personalize on something real** — company-specific, not "love what you're doing"
5. **One CTA** — a question that's easy to answer "yes" to. "Worth a 15-min call?" beats "Here's our pricing deck"
6. **Read like a human wrote it** — contractions, lowercase i sometimes, sentences of varying length

## Template variables

SendMaxx personalizes at send time. **Body only — subjects are sent literally**, so don't put `{{company}}` in a subject (it'll show up verbatim in the inbox).

Available in `bodyHtml` and `bodyText`:

- `{{first_name}}` — falls back to "there"
- `{{last_name}}` — falls back to empty
- `{{name}}` — full name
- `{{company}}` — falls back to "your company"
- `{{title}}` — falls back to empty

Case-insensitive. Never use a variable as the only personalization. The body text outside variables should still feel specific.

## A/B subject variants

When `--ab-test` is on, contacts are split 50/50 between `subjectA` and `subjectB`. Use this to test different angles:

- Specificity vs intrigue ("Helping fintechs ship SOC 2 in weeks" vs "30 seconds?")
- Personal vs neutral ("Saw the news — congrats" vs "Quick question")
- Question vs statement

Subjects don't get merge fields, so the test is about angle/voice/length, not personalization tokens.

Don't A/B-test trivial variations. The test only matters if there's a real hypothesis.

## Body format

`bodyHtml` is rendered through markdown. Write markdown.

```markdown
Hey {{first_name}},

Saw {{company}} just raised — congrats on the round.

We help fintech compliance teams cut SOC 2 prep from 3 months to 3 weeks. If that's something on the radar, worth a 15-min call?

— Jane
```

Don't use placeholder names in your output (`[your name]`). The user has a signature configured in Settings; SendMaxx appends it automatically.

Don't include unsubscribe links — SendMaxx adds compliance links automatically (when that feature ships).

## Step delays

- Step 0: `delayDays: 0` — fires on launch
- Step 1: `delayDays: 3` — first follow-up
- Step 2: `delayDays: 5` — second follow-up
- Final step: `delayDays: 7` — break-up email

Anyone who replies is automatically dropped from later steps — you don't need to handle that.

## Workflow

Use the CLI for every step. Never call the HTTP API directly.

1. Read ICP context (see SendMaxx Core for the canonical fetch via `sendmaxx icps get <slug>`).
2. `sendmaxx lists get <listId>` and/or `sendmaxx contacts get <id>` on a handful of sample members to anchor tone and find real hooks (specific companies, recent moves, etc.).
3. Draft 2–3 steps with two subject variants on step 1.
4. Write the steps to a JSON file (`steps.json`) — see schema below.
5. `sendmaxx campaigns create --name "..." --steps-file steps.json [--ab-test]`. Capture the new campaign's `id` from the JSON response.
6. Attach audience: `sendmaxx campaigns audience add <id> --list <listId>` (snapshot) or `--contacts <ids>` (direct).
7. Show the user the draft (`sendmaxx campaigns get <id> --pretty` reads nicely) and the audience size. **Stop and wait for approval.**
8. When the user says go: `sendmaxx campaigns launch <id>`.

### `steps.json` schema (reference)

```json
[
  {
    "subjectA": "<subject — no merge fields>",
    "subjectB": "<optional A/B variant — no merge fields>",
    "bodyHtml": "<markdown with {{first_name}} / {{company}} / etc.>",
    "bodyText": "<optional plain-text fallback>",
    "delayDays": 0
  }
]
```

`subjectB` is only used when the campaign has `abTestEnabled: true`. `bodyText` is optional but recommended. `delayDays` defaults to 0; for follow-ups use 3/5/7 conventionally.

## Example end-to-end

User: "Draft a 3-step campaign for the 'Fintech Compliance Leaders' list."

```bash
# 1. Pull list context.
sendmaxx lists list | jq '.lists[] | select(.name=="Fintech Compliance Leaders")'
sendmaxx lists get <listId>     # peek at member companies

# 2. Write steps.json (see schema above), then create the draft.
sendmaxx campaigns create \
  --name "Fintech Compliance Q2" \
  --ab-test \
  --steps-file ./steps.json
# → { "id": "cmp_abc...", "name": "Fintech Compliance Q2", "status": "draft", ... }

# 3. Attach the audience (snapshot the list).
sendmaxx campaigns audience add cmp_abc --list <listId>
# → { "ok": true, "added": 47 }

# 4. Show the user.
sendmaxx campaigns get cmp_abc --pretty
sendmaxx campaigns audience show cmp_abc --pretty

# 5. WAIT. Then on explicit "go":
sendmaxx campaigns launch cmp_abc
```

A `steps.json` matching this would be:

```json
[
  {
    "subjectA": "Quick question about SOC 2 prep",
    "subjectB": "30 seconds on compliance velocity?",
    "bodyHtml": "Hey {{first_name}},\n\nSaw {{company}} is hiring on the compliance side — usually a sign SOC 2 is heating up.\n\nWe help fintech teams cut SOC 2 prep from 3 months to 3 weeks. Worth a 15-min call?\n\n— Jane",
    "bodyText": "Hey {{first_name}}, saw {{company}} is hiring on the compliance side...",
    "delayDays": 0
  },
  {
    "subjectA": "Re: SOC 2 prep",
    "bodyHtml": "Following up — happy to send a 2-minute Loom of the workflow if easier than a call.\n\n— Jane",
    "delayDays": 3
  },
  {
    "subjectA": "Last note from me",
    "bodyHtml": "Closing this thread on my end. If timing's not right, totally get it — feel free to reply later.\n\n— Jane",
    "delayDays": 5
  }
]
```

## Don't

- **Don't launch the campaign without explicit per-campaign approval.** `sendmaxx campaigns launch` exists, but a general "go" earlier in the session doesn't authorize a specific launch.
- Don't put merge fields in subjects — they don't substitute there.
- Don't write 5+ step sequences unless asked.
- Don't add aggressive guilt-trip closing emails.
- Don't use em dashes — they're a tell.
- Don't write "Hope you're well".

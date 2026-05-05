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

SendMaxx personalizes at send time. Use these in your bodies — they're substituted from the contact record:

- `{{first_name}}` — falls back to "there" if missing
- `{{last_name}}`
- `{{name}}` — full name
- `{{company}}` — falls back to "your company"
- `{{title}}`

Never use a variable as the only personalization. The body text outside variables should still feel specific.

## A/B subject variants

When `abTestEnabled: 1`, contacts are split 50/50 between `subjectA` and `subjectB`. Use this to test:

- Different angles ("Quick question about {{company}}" vs "{{first_name}}, 30 seconds?")
- Specificity vs intrigue ("Helping {{company}} with X" vs "Saw the news on {{company}}")
- Personal vs neutral

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

1. Read ICP context
2. `GET /api/contacts/{id}` for a few sample audience members to anchor the tone
3. Draft 2–3 steps with two subject variants on step 1
4. `POST /api/campaigns` with the steps
5. Tell the user the campaign is in draft, share a couple of preview lines, and let them launch from the UI

## Example

User: "Draft a 3-step campaign for the 'Fintech Compliance Leaders' list."

```
1. GET /api/lists?... → find the list ID
2. GET /api/lists/{id} → see member companies for context
3. POST /api/campaigns:
   {
     "name": "Fintech Compliance Q2",
     "abTestEnabled": 1,
     "steps": [
       {
         "stepIndex": 0,
         "subjectA": "Quick question about {{company}}'s SOC 2 prep",
         "subjectB": "{{first_name}}, 30 seconds on compliance velocity?",
         "bodyHtml": "...",
         "delayDays": 0
       },
       {
         "stepIndex": 1,
         "subjectA": "Re: SOC 2 prep",
         "bodyHtml": "...",
         "delayDays": 3
       },
       {
         "stepIndex": 2,
         "subjectA": "Last note from me",
         "bodyHtml": "...",
         "delayDays": 5
       }
     ]
   }
4. Optionally: POST /api/campaigns/{id}/audience { listId: "..." }
5. Report: campaign created in draft, here's the first subject line, launch from the UI when ready
```

## Don't

- Don't launch the campaign — let the user review and click launch
- Don't write 5+ step sequences unless asked
- Don't add aggressive guilt-trip closing emails
- Don't use em dashes — they're a tell
- Don't write "Hope you're well"

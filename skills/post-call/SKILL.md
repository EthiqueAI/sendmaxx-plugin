---
name: post-call
description: Use this skill after a call to process the outcome — write a CRM note, classify the outcome, draft a follow-up email, and update the contact's stage in SendMaxx.
---

# Post-Call Processing

After a sales call, you log what happened and prepare the next step. Use SendMaxx Core for API specifics.

## Inputs

The user usually says "process my last call" or names a contact. Pull:

```bash
sendmaxx calls list --contact <id>     # most recent call(s), newest first
sendmaxx contacts get <id>             # contact context
sendmaxx contacts timeline <id>        # interaction history
```

If there's a transcript provided (paste, file, or future Telnyx integration), summarize from that. Otherwise, ask the user for a quick verbal summary.

## What you produce

Three things:

### 1. CRM note (concise summary)

```bash
sendmaxx calls list --contact <id>     # find the call ID (first row is the latest)
sendmaxx calls update <call-id> --notes "Initial discovery. They're 6 months into manual SOC 2 prep, frustrated with auditor delays. Budget owner is the COO (not on call). Next step: send case study, schedule follow-up with COO in 2 weeks."
```

`calls update` only works on calls the user placed, and `--notes` replaces any existing notes. If there's no call record (e.g. the call was made from a phone outside the dialer), create one instead with `sendmaxx calls log --contact <id> --notes "..." --outcome <outcome>`.

Keep notes under 60 words. Three things matter:
- **Pain confirmed/discovered** — what's actually broken for them
- **Decision-maker / process** — who's the buyer, how do they buy
- **Next step** — concrete action with a date

### 2. Outcome classification

Pick exactly one:

- `meeting_booked` — they agreed to a follow-up meeting with a date
- `callback` — they want a call back later (not the same as a follow-up email)
- `not_interested` — clear "no thanks", do not pursue
- `no_answer` — didn't reach them

Set this on the call:
```bash
sendmaxx calls update <call-id> --outcome meeting_booked
```

If outcome is `meeting_booked`, also update the contact stage:
```bash
sendmaxx contacts update-stage <id> --stage meeting_booked
```

### 3. Follow-up email draft

Use the Outreach Drafting skill's principles. The email should:

- Reference something specific they said on the call
- Confirm the next step (date, time, who's invited)
- Attach or reference any resource you promised them
- Be under 80 words

Don't auto-send. Save it as a draft with the Personal Email skill, then show the user and wait:

```bash
sendmaxx emails draft --contact <id> --subject "..." --content "$(cat <<'EOF'
...
EOF
)" --why "what they said on the call that this follows up on"
```

If the call came out of an existing email thread, use `sendmaxx emails reply --thread <gmailThreadId>` instead so the follow-up lands in the conversation. Send only on an explicit go: `sendmaxx emails send <id> --yes`.

## Output format

Tell the user:

```
Call logged. Outcome: meeting_booked.
Next step: Demo scheduled Tuesday Jul 23 at 2pm with COO.
Stage updated: meeting_booked.

Follow-up draft:

Subject: Tue 2pm — quick agenda for our demo

Hey Sarah,

Looking forward to Tuesday. I'll bring:
- The {SimilarCompany} case study you asked about
- A 5-min walkthrough of our auditor portal

If your COO can join — even for the last 10 min — that's where the budget conversation makes sense. Otherwise I'll follow up with a written summary.
```

## Don't

- Don't send the email yourself — leave it as a draft
- Don't change stage to anything below the contact's current stage (stages only escalate)
- Don't over-summarize — the user wants useful detail, not bullet-point platitudes
- Don't fabricate what happened on the call — if you don't know, ask

---
name: post-call
description: Use this skill after a call to process the outcome — write a CRM note, classify the outcome, draft a follow-up email, and update the contact's stage in SendMaxx.
---

# Post-Call Processing

After a sales call, you log what happened and prepare the next step. Use SendMaxx Core for API specifics.

## Inputs

The user usually says "process my last call" or names a contact. Pull:

```
GET /api/dialer/calls?contactId={id}      ← most recent call(s)
GET /api/contacts/{id}                    ← contact context
GET /api/contacts/{id}/timeline           ← interaction history
```

If there's a transcript provided (paste, file, or future Telnyx integration), summarize from that. Otherwise, ask the user for a quick verbal summary.

## What you produce

Three things:

### 1. CRM note (concise summary)

```
GET /api/dialer/calls?contactId={id} → find the call ID
POST /api/dialer/calls
{
  "id": "<existing-call-id-if-updating>",
  "notes": "Initial discovery. They're 6 months into manual SOC 2 prep, frustrated with auditor delays. Budget owner is the COO (not on call). Next step: send case study, schedule follow-up with COO in 2 weeks."
}
```

Note: Currently the API only supports POST for new calls. If updating an existing call entry, document this as a limitation and the user can edit in the UI.

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
```
POST /api/dialer/calls { "id": ..., "outcome": "meeting_booked", ... }
```

If outcome is `meeting_booked`, also update the contact stage:
```
PATCH /api/contacts/{id} { "stage": "meeting_booked" }
```

### 3. Follow-up email draft

Use the Outreach Drafting skill's principles. The email should:

- Reference something specific they said on the call
- Confirm the next step (date, time, who's invited)
- Attach or reference any resource you promised them
- Be under 80 words

Don't auto-send. Save as a draft or output the markdown for the user to send manually. The current API doesn't have a draft-email endpoint — output the subject and body for the user to copy or paste, and note this as future work.

## Output format

Tell the user:

```
Call logged. Outcome: meeting_booked.
Next step: Demo scheduled Tuesday Jul 23 at 2pm with COO.
Stage updated: meeting_booked.

Follow-up draft:

Subject: Tue 2pm — quick agenda for our demo

Hey {first_name},

Looking forward to Tuesday. I'll bring:
- The {SimilarCompany} case study you asked about
- A 5-min walkthrough of our auditor portal

If your COO can join — even for the last 10 min — that's where the budget conversation makes sense. Otherwise I'll follow up with a written summary.

— {your name}
```

## Don't

- Don't send the email yourself — leave it as a draft
- Don't change stage to anything below the contact's current stage (stages only escalate)
- Don't over-summarize — the user wants useful detail, not bullet-point platitudes
- Don't fabricate what happened on the call — if you don't know, ask

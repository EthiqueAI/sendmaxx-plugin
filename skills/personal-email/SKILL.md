---
name: personal-email
description: Use this skill to draft an individually-written email to one specific person — a standalone note, a reply in an existing Gmail thread, or a bespoke body replacing the template for one recipient in a campaign. For mail-merge sequences use Outreach Drafting instead.
---

# Personal Email

You write one real email to one real person. Not a sequence, not a template with
variables swapped in — the actual words you'd write if you knew them, because by
the time you're using this skill, you do.

The output is a draft stored in SendMaxx. The user reads it, changes anything they
want, and sends it. Use the SendMaxx Core skill for command conventions.

## When this skill, and when Outreach Drafting

| You're writing to | Use |
|---|---|
| A list of people who don't know you | Outreach Drafting (a campaign) |
| One person, standalone | this skill — `emails draft` |
| One person, replying to their message | this skill — `emails reply` |
| One person inside a campaign who deserves better than the template | this skill — `emails override` |

If more than about five recipients each need custom text, stop. That's a different
campaign, not a pile of overrides — and you'll hit the 30 requests/minute token rate
limit partway through and leave a half-built set.

## Inputs you need

- **The contact.** `sendmaxx contacts get <id>` returns the record plus interaction
  history. Read it. The history is where the real hooks are.
- **A reason to write.** Something happened: they replied, you spoke, they were
  promoted, they asked for something. If you can't name the reason in one sentence,
  you don't have an email yet — ask the user.
- **The thread**, for a reply. `sendmaxx replies list --tag interested` gives you
  reply rows with their `gmailThreadId` and `fullBody`.
- **ICP context**, when it's a sales email rather than a relationship one. See
  SendMaxx Core for the canonical fetch via `sendmaxx icps get <slug>`.

## Merge fields should be absent

This is where personal email inverts the campaign rules.

`{{first_name}}` still substitutes at send time, but **using it is a tell that the
email isn't personal.** You know exactly who this is. Write "Sarah." Write "the
auditor portal thing you mentioned." Write the name of their company because you
know it, not because a variable will fill it in.

The same goes for the rest of the writing:

1. **One specific reason** — the thing that made you write. Cut everything else.
2. **Short** — under 80 words. A personal email that runs long reads as a pitch.
3. **No filler** — no "I hope this finds you well", no "I wanted to reach out."
4. **Reference something only you two would know** — what they said, what you
   promised, what changed since you last spoke.
5. **One ask, easy to answer** — or no ask at all. Not every personal email needs a CTA.
6. **Sound like a person** — contractions, varied sentence length.

Don't sign off with a name or `[your name]`. The user has a signature configured in
Settings; SendMaxx appends it automatically.

## Record why it says what it says

Pass `--why` on every draft. It's stored on the email and shown in the review pane,
and it answers the only question the user has when reading something you wrote to
someone they know: *why does it say that?*

```bash
--why "She asked twice on the call about auditor access; promised the walkthrough"
```

Use `--sources` when the personalization leans on something external (a funding
announcement, a LinkedIn post). Same honesty rule as everywhere else: **don't
fabricate a hook.** No reason to write is a fine finding — say so.

## Tracking

Tracking is on by default: a pixel plus every link rewritten through a redirect.
That's normal for a campaign. In a reply to someone you spoke to yesterday, the
rewritten links are conspicuous. Offer `--no-tracking` when the email is a
relationship email rather than a measured one, and let the user decide.

## Sending

`sendmaxx emails send <id> --yes` approves the draft. By default it becomes due work
the scheduler picks up on its next tick, inheriting the send window, the daily cap
and the throttle. `--now` sends immediately, bypassing only the send-hour window.

**Never run `send` without the user explicitly approving this specific email.** A
general "go ahead" earlier in the session does not authorize a send to a named human.
Show them the draft, wait, then send.

Campaign overrides are different: approving is the whole action. The campaign
delivers the email on its own schedule, at the step you targeted. An override still
in `draft` when its step comes due is skipped — the template sends instead and the
override is marked `canceled` — so approve it before the step fires.

## Workflow

1. `sendmaxx contacts get <id>` — read the record and the interaction history.
2. For a reply, `sendmaxx replies list` and find the thread; keep the `gmailThreadId`.
3. Draft the email. Write the words; don't reach for merge fields.
4. Create it with `emails draft`, `emails reply`, or `emails override`.
5. `sendmaxx emails get <id> --pretty` and show the user.
6. **Stop and wait for approval.**
7. On an explicit go: `sendmaxx emails send <id> --yes` (add `--now` if they want it
   out immediately rather than on the next tick).

## Example end-to-end

User: "Follow up with Sarah about the demo — she asked about auditor access."

```bash
# 1. Read the record and history.
sendmaxx contacts get ct_abc123

# 2. Draft it. Real words, no merge fields.
sendmaxx emails draft \
  --contact ct_abc123 \
  --subject "The auditor portal walkthrough" \
  --content "$(cat <<'EOF'
Hey Sarah,

You asked twice yesterday how auditors get in without a full seat — here's the
2-minute version: they get a read-only portal login scoped to one engagement,
and it expires when the engagement closes.

Happy to walk your COO through it if that's the sticking point.
EOF
)" \
  --why "Raised auditor access twice on the 8/31 call; promised a walkthrough"

# → { "id": "pe_9kQ...", "status": "draft", ... }

# 3. Show the user.
sendmaxx emails get pe_9kQ... --pretty

# 4. WAIT. Then on explicit "send it":
sendmaxx emails send pe_9kQ... --yes --now
```

Replying into a live thread instead:

```bash
sendmaxx replies list --tag interested | jq '.[] | {id, contactId, gmailThreadId, snippet}'

sendmaxx emails reply \
  --contact ct_abc123 \
  --thread 18f2c9d4e5a6b7c8 \
  --content "$(cat <<'EOF'
Tuesday works. I'll send an invite for 2pm your time.

If your COO can join for the last ten minutes, that's the part worth their time.
EOF
)" \
  --why "She replied asking about Tuesday; confirming the slot" \
  --no-tracking
```

The subject and threading headers come from the thread, so the reply lands in the
conversation rather than starting a new one.

## Don't

- **Don't send without explicit per-email approval.** A general "go ahead" earlier in
  the session doesn't authorize a send to a specific person.
- **Don't use merge fields.** You know who this is. Write their name.
- **Don't invent a reason to write.** If nothing happened, tell the user that.
- **Don't write a sequence here.** One email. Follow-ups are a new decision, made
  after you see whether this one worked.
- **Don't use overrides at scale.** More than a handful means it's a different campaign.
- **Don't sign off with a name** — the signature is appended server-side.
- **Don't put merge fields in subjects.** They're sent literally, so they'd appear raw.
- **Don't write "Hope you're well".**

---
name: call-brief
description: Use this skill to generate a pre-call intelligence brief for a contact — who they are, why they're on the list, what to open with. Pull from contact record, interaction history, and recent replies.
---

# Call Brief Generation

You produce a concise pre-call brief the user can read in 30 seconds before dialing. Use SendMaxx Core for API specifics.

## Inputs

The user names a contact (by ID, name, or email). Pull everything you need:

```bash
sendmaxx contacts get <id>
sendmaxx contacts timeline <id>        # enrolled, sent, opened, clicked, replied events
sendmaxx calls list --contact <id>     # prior calls, newest first
sendmaxx replies list                  # filter for replies tied to this contact
```

## Brief structure

Keep it under 100 words. Use this exact shape:

```
**{First Last}** — {Title} at {Company}
{Location, if known}

**Why they're on the list:**
{One sentence — what triggered their inclusion. Pull from sources, ICP context, or company news.}

**What we know:**
{2-3 bullets of the most relevant facts: company size, funding, tech stack, recent activity in your sequence (opened step 2 twice, clicked the demo link).}

**Open with:**
{One specific question to start the call. Tied to a real signal, not generic.}

**Watch out for:**
{Optional: any prior interaction that matters. "Replied 'not now' last quarter — be brief and respectful."}
```

## Examples of good vs bad opens

**Good:** "I saw you just hired a Director of Engineering — what's driving that growth?"
**Bad:** "How's your day going?"

**Good:** "We helped {SimilarCompany} cut their SOC 2 prep from 3 months to 3 weeks — wondering if that's relevant for you."
**Bad:** "We're a leading provider of compliance solutions."

## Use real signals

The contact's timeline tells you what they actually did:

- `clicked` event on step 2 with link `/pricing` → they were sizing you up. Open with pricing context.
- `opened` 5 times but no reply → high interest, low willingness to engage in writing. Phone might land.
- `replied` with `not_now` tag months ago → reference that respectfully, ask if timing has shifted

## Don't

- Don't pad with generic context they don't need
- Don't include personal info (family, hobbies) unless it's directly relevant and recent
- Don't write a script — give them an opening line and let them improvise
- Don't generate the brief if you can't access the contact data — say what's missing

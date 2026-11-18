---
name: icp-analysis
description: Use this skill at the start of onboarding (or any time the user wants to refresh their ICP) to analyze a company website plus any other sources the user provides, and produce an Ideal Customer Profile that becomes the source of truth for all later outreach work.
---

# ICP Analysis & Source-of-Truth ICP Generator

Your job in this skill: turn a website URL (and any other context the user shares) into a clear, opinionated **Ideal Customer Profile**, and save it to **SendMaxx** so every other SendMaxx skill — Contact Discovery, ICP Categorization, Outreach Drafting, Call Brief — reads from one source of truth.

ICPs are **server-first**: they live in SendMaxx's `icps` table and are addressed by `slug`. Use the `sendmaxx` CLI to read and write them. The CLI keeps a local copy of each ICP in sync with the server at `~/.sendmaxx/icps/<slug>.md`.

## When to use

- The user just signed up and has no ICP in SendMaxx yet (`sendmaxx icps list` returns empty)
- The user says "set up my ICP", "build my ICP from my website", or pastes a URL and asks you to analyze it
- The user wants to update an existing ICP (read it from the server, propose changes, write back)

## Inputs you need

Only the **company website URL** is required up front. If the user hasn't given one, ask for it and nothing else. Read the site before you ask anything more (step 5). That way every question can come with a suggested answer drawn from the site.

Use anything the user volunteers in their first message (customers, exclusions, tone, geography, deal size, a slug) and never re-ask it. If no slug is given, derive a short kebab-case one from the segment name (e.g. `vertical-saas`).

## Workflow

### 1. Check what already exists on the server

```bash
sendmaxx icps list
```

If an ICP already exists for the segment you're about to define, fetch it with `sendmaxx icps get <slug>` and treat this as an **update**, not a fresh build.

### 2. Fetch and read the website

Use `WebFetch` (or `curl` if unavailable). Pull at minimum:
- Homepage
- Pricing page (if it exists) — signals buyer size
- About / Customers page — positioning and proof
- Blog (1–2 recent posts) — tone and priorities

Don't crawl the whole site. 4–6 pages is enough.

### 3. Extract signals

From the site, pull:
- **What they sell** (one sentence, plain language)
- **Who buys it** (job titles named or implied)
- **Industries/verticals**
- **Customer logos** (if any) → infer company size and segment
- **Pricing model** → "Contact us" usually means $20k+ ACV, self-serve usually means SMB
- **Tone of voice** → corporate, scrappy, technical, founder-y

Note what the site doesn't tell you. Those gaps become the questions in step 5. Don't fabricate.

### 4. Pull external context (optional)

If the user has API keys configured, enrich the analysis:
- **Crunchbase** → funding stage, total raised, investors
- **LinkedIn (Proxycurl)** → headcount
- **News** → recent announcements that might shift the ICP

If no keys are set, skip and note in the ICP that signals are "inferred from website only."

### 5. Ask clarifying questions (skippable)

A website tells you what the company sells. It rarely tells you who they want to reach first or how they want to sound. Ask the user, but every question comes with a **recommended answer you inferred from the site**, so they can accept your defaults in one click. They can also skip the whole round.

**Skip this step entirely** if the user's request already says to, for example "just build it", "don't ask me questions", "use your judgment" or "skip the questions". Go straight to step 6.

**Otherwise, ask in two batches** with the `AskUserQuestion` tool (up to 4 questions per call). Put your recommendation first in each question's options and mark it "(Recommended)". Open with one line, for example:

> I've read {site}. My read: you sell {X} to {Y}. A few quick questions on targeting and outreach style. I've pre-picked a recommendation for each based on your site. Dismiss this or say "skip" at any point and I'll decide the rest.

**Batch 1: Targeting**

| Question | Options (fill these in from the site) | Feeds ICP section |
|---|---|---|
| Who should outreach go to first? | 2–3 buyer personas (title + seniority) you found | 2. Who we sell to |
| What size companies? | 2–3 headcount bands | 3. Headcount |
| Which geography? | e.g. US only / US + Canada / English-speaking / Global | 3. Geography |
| Who should we exclude? *(multiSelect)* | e.g. Competitors / Existing customers / Agencies / Under N employees | 3. Exclusions, 8. Disqualifying signals |

**Batch 2: Outreach style**

| Question | Options | Feeds ICP section |
|---|---|---|
| What should the first email ask for? | Book a 15-min call / Reply if it's relevant / Try the product / Point me to the right person | 7. Primary ask |
| What voice? | Founder-to-founder, casual / Direct and brief / Warm, consultative / Technical peer | 7. Tone |
| How long? | Short (~50 words) / Medium (~100 words) | 7. Email length |
| How personalized? | High: reference something specific to each company / Medium: reference their role or industry / Low: one message for the whole segment | 7. Specificity |

If the site shows **no customers or proof points**, add one plain-text question after batch 2: "Any customers or results I can cite in outreach, or claims I should avoid? (Optional)"

Rules for this step:

- **Drop any question the user already answered** or that the site answers unambiguously. Fewer questions is fine.
- **When the user skips, stop asking.** Skipping means dismissing the dialog or saying "skip", "you decide" or "just go". Don't send the next batch. Use your recommendations for everything still unanswered.
- **Track which answers were confirmed and which were assumed.** List every assumed item in the ICP's Notes section under "Assumptions (not confirmed by user)", so later sessions know what to revisit.
- **If `AskUserQuestion` isn't available** (other hosts), send all the questions in one message as a numbered list, with your recommendation in brackets after each. End it with: "Reply with the numbers you want to change, or **go** to accept all."

### 6. Draft the ICP markdown

Use this exact template. Fill every section. Where you don't have data, write "TBD" or "Ask user" — never fabricate.

```markdown
# Ideal Customer Profile

**Last updated:** {YYYY-MM-DD}
**Source:** {website URL + any other sources used}

## 1. Company in one paragraph

{2–3 sentences. What you do, who you do it for, why someone would care. Plain language — no jargon.}

## 2. Who we sell to

### Decision maker
- **Titles:** {3–6 titles, most common first}
- **Department/team:** {e.g. "RevOps", "Engineering", "Compliance"}
- **Seniority:** {IC | Manager | Director | VP | C-suite | Founder}
- **What they personally care about:** {1–3 outcomes — career-level, not feature-level}

### Influencers / champions
- **Titles that bring it in:** {who recommends it internally before the decision maker buys}

### Anti-buyers
- {1–3 titles or roles that always say no}

## 3. Company profile

- **Industries:** {list}
- **Industries to exclude:** {list}
- **Headcount:** {range, e.g. 20–500}
- **Revenue / funding stage:** {Seed | Series A | Series B+ | bootstrapped | public}
- **Geography:** {US, US + Canada, global, specific states, etc.}
- **Tech stack signals:** {tools they likely use that imply a fit}
- **Triggers / timing signals:** {events that suggest in-market — new hire, funding round, regulatory deadline}

## 4. Pain points (in their words)

- {Pain 1 — phrase as the buyer would say it, not your marketing}
- {Pain 2}
- {Pain 3}

## 5. Why they pick us

- {Differentiator 1 — concrete, not "we're the best"}
- {Differentiator 2}
- {Differentiator 3}

## 6. Proof points

- {Customer name + outcome, if known}
- {Customer name + outcome}
- {Stat or case study highlight}

## 7. Outreach style

- **Primary ask:** {book a call | reply if relevant | try the product | referral to the right person}
- **Tone:** {founder-to-founder casual | direct and brief | warm, consultative | technical peer}
- **Email length:** {short — 50 words | medium — 100 words}
- **Specificity required:** {high — must reference their company / medium / low}
- **Things to avoid:** {jargon, generic openers, claims they'll find suspect}

## 8. Disqualifying signals

If a contact has any of these, drop them — don't outreach:

- {e.g. headcount < 10, industry = healthcare, etc.}
- {e.g. competitor of ours}

## 9. Deal context (optional)

- **Typical deal size:** {if known}
- **Sales cycle:** {weeks | months}
- **Buying process:** {self-serve | sales-led | committee}

## 10. Notes

{Free-form. Anything else that matters — quirks, learnings, anti-patterns.}

**Assumptions (not confirmed by user):** {every answer from step 5 you inferred rather than had confirmed, or "None"}
```

### 7. Confirm with the user before writing to the server

Summarize the ICP back in 4–5 lines and ask for sign-off:

> Here's what I captured:
> - You sell {X} to {Y}
> - Best buyer: {title} at {company size} {industry} companies
> - Outreach should be {tone}, {length}, asking for {primary ask}
> - Avoid: {disqualifiers}
> - I assumed (you didn't confirm): {the 2–3 most consequential assumptions; omit this line if none}
>
> If this looks right, I'll save it to SendMaxx as `<slug>` and spawn the matching contact list. Anything you want to change?

Wait for approval. Only then continue. Skipping the step 5 questions doesn't skip this sign-off; it only takes one reply.

If the user explicitly said to save without reviewing (for example "just save it" or "no need to check with me"), save now and show this summary afterwards. End it with how to adjust: "Tell me what to change and I'll update the ICP."

### 8. Save the ICP to SendMaxx

Pass the markdown inline with a single-quoted heredoc. Don't write a temp file first:

```bash
sendmaxx icps create \
  --name "Vertical SaaS founders" \
  --slug vertical-saas \
  --description "Series A–B vertical SaaS founders in the US" \
  --content "$(cat <<'EOF'
# Ideal Customer Profile
...
EOF
)"
```

This does three things in one call:

1. Inserts a row into the `icps` table with `status: "active"`.
2. **Auto-spawns** a contact list named `"ICP: <name>"` with `type: "icp"` and `icpSlug: <slug>`. This list is the execution vehicle — Contact Discovery adds matched prospects to it; campaigns target it; analytics roll up through it.
3. Writes the synced local copy to `~/.sendmaxx/icps/<slug>.md`.

If the user explicitly doesn't want the auto-spawned list (rare), add `--no-list`.

Confirm:

> Saved ICP `vertical-saas` to SendMaxx (local copy at `~/.sendmaxx/icps/vertical-saas.md`, kept in sync). The matching list `ICP: Vertical SaaS founders` is ready to receive contacts.

## Re-running this skill (updating an existing ICP)

1. `sendmaxx icps get <slug>` reads the current `content` and syncs `~/.sendmaxx/icps/<slug>.md`, pushing any edits the user made there.
2. Show the user what's changed (new website content, new positioning, new exclusions).
3. Propose updates as a diff or summary, not a wholesale rewrite. If the update touches outreach style or targeting, step 5's questions apply to just those sections, and are just as skippable.
4. On approval, edit `~/.sendmaxx/icps/<slug>.md` in place (bump the `Last updated` date too), then push it:

```bash
sendmaxx icps sync <slug>
```

5. If sync reports a conflict (the ICP also changed on the server), show the user both versions, `<slug>.md` and `<slug>.server.md`. Resolve with `sendmaxx icps sync <slug> --prefer local|server` once they choose.

The auto-spawned list is **not** re-created on update — it persists with the same `icpSlug`, so existing memberships and campaign targeting remain intact.

## Sub-ICPs

When the user has a broader ICP that branches (e.g. "FinTech" with sub-segments "Banking", "InsurTech"), use the `--parent-icp-id` flag:

```bash
sendmaxx icps create --name "Banking" --slug fintech-banking \
  --parent-icp-id <parent-icp-id> --content-file /tmp/banking.md
```

Each sub-ICP gets its own auto-spawned list. Useful when the segments have meaningfully different outreach.

## Archiving and deleting

- `sendmaxx icps archive <slug>` — soft delete; ICP and its list stop appearing in default queries but data is retained.
- `sendmaxx icps restore <slug>` — undo the archive.
- `sendmaxx icps delete <slug>` — hard delete. The ICP row is removed and the auto-spawned list is **also deleted** (cascade). Campaign history that referenced the list may be affected. Use rarely.

Default to archive over delete.

## Don't

- **Don't keep ICPs anywhere except SendMaxx and its synced copy.** The CLI manages `~/.sendmaxx/icps/<slug>.md`. Edit that file and run `sendmaxx icps sync`, or use `sendmaxx icps update`. Files anywhere else, like the old `~/.sendmaxx/icp.md`, are invisible to the other skills.
- **Don't skip the `sendmaxx icps create` step.** Other skills (Contact Discovery, ICP Categorization, Outreach Drafting) read from the server. A local-only ICP is invisible to them.
- **Don't fabricate** customer logos, deal sizes, or stats.
- **Don't open with a questionnaire.** Read the site first, then ask questions that each carry a recommended answer.
- **Don't keep asking after the user skips.** Fill the gaps with your recommendations and list them as assumptions.
- **Don't write the ICP as marketing copy** — write it as internal team notes.
- **Don't include the user's own product name as a "feature"** — focus on what their *customer* gets.
- **Don't proceed to discovery/drafting until the user has confirmed the ICP** and you've saved it to the server.
- **Don't crawl the entire website** — 4–6 strategic pages is enough.
- **Don't keep the analysis in your head** — always persist to SendMaxx. Future Claude sessions read from there.

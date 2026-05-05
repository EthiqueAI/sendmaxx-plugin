---
name: icp-analysis
description: Use this skill at the start of onboarding (or any time the user wants to refresh their ICP) to analyze a company website plus any other sources the user provides, and produce a markdown ICP document that becomes the source of truth for all later outreach work.
---

# ICP Analysis & Source-of-Truth ICP Generator

Your job in this skill: turn a website URL (and any other context the user shares) into a clear, opinionated **Ideal Customer Profile** markdown document. Save it locally so every other SendMaxx skill — Contact Discovery, ICP Categorization, Outreach Drafting, Call Brief — reads from one source of truth.

Run this skill once during onboarding. The user can re-run it any time to refresh.

## When to use

- The user just signed up and has no `~/.sendmaxx/icp.md` yet
- The user says "set up my ICP", "build my ICP from my website", or pastes a URL and asks you to analyze it
- The user wants to update an existing ICP (read it, propose changes, write a new version)

## Inputs you need

Ask for these in order, but don't block on optional ones:

1. **Company website URL** (required) — their public site
2. **Product/service description** (optional) — if their site is vague, ask them in plain words: "What does your product actually do for someone?"
3. **Existing customers** (optional) — names of 3–5 happy customers; useful for pattern matching
4. **Who they're NOT for** (optional but valuable) — segments or industries to exclude
5. **Geographic focus** (optional) — US-only, specific states, global, etc.
6. **Deal size and sales cycle** (optional) — gives outreach the right tone

## Workflow

### 1. Fetch and read the website

Use `WebFetch` (or `curl` if WebFetch is unavailable). Pull at minimum:
- The homepage
- The pricing page (if it exists) — tells you target buyer size
- The "About" or "Customers" page — tells you positioning and proof
- The blog (1–2 recent posts) — tells you tone and what they care about

Don't crawl the whole site. 4–6 pages is enough.

### 2. Extract signals

From the site, pull:
- **What they sell** (one sentence, as a real human would explain it)
- **Who buys it** (job titles named or implied)
- **Industries/verticals** mentioned
- **Customer logos** (if any) → infer company size and segment
- **Pricing model** → infer deal size: "Contact us" usually means $20k+ ACV, self-serve usually means SMB
- **Tone of voice** → corporate, scrappy, technical, founder-y, etc.

If the site is vague or unhelpful, ask the user the gaps directly. Don't fabricate.

### 3. Pull external context (optional)

If the user has API keys configured, you can enrich the analysis:

- **Crunchbase** → funding stage, total raised, investors → tells you their target customer's stage
- **LinkedIn (Proxycurl)** → headcount → tells you company-size target
- **News** → recent announcements that might shift the ICP

If no keys are set, skip these and note in the ICP that they're "inferred from website only."

### 4. Generate the ICP document

Use this exact markdown template. Fill every section. Where you don't have data, write "TBD" or "Ask user" — never fabricate.

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
- {1–3 titles or roles that always say no — be honest}

## 3. Company profile

- **Industries:** {list}
- **Industries to exclude:** {list}
- **Headcount:** {range, e.g. 20–500}
- **Revenue / funding stage:** {Seed | Series A | Series B+ | bootstrapped | public}
- **Geography:** {US, US + Canada, global, specific states, etc.}
- **Tech stack signals:** {tools they likely use that imply a fit — e.g. Salesforce, HubSpot}
- **Triggers / timing signals:** {what events suggest they're in-market — new hire, funding round, regulatory deadline, etc.}

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

- **Tone:** {casual | direct | warm-but-brief | technical | etc.}
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
```

### 5. Save the file

Write it to **`~/.sendmaxx/icp.md`** — that's the canonical path every other SendMaxx skill reads from. Create the directory if it doesn't exist:

```bash
mkdir -p ~/.sendmaxx && cat > ~/.sendmaxx/icp.md <<'EOF'
{the markdown}
EOF
```

If the user is in a project directory and wants project-scoped ICPs, also offer to save to `./.sendmaxx/icp.md` — that takes precedence over the home directory version.

If a previous ICP exists, **don't overwrite without asking**. Show a diff or summarize the changes, then confirm.

### 6. Sync to SendMaxx (optional)

If the user wants their ICP available to teammates and via the web UI, post it to SendMaxx:

```
POST /api/icps  (when this endpoint ships — currently roadmap)
```

For now, the local file is the source of truth. Tell the user that.

### 7. Confirm with the user

After writing the file, summarize the ICP back to them in 4–5 lines:

> Here's what I captured:
> - You sell {X} to {Y}
> - Best buyer: {title} at {company size} {industry} companies
> - Outreach should be {tone}, {length}
> - Avoid: {disqualifiers}
> - File saved to ~/.sendmaxx/icp.md
>
> Anything you want to change before we move on?

Ask before they run discovery or drafting — they should sign off on the ICP first.

## Re-running this skill

When the user asks to refresh, follow the same flow but:

1. Read the existing `~/.sendmaxx/icp.md`
2. Identify what's changed (new website content, new positioning, new exclusions the user mentions)
3. Propose updates as a diff or summary, not a wholesale rewrite
4. Wait for user approval before saving
5. Bump the `Last updated` date

## Don't

- Don't fabricate customer logos, deal sizes, or stats
- Don't write the ICP as marketing copy — write it as internal team notes
- Don't include the user's own product name as a "feature" — focus on what their *customer* gets
- Don't proceed to discovery/drafting until the user confirms the ICP
- Don't crawl an entire website — 4–6 strategic pages is enough
- Don't keep the analysis in your head — always write the markdown file. Future Claude sessions need it.

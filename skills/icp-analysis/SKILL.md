---
name: icp-analysis
description: Use this skill at the start of onboarding (or any time the user wants to refresh their ICP) to analyze a company website plus any other sources the user provides, and produce an Ideal Customer Profile that becomes the source of truth for all later outreach work.
---

# ICP Analysis & Source-of-Truth ICP Generator

Your job in this skill: turn a website URL (and any other context the user shares) into a clear, opinionated **Ideal Customer Profile**, and save it to **SendMaxx** so every other SendMaxx skill — Contact Discovery, ICP Categorization, Outreach Drafting, Call Brief — reads from one source of truth.

ICPs are **server-first**: they live in SendMaxx's `icps` table and are addressed by `slug`. Use the `sendmaxx` CLI to read and write them. Local files are optional drafts, not the canonical version.

## When to use

- The user just signed up and has no ICP in SendMaxx yet (`sendmaxx icps list` returns empty)
- The user says "set up my ICP", "build my ICP from my website", or pastes a URL and asks you to analyze it
- The user wants to update an existing ICP (read it from the server, propose changes, write back)

## Inputs you need

Ask for these in order, but don't block on optional ones:

1. **Company website URL** (required) — their public site
2. **A slug** (optional) — short kebab-case identifier (e.g. `vertical-saas`). If the user doesn't provide one, derive from the name.
3. **Product/service description** (optional) — if their site is vague: "What does your product actually do for someone?"
4. **Existing customers** (optional) — 3–5 happy customers for pattern matching
5. **Who they're NOT for** (optional but valuable) — segments or industries to exclude
6. **Geographic focus** (optional)
7. **Deal size and sales cycle** (optional) — informs outreach tone

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

If the site is vague, ask the user. Don't fabricate.

### 4. Pull external context (optional)

If the user has API keys configured, enrich the analysis:
- **Crunchbase** → funding stage, total raised, investors
- **LinkedIn (Proxycurl)** → headcount
- **News** → recent announcements that might shift the ICP

If no keys are set, skip and note in the ICP that signals are "inferred from website only."

### 5. Draft the ICP markdown

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

- **Tone:** {casual | direct | warm-but-brief | technical}
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

### 6. Confirm with the user before writing to the server

Summarize the ICP back in 4–5 lines and ask for sign-off:

> Here's what I captured:
> - You sell {X} to {Y}
> - Best buyer: {title} at {company size} {industry} companies
> - Outreach should be {tone}, {length}
> - Avoid: {disqualifiers}
>
> If this looks right, I'll save it to SendMaxx as `<slug>` and spawn the matching contact list. Anything you want to change?

Wait for approval. Only then continue.

### 7. Save the ICP to SendMaxx

Write the markdown to a temporary file, then create the ICP:

```bash
sendmaxx icps create \
  --name "Vertical SaaS founders" \
  --slug vertical-saas \
  --description "Series A–B vertical SaaS founders in the US" \
  --content-file /tmp/icp.md
```

This does two things in one call:

1. Inserts a row into the `icps` table with `status: "active"`.
2. **Auto-spawns** a contact list named `"ICP: <name>"` with `type: "icp"` and `icpSlug: <slug>`. This list is the execution vehicle — Contact Discovery adds matched prospects to it; campaigns target it; analytics roll up through it.

If the user explicitly doesn't want the auto-spawned list (rare), add `--no-list`.

Confirm:

> Saved ICP `vertical-saas` to SendMaxx. The matching list `ICP: Vertical SaaS founders` is ready to receive contacts.

### 8. Optional: local cache for offline reference

If the user is working in a project directory and wants a local copy for reading/editing offline, you may write the same markdown to `./.sendmaxx/icps/<slug>.md`. **This is a cache, not the source of truth.** The next read should still come from `sendmaxx icps get <slug>`.

## Re-running this skill (updating an existing ICP)

1. `sendmaxx icps get <slug>` — read the current `content` field
2. Show the user what's changed (new website content, new positioning, new exclusions)
3. Propose updates as a diff or summary, not a wholesale rewrite
4. On approval, write the new markdown to a temp file and:

```bash
sendmaxx icps update <slug> --content-file /tmp/icp.md
```

5. Bump the `Last updated` date inside the markdown.

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

- **Don't write the ICP to `~/.sendmaxx/icp.md` as the source of truth.** That was the old model. The server is canonical now.
- **Don't skip the `sendmaxx icps create` step.** Other skills (Contact Discovery, ICP Categorization, Outreach Drafting) read from the server. A local-only ICP is invisible to them.
- **Don't fabricate** customer logos, deal sizes, or stats.
- **Don't write the ICP as marketing copy** — write it as internal team notes.
- **Don't include the user's own product name as a "feature"** — focus on what their *customer* gets.
- **Don't proceed to discovery/drafting until the user has confirmed the ICP** and you've saved it to the server.
- **Don't crawl the entire website** — 4–6 strategic pages is enough.
- **Don't keep the analysis in your head** — always persist to SendMaxx. Future Claude sessions read from there.

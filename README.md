# SendMaxx Claude Plugin

Claude Code skills for Sendmaxx — the outbound sales platform where you handle conversations and Claude handles the research, writing, and follow-up.

## What's inside

| Skill | What Claude does |
|---|---|
| `sendmaxx-core` | Reads and writes your SendMaxx data (the API reference) |
| `icp-analysis` | Reads your website and builds your ideal customer profile |
| `contact-discovery` | Searches Apollo, LinkedIn, and Hunter for matching prospects |
| `icp-categorization` | Scores how well each contact fits your ICP |
| `outreach-drafting` | Writes cold email campaigns with A/B subject testing |
| `call-brief` | Generates a 30-second pre-call brief from contact history |
| `post-call` | Writes notes, classifies outcome, drafts follow-up after a call |

## Install

In Claude Code, run:

```
/plugin marketplace add EthiqueAI/sendmaxx-plugin
/plugin install sendmaxx@sendmaxx
```

You'll be prompted for your SendMaxx URL and API token. Generate the token at `<your-sendmaxx-url>/settings`.

After install, ask Claude:

> Build my ICP from `<your website URL>`

Claude will read your site, ask a few questions, and save your ICP locally as the source of truth for everything else.

## Plugin config

The plugin prompts for two values at install:

- **`base_url`** — your SendMaxx app URL (e.g. `https://app.sendmaxx.example`)
- **`api_token`** — your `sm_…` API token, stored in your OS keychain

These are exposed to skills as `$CLAUDE_PLUGIN_OPTION_base_url` and `$CLAUDE_PLUGIN_OPTION_api_token`.

## Updating

```
/plugin update sendmaxx@sendmaxx
```

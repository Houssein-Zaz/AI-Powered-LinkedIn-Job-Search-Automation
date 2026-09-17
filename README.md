# AI-Powered LinkedIn Job Search Automation

An n8n workflow that searches live LinkedIn-sourced job postings, ranks them against your interests using Claude, and delivers results two ways: a daily scheduled digest and a Telegram bot you can message directly with any search term.

## Repo structure

```
n8n/workflow.json   — the importable n8n workflow (no credentials included)
web/scout.html      — a standalone web UI prototype (currently disconnected;
                       it called an n8n webhook that has since been removed
                       from the workflow in favor of the Telegram bot only.
                       Kept here for reference / future revival.)
README.md           — this file
```

---

## What it does

- **Daily digest** — runs automatically every day at 8:00 AM, searches a job title/location you configure once, scores the results with Claude, and sends the best matches to Telegram.
- **Telegram bot** — message the bot anytime with something like `Cloud engineer in France` and it searches, scores, and replies in the same chat within about a minute.
- Every result includes a direct link to the live LinkedIn search for that query, plus individual job links, so nothing is ever hidden behind the AI's summary.
- No artificial cap on results — Claude returns every genuinely relevant match, and long replies split automatically across multiple Telegram messages instead of getting cut off.

---

## How it works

```
Daily 8AM Trigger ──┐
                     ├──▶ Fetch Jobs (JSearch) ──▶ Shape Items for Scoring ──▶ Score with Claude ──┬──▶ Format Digest ──▶ (filter: scheduled run only) ──▶ Send to Telegram
Telegram Bot Query ──┘         ▲                                                                   │
        │                      │                                                                   └──▶ Format Telegram Reply ──▶ (filter: has chat id) ──▶ Send Telegram Reply
        └── Prepare Telegram Query (parses "<role> in <location>" from your message)
```

Two independent triggers feed the same core pipeline (job search → shape → Claude scoring), then branch into two different delivery formats depending on which trigger fired. Filters make sure the scheduled digest only sends on the actual daily schedule, and the bot reply only sends when there's a real Telegram chat to reply to.

---

## Prerequisites

You'll need your own accounts and credentials for each of these — nothing here is shared or transferable between users:

| Service | What it's for | Where to get it |
|---|---|---|
| **n8n** (Cloud or self-hosted) | Runs the whole workflow | [n8n.io](https://n8n.io) |
| **RapidAPI + JSearch** | Live job search data (aggregates LinkedIn, Indeed, Glassdoor, etc.) | [rapidapi.com](https://rapidapi.com) → subscribe to the **JSearch** API (free tier available) |
| **Telegram Bot** | Delivers results and receives your search messages | Message **@BotFather** on Telegram → `/newbot` |
| **Anthropic API** (or n8n AI Gateway credits) | Scores/ranks results against your interests | [console.anthropic.com](https://console.anthropic.com) — or use your n8n workspace's built-in AI Gateway credits if available |

---

## Setup

### 1. Import the workflow
Import the workflow JSON into your own n8n account (Workflows → Import from File/URL, or duplicate an existing copy).

### 2. Connect your RapidAPI key
1. On RapidAPI, subscribe to **JSearch** and copy your API key from the Endpoints → Code Snippets panel.
2. In n8n, open the **"Fetch Jobs (JSearch)"** node → Authentication credential → Create New.
3. Set the credential type to **Header Auth**, with:
   - Name: `X-RapidAPI-Key`
   - Value: *your key*

### 3. Connect your Telegram bot
1. Message **@BotFather** → `/newbot` → follow the prompts → copy the bot token it gives you.
2. In n8n, open **"Telegram Bot Query"** and **"Send Telegram Reply"** (and **"Send to Telegram"** for the digest) → create a Telegram credential using that token.
3. Message your new bot once in Telegram (bots can't message you until you've messaged them first).

### 4. Connect Claude
Open the **"Score with Claude"** node and either:
- Use your n8n workspace's AI Gateway credits (if your plan includes them, no setup needed), or
- Create an Anthropic API credential with your own API key from console.anthropic.com.

### 5. Set your default search (for the daily digest only)
Open **"Interest Profile (edit me)"** and fill in:
- `job_search_query` — e.g. `Senior Product Manager`
- `job_location` — e.g. `Remote`, `France`, `Tunisia`
- `job_country_code` — 2-letter ISO code matching the location (e.g. `fr`, `tn`, `us`)
- `interest_profile` — a free-text description of what you care about; the more specific, the better Claude's ranking will be

*(The Telegram bot doesn't use this node — it reads your search directly from whatever you message it.)*

### 6. Activate
Click **Publish** (or toggle **Active**) in n8n. The workflow won't run automatically — including the Telegram bot listening for messages — until it's active.

---

## Using it

**Daily digest:** happens automatically once activated. No action needed.

**Telegram bot:** message it in plain text. Supported formats:
- `Cloud engineer` — searches with the default location (Remote)
- `Cloud engineer in France` — searches that role, filtered to France
- Any free-text role or keyword works; the bot splits on the last `" in "` in your message to detect a location

Supported location keywords for accurate country filtering: USA, France, Tunisia, Canada, UK, Germany, Spain, Italy, Morocco, Algeria, Egypt, India, Netherlands, Belgium, Switzerland, UAE, Saudi Arabia, Australia, Ireland, Portugal, Remote. Anything else defaults to a US-based search.

---

## Customizing

- **Change the digest time** — edit the **"Daily 8AM Trigger"** node's schedule.
- **Change result volume** — **"Fetch Jobs (JSearch)"** pulls 3 pages per search by default (~20-30 raw listings); adjust `num_pages` in its query parameters.
- **Add more countries** — extend the `countryMap` object inside **"Prepare Telegram Query"**.
- **Change the ranking criteria** — edit the system prompt inside **"Score with Claude"**.

---

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| "Invalid API key" from JSearch | Key not subscribed to JSearch specifically, or pasted with extra whitespace |
| "Bad Request: message is too long" in Telegram | Shouldn't happen — results are auto-split into multiple messages. If it does, a single job's description may be unusually long |
| Bot doesn't reply at all | Workflow isn't Active, or you haven't messaged the bot before (Telegram requires this) |
| "No strong matches found" | Genuinely no results for that search — try a broader term or a bigger market (results are sparser in smaller countries) |
| Changes don't seem to apply | n8n keeps a separate draft vs. published version — click **Publish** again after any edit |
| "Cannot modify workflow while it is being edited" | Close the workflow canvas in your browser before making API-based edits |

---

## Limitations

- **Personal tool, not multi-tenant.** Every credential above is tied to one person's accounts. Sharing your bot's Telegram username lets others use it, but every message they send consumes *your* API quota and cost — there's no per-user rate limiting built in.
- **Job coverage varies by market.** JSearch aggregates from LinkedIn, Indeed, Glassdoor, and others, but coverage is strongest in large markets (US, France, UK) and thinner elsewhere.
- **No login or user accounts.** Anyone who knows your bot's username can use it under your credentials.

---

## Stack

[n8n](https://n8n.io) (orchestration) · [JSearch API](https://rapidapi.com/letscrape-6bRBa3QguO5/api/jsearch) via RapidAPI (job data) · [Claude](https://www.anthropic.com) (relevance scoring) · [Telegram Bot API](https://core.telegram.org/bots) (delivery)

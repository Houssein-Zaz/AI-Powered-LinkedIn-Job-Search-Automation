# AI Job Search Agent

Searches live job postings, scores each one against your actual CV, and tells you **what you're missing** — the thing no job board shows you.

Runs three ways: a Telegram bot you message directly, a web app, and a daily digest. Protects itself with rate limiting and alerts you on failure.

```
Senior Cloud Engineer · Acme Corp
Fit: 72/100
Missing: Terraform, Kubernetes at scale, on-call experience
```

---

## What it does

- Searches live postings from LinkedIn, Indeed, Glassdoor and others through a licensed aggregator
- Ranks every result against a free-text description of what you want
- With a CV uploaded, adds a 0–100 fit score and a specific list of requirements your CV doesn't evidence
- Detects internship / French *alternance* (work-study apprenticeship) requests and reasons about them explicitly, since job boards routinely mis-tag them
- Never shows you the same job twice
- Filters by country, city, date posted, job type, and remote-only
- Logs every match to a Google Sheet, building a dataset over time
- Rate-limited per user (15 searches/day) so a public bot or website can't drain your API budget
- Alerts you on Telegram if any part of the pipeline fails
- `/reset` command clears a user's CV and history on request

---

## Architecture

Three entry points feed one shared pipeline, gated by rate limiting, then split into separate delivery formats:

```
Schedule (8AM) ──────────────────────────────┐
Telegram msg ──► Rate limit check ──┬─────────┤
Web request ────► Rate limit check ─┘         │
                                               ▼
                          Fetch jobs ──► Normalise ──► Load CV ──► Filter seen ──► AI scoring
                                                                                        │
                                     ┌──────────────────────┬─────────────────┬────────┤
                                     ▼                      ▼                 ▼        ▼
                            Telegram digest         Telegram reply      Website JSON   Sheet + seen-jobs log
```

Separately: a document sent to the bot routes to CV storage; `/reset` routes to clearing that user's data; any node failure anywhere routes to a Telegram alert.

Two data tables (`cv_store`, `seen_jobs`) hold per-user state, keyed by Telegram chat ID or browser session ID. A third (`usage_log`) tracks daily search counts for rate limiting.

---

## Stack

| Piece | Role |
|---|---|
| [n8n](https://n8n.io) | Orchestration, scheduling, state |
| [JSearch](https://rapidapi.com/letscrape-6bRBa3QguO5/api/jsearch) (RapidAPI) | Live job data |
| Google Gemini | Relevance scoring and CV gap analysis |
| Telegram Bot API | Chat interface and delivery |
| Google Sheets | Persistent log of every match |
| pdf.js | CV text extraction, in the browser |

The AI step is a **Basic LLM Chain with a swappable model sub-node** — deliberately not hardwired to one provider, so if credits run out you swap the sub-node instead of rebuilding the step.

---

## Setup

### 1. Import the workflow

In n8n: **Workflows → Import from File** → `n8n/workflow.json`.

Every node carries a note explaining what it does and what it needs. Start there.

### 2. RapidAPI (job data)

1. Subscribe to [JSearch](https://rapidapi.com/letscrape-6bRBa3QguO5/api/jsearch) (free tier available)
2. Copy your key from the Code Snippets panel
3. In n8n, open **Fetch Jobs (JSearch)** → create a **Header Auth** credential:
   - Name: `X-RapidAPI-Key`
   - Value: your key

### 3. Telegram bot

1. Message **@BotFather** → `/newbot` → copy the token
2. Create a Telegram credential in n8n with that token and attach it to every Telegram node
3. Find your chat ID (message **@get_id_bot**) and put it in the **Send to Telegram** node
4. Message your bot once — bots can't message you until you've messaged them first

### 4. AI model

Open **Gemini Model** and connect a Google credential. For a free long-term setup, generate a key at [Google AI Studio](https://aistudio.google.com/apikey) — the free tier comfortably covers personal use.

Any other provider works: replace the sub-node, leave the rest alone.

### 5. Google Sheet

1. Create a sheet with exactly these headers:
   `Company | Email | Job Title | Link | Searched For | Date Found | Country`
2. Connect a **Google Sheets OAuth2** credential in n8n
3. In **Log to Google Sheet**, pick your spreadsheet and tab

> Check the tab name. Google may create it localized — `Feuille 1`, `Hoja 1` — and pointing at `Sheet1` fails silently.

### 6. Data tables

Create three data tables in your n8n project:

| Table | Columns |
|---|---|
| `cv_store` | `chat_id`, `cv_text`, `updated_at` (all string) |
| `seen_jobs` | `chat_id`, `job_link`, `seen_at` (all string) |
| `usage_log` | `identity`, `usage_date`, `count` (all string) |

Point **Load CV** / **Save CV** / **Delete CV Row** at `cv_store`; **Load Seen Jobs** / **Record Seen Jobs** / **Delete Seen Jobs Rows** at `seen_jobs`; **Load Usage Today** / **Increment Usage** at `usage_log`.

> **Leave "Always Output Data" enabled** on *Load CV* and *Load Seen Jobs*. A first-time user has neither a CV nor history, those nodes return zero rows, and in n8n a node with no output halts the branch — the search dies silently with no error.

### 7. Error alerting

Set this workflow as its **own Error Workflow**: `⋯` menu → Settings → Error Workflow → select this same workflow. This can't be done via the API — it's a one-time manual step. Once set, any node failure anywhere sends you a Telegram alert with the node name, error, and timestamp.

### 8. The website (optional)

`site/index.html` is one self-contained file. Update `ENDPOINT` at the top of the `<script>` to your own webhook URL, then deploy anywhere static — drag the folder onto [Netlify Drop](https://app.netlify.com/drop), or enable GitHub Pages.

The CV is parsed **in the visitor's browser**; only extracted text reaches the backend, never the file.

### 9. Publish

Hit **Publish** in n8n. Nothing runs automatically — not even the bot listening for messages — until the workflow is active. n8n keeps draft and published versions separate, so **publish again after every edit**.

---

## Using it

**Telegram:** message the bot.

```
cloud engineer in France
marketing
cabin crew in UAE
data analyst internship in France
alternance développeur France
```

It splits on the last ` in ` to separate role from location, and detects internship/alternance/stage keywords automatically. Send a **PDF** (as a file, not a photo) to store your CV. Send `/reset` to clear your CV and search history.

**Website:** pick a field or type a role, choose country, optionally add city, date window, job type, remote-only, internship/alternance toggle, and a CV.

**Daily digest:** edit the **Interest Profile** node once; it runs itself.

**Limits:** each user (Telegram chat or browser) gets 15 searches/day. Hitting the cap returns a clear message rather than failing silently.

---

## Gotchas worth knowing

Each of these cost real debugging time:

- **Empty lookups halt everything.** See the Always Output Data note above. The execution reports *success* while doing nothing.
- **Telegram caps messages at 4096 characters.** Long result lists get rejected outright. Formatters split at 3800.
- **The country parameter is separate from the query text.** Searching "jobs in France" while the country parameter says `us` returns US jobs. It defaults to `us`, so unmapped locations fail quietly.
- **Alternance/apprenticeship postings are frequently mis-tagged** as full-time by job boards, so filtering by employment type alone misses them — the fix lives in the AI prompt, not the API filter.
- **Draft is not published.** Editing a node changes the draft. The live webhook and bot keep running the last published version until you publish again.
- **The n8n editor locks the workflow.** API edits fail while the canvas is open in a browser tab — and edits made while it's open can be silently lost.
- **Scanned PDFs yield no text.** CV extraction needs a text-based PDF.
- **Every trigger runs the shared pipeline**, so without gate nodes each trigger fires *all* delivery branches — the daily digest was arriving every time someone used the bot, until a filter gated it to the actual schedule.
- **A hardcoded node reference breaks on other paths.** An expression pointing directly at one trigger's node name throws "node hasn't been executed" when a different trigger runs the same shared step. Use `isExecuted` checks or try/catch fallbacks instead.

---

## Limitations

- **Single-tenant by design.** All credentials are one person's. Sharing the bot works technically — chat IDs and rate limits are handled per-user — but every message still spends *your* API quota, just capped at 15/day/user now.
- **The webhook is unauthenticated.** Anyone with the URL can trigger searches, subject to the same rate limit.
- **Fit scores are a language model's judgment**, not an ATS. Useful as direction, not as a verdict.
- **Coverage varies by market.** Strong in the US, UK and France; thinner in smaller markets. Narrow filters (remote + full-time + past week) can empty a result set fast.
- **Emails are rarely present.** Most postings link to an apply page rather than an address. The field is populated only when a real address appears in the description — never invented.

---

## Repo layout

```
n8n/workflow.json   the workflow, credentials stripped
site/index.html     standalone web app
README.md           this file
```

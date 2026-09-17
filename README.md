# AI Job Search Agent

Searches live job postings, scores each one against your actual CV, and tells you **what you're missing** — the thing no job board shows you.

Runs three ways: a Telegram bot you message directly, a web app, and a daily digest.

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
- Never shows you the same job twice
- Logs every match to a Google Sheet, building a dataset over time
- Works from Telegram, from a website, or automatically every morning

---

## Architecture

Three entry points feed one shared pipeline, then split into separate delivery formats:

```
Schedule (8AM) ─┐
Telegram msg ───┼──► Fetch jobs ──► Normalise ──► Load CV ──► Filter seen ──► AI scoring ──┬──► Telegram digest
Web request ────┘                                                                          ├──► Telegram reply
                                                                                           ├──► JSON to website
                                                                                           └──► Sheet + seen-jobs log
```

Two filter nodes gate the delivery branches so each trigger only fires its own output. Two data tables (`cv_store`, `seen_jobs`) hold per-user state, keyed by Telegram chat ID or browser session ID.

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
2. Create a Telegram credential in n8n with that token
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

Create two data tables in your n8n project:

| Table | Columns |
|---|---|
| `cv_store` | `chat_id`, `cv_text`, `updated_at` (all string) |
| `seen_jobs` | `chat_id`, `job_link`, `seen_at` (all string) |

Then point **Load CV** / **Save CV** at the first, and **Load Seen Jobs** / **Record Seen Jobs** at the second.

> **Leave "Always Output Data" enabled** on *Load CV* and *Load Seen Jobs*. A first-time user has neither a CV nor history, those nodes return zero rows, and in n8n a node with no output halts the branch — the search dies silently with no error.

### 7. The website (optional)

`site/index.html` is one self-contained file. Update `ENDPOINT` at the top of the `<script>` to your own webhook URL, then deploy anywhere static — drag the folder onto [Netlify Drop](https://app.netlify.com/drop), or enable GitHub Pages.

The CV is parsed **in the visitor's browser**; only extracted text reaches the backend, never the file.

### 8. Publish

Hit **Publish** in n8n. Nothing runs automatically — not even the bot listening for messages — until the workflow is active. n8n keeps draft and published versions separate, so **publish again after every edit**.

---

## Using it

**Telegram:** message the bot.

```
cloud engineer in France
marketing
cabin crew in UAE
```

It splits on the last ` in ` to separate role from location. Send a **PDF** (as a file, not a photo) to store your CV.

**Website:** pick a field or type a role, choose country, optionally add city, date window, job type, remote-only, and a CV.

**Daily digest:** edit the **Interest Profile** node once; it runs itself.

---

## Gotchas worth knowing

Each of these cost real debugging time:

- **Empty lookups halt everything.** See the Always Output Data note above. The execution reports *success* while doing nothing.
- **Telegram caps messages at 4096 characters.** Long result lists get rejected outright. Both formatters split at 3800.
- **The country parameter is separate from the query text.** Searching "jobs in France" while the country parameter says `us` returns US jobs. It defaults to `us`, so unmapped locations fail quietly.
- **Draft is not published.** Editing a node changes the draft. The live webhook and bot keep running the last published version until you publish again.
- **The n8n editor locks the workflow.** API edits fail while the canvas is open in a browser tab.
- **Scanned PDFs yield no text.** CV extraction needs a text-based PDF.
- **Every trigger runs the shared pipeline**, so without gate nodes each trigger fires *all* delivery branches — the daily digest was arriving every time someone used the bot.

---

## Limitations

- **Single-tenant by design.** All credentials are one person's. Sharing the bot works technically — chat IDs are handled per-user — but every message spends *your* API quota, with no rate limiting.
- **The webhook is unauthenticated.** Anyone with the URL can trigger searches on your account.
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

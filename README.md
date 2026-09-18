# AI Job Search Agent

Searches live job postings, scores each one against your actual CV, and tells you **what you're missing** — the thing no job board shows you.

Runs three ways: a Telegram bot you message directly, a web app, and a daily digest.

```
Senior Cloud Engineer · Acme Corp
Fit: 72/100
Missing: Terraform, Kubernetes at scale, on-call experience
```

## Repo structure

```
n8n/workflow.json   — the importable n8n workflow (credentials stripped to name only)
site/index.html     — a standalone web UI
README.md           — this file
```

---

## What it does

- Searches live postings from LinkedIn, Indeed, Glassdoor and others through a licensed aggregator (JSearch)
- Ranks every result against a free-text description of what you want
- With a CV uploaded, adds a 0–100 fit score and a specific list of requirements your CV doesn't evidence
- Company-specific search 
- Detects internship / French *alternance* (work-study apprenticeship) requests and reasons about them explicitly, since job boards routinely mis-tag them
- Never shows you the same job twice
- Filters by country, city, date posted, job type, and remote-only
- Logs every match to a Google Sheet, building a dataset over time
- Automatic AI model fallback — if the primary model hits a quota or errors, a second model picks up the same request automatically
- Alerts you on Telegram if any part of the pipeline fails
- `/reset` command clears your CV and search history

---

## Architecture

Three entry points feed one shared pipeline, then split into separate delivery formats:

```
Schedule (8AM) ──────────────────────────────┐
Telegram msg ────────────────────────────────┤
Web request ─────────────────────────────────┘
                                               ▼
                          Fetch jobs ──► Normalise ──► Load CV ──► Filter seen ──► AI scoring
                                                                                        │
                                     ┌──────────────────────┬─────────────────┬────────┤
                                     ▼                      ▼                 ▼        ▼
                            Telegram digest         Telegram reply      Website JSON   Sheet + seen-jobs log
```

The AI scoring step (`Score Matches (AI)`) has two model sub-nodes attached: a primary and a fallback. If the primary errors — quota exceeded, billing issue, model deprecated — n8n automatically retries the exact same request on the fallback before giving up. This is a native n8n feature (`Enable Fallback Model`), not custom code.

A document sent to the bot routes to CV storage; `/reset` routes to clearing that user's data; any node failure anywhere routes to a Telegram alert.

Two data tables hold per-user state: `cv_store` (your CV) and `seen_jobs` (dedupe history), keyed by Telegram chat ID or browser session ID.

**No rate limiting or usage caps** — this is meant to be self-hosted, one copy per person. If you're deploying this for public/shared use, add your own limits.

---

## Stack

| Piece | Role |
|---|---|
| [n8n](https://n8n.io) | Orchestration, scheduling, state |
| [JSearch](https://rapidapi.com/letscrape-6bRBa3QguO5/api/jsearch) (RapidAPI) | Live job data |
| Google Gemini (primary + fallback model) | Relevance scoring and CV gap analysis |
| Telegram Bot API | Chat interface and delivery |
| Google Sheets | Persistent log of every match |
| pdf.js | CV text extraction, in the browser |

The AI step is a **Basic LLM Chain with two swappable model sub-nodes** (primary + fallback) — deliberately not hardwired to one provider or one model, so a quota hit or a deprecated model ID doesn't take the whole thing down.

---

## Setup

### 1. Import the workflow

In n8n: **Workflows → Import from File** → `n8n/workflow.json`.

Every node carries a note explaining what it does and what it needs.

### 2. RapidAPI (job data)

1. Subscribe to [JSearch](https://rapidapi.com/letscrape-6bRBa3QguO5/api/jsearch) — the free tier is 200 requests/month, each search uses 3 (one per page fetched), so budget accordingly
2. Copy your key from the Code Snippets panel
3. In n8n, open **Fetch Jobs (JSearch)** → create a **Header Auth** credential:
   - Name: `X-RapidAPI-Key`
   - Value: your key

### 3. Telegram bot

1. Message **@BotFather** → `/newbot` → copy the token
2. Create a Telegram credential in n8n with that token and attach it to every Telegram node
3. Find your chat ID (message **@get_id_bot**) and put it in the **Send to Telegram** node
4. Message your bot once — bots can't message you until you've messaged them first

### 4. AI model (primary + fallback)

Open **Gemini Model** and **Gemini Fallback Model**, connect a Google credential to both. Generate a free key at [Google AI Studio](https://aistudio.google.com/apikey).

Model names drift over time as Google retires/renames them — if either node ever errors with "model not found" or "no longer available," check [Google's current model list](https://ai.google.dev/gemini-api/docs/models) and update the `modelName` field. This happened once already during development (`gemini-2.5-flash-lite` was retired mid-project).

Any provider works for either sub-node: replace it, leave the rest alone.

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

Point **Load CV** / **Save CV** / **Delete CV Row** at `cv_store`; **Load Seen Jobs** / **Record Seen Jobs** / **Delete Seen Jobs Rows** at `seen_jobs`.

> **Leave "Always Output Data" enabled** on *Load CV* and *Load Seen Jobs*. A first-time user has neither a CV nor history, those nodes return zero rows, and in n8n a node with no output halts the branch — the search dies silently with no error.

### 7. Error alerting

Set this workflow as its **own Error Workflow**: `⋯` menu → Settings → Error Workflow → select this same workflow. This can't be done via the API — it's a one-time manual step.

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
Consultant at EY
jobs at Google in Germany
```

Supports role, location (` in `), and company (` at `) in any combination. Send a **PDF** to store your CV. Send `/reset` to clear your CV and search history.

**Website:** pick a field or type a role, choose country, optionally add city, company, date window, job type, remote-only, internship/alternance toggle, and a CV.

**Daily digest:** edit the **Interest Profile** node once; it runs itself.

---

## Gotchas worth knowing (found the hard way)

- **Empty lookups halt everything.** A node with `alwaysOutputData` off that returns zero rows silently kills the entire branch downstream — the execution reports *success* while doing nothing. Applies to any Data Table "get" with no matching rows.
- **A write node dropped my search fields.** Chaining a node through a database write operation lost the input data — write nodes typically don't pass through their input, only their own result. Keep write operations as side branches, not in the main data path.
- **A Switch node's output indices shift when you add a rule.** Adding a new rule to a Switch inserts a new output and pushes the fallback output's index down — any existing connection to the old fallback index needs to move too, or it silently points at a dead end.
- **"Payment required" doesn't always mean billing is missing.** A free-tier model can return payment-style errors once its daily quota is fully exhausted, not just a rate-limit warning. The fallback model exists specifically for this.
- **Model IDs get deprecated without much warning.** If a model node suddenly 404s with "no longer available," it's Google retiring that ID — the error message usually names the replacement directly.
- **Telegram caps messages at 4096 characters.** Long result lists get rejected outright. Formatters split at 3800.
- **The country parameter is separate from the query text.** Searching "jobs in France" while the country parameter says `us` returns US jobs. It defaults to `us`, so unmapped locations fail quietly.
- **Alternance/apprenticeship postings are frequently mis-tagged** as full-time by job boards, so filtering by employment type alone misses them — the fix lives in the AI prompt, not the API filter.
- **Draft is not published.** Editing a node changes the draft. The live webhook and bot keep running the last published version until you publish again — and having the editor open while making API changes can cause edits to silently revert or diverge from the draft.
- **Scanned PDFs yield no text.** CV extraction needs a text-based PDF.

---

## Limitations

- **Personal tool by design.** No auth, no per-user isolation beyond chat ID/session ID, no shared rate limiting. Fine for self-hosted personal use; add your own protections before exposing it publicly.
- **Fit scores are a language model's judgment**, not an ATS. Useful as direction, not as a verdict.
- **Coverage varies by market.** Strong in the US, UK and France; thinner in smaller markets. Narrow filters (remote + full-time + past week) can empty a result set fast.
- **Emails are rarely present.** Most postings link to an apply page rather than an address. The field is populated only when a real address appears in the description — never invented.
- **Company search isn't a hard filter.** It biases the search query and instructs the AI to exclude non-matches, but depends on JSearch's underlying data actually having current postings for that company.

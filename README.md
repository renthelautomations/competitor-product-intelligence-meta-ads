# Competitor Product Intelligence System — META ADS

An end-to-end automated competitor ad intelligence pipeline built with n8n. Scrapes the Meta Ads Library daily via Apify, detects new competitor ads, tracks how long they run, identifies winning creatives (3+ days active), and delivers structured alerts to Telegram — all logged to Google Sheets.

## What the System Does

This is a two-workflow system:

### 1. Main Workflow (`workflows/Competitor Product Intelligence System - META ADS.json`)

Runs every morning at 9:00 AM with two sequential flows:

**Scraping & New Ad Detection Flow**
- Reads the existing ads database from Google Sheets to know which ad IDs are already tracked
- Builds a structured Apify scrape request from configured Meta Ads Library URLs (by keyword/competitor)
- Fires the Apify Facebook Ads Library Scraper actor and waits 2 minutes for it to finish
- Fetches the completed dataset from Apify
- Normalizes each ad record: extracts landing URL, start date, creative type (VIDEO / IMAGE / TEXT), and active status
- Merges scraped ads with the existing database and deduplicates
- Filters to active-only, new-only ads
- **If new ads found:** saves them to Google Sheets with status `NEW`, then sends a grouped summary to Telegram broken down by product (landing URL)
- **If no new ads:** sends a "no new ads today" alert to Telegram

**Database Update & Winner Detection Flow**
- Reads the full ads database after the new-ad phase completes
- Recalculates `days_active` for every tracked ad based on its `start_date`
- Updates `last_seen`, `status`, and `days_active` for all rows in the sheet
- Runs three parallel processes:
  - **Winning Ads Report** — identifies all ads active 3+ days, groups by product, sorts by ad count, sends ranked leaderboard to Telegram
  - **Run Log** — writes today's scrape stats (total scraped, new found, winning count) to a separate `Run Logs` sheet
  - **New Winner Alert** — detects ads that just crossed the 3-day threshold for the first time (not yet notified), sends a dedicated "new winner" alert to Telegram, and marks them as `notified_winner: true` in the sheet

### 2. Error Handler (`workflows/competitor-error-handler.json`)

Attaches to the main workflow as an error workflow via `settings.errorWorkflow`:
- Captures execution errors from any failed node
- Formats a structured alert with: workflow name, failed node, error message, execution ID, and timestamp
- Sends the alert immediately to Telegram

## Ad Lifecycle

```
Scraped → NEW → UPDATED (daily recalc) → WINNING AD (3+ days active)
```

| Status | Meaning |
|--------|---------|
| `NEW` | First time this ad ID was detected |
| `UPDATED` | Ad is still active; days_active and last_seen refreshed |
| `WINNING AD` | Ad has been running for 3+ days — proven creative |

## Tools Used

| Tool | Purpose |
|------|---------|
| **n8n** | Workflow automation engine |
| **Apify** | Facebook Ads Library Scraper actor |
| **Google Sheets** | Ads database and run log storage |
| **Telegram Bot** | Real-time alerts and daily digests |

## Google Sheets Schema

### `ads_db` Sheet

```
ad_id | page_name | ad_url | landing_url | start_date | first_seen | last_seen | creative_type | status | is_active | days_active | notified_winner
```

### `Run Logs` Sheet

```
date | ads_scraped | new_ads_found | winning_ads
```

Status flow: `NEW` → `UPDATED` → `WINNING AD`

## Telegram Alerts

The workflow sends up to five messages per daily run:

| Alert | Trigger |
|-------|---------|
| **New Ads Summary** | One or more new competitor ads detected today |
| **No New Ads** | Zero new ads found in today's scrape |
| **Winning Ads Leaderboard** | Daily ranked list of all ads running 3+ days |
| **New Winner Alert** | An ad just hit 3 days active for the first time |
| **Error Alert** | Any workflow node fails |

## Setup & Configuration

### Step 1 — Replace Placeholders

Before importing, search and replace all placeholders in both JSON files:

| Placeholder | Replace With |
|-------------|-------------|
| `YOUR_APIFY_API_TOKEN` | Your Apify API token (from Apify Console → Settings → API) |
| `YOUR_GOOGLE_SHEET_ID` | Google Sheets document ID (from the URL) |
| `YOUR_TELEGRAM_CHAT_ID` | Your Telegram user or group chat ID |
| `YOUR_GOOGLE_SHEETS_CREDENTIAL_ID` | n8n Google Sheets OAuth2 credential ID |
| `YOUR_TELEGRAM_CREDENTIAL_ID` | n8n Telegram credential ID |
| `YOUR_ERROR_WORKFLOW_ID` | The n8n workflow ID of the Error Handler (assigned after import) |

### Step 2 — Set Up Credentials in n8n

Create the following credentials in your n8n instance:
- **Telegram** — Bot API token (create a bot via @BotFather, find your chat ID via @userinfobot)
- **Google Sheets OAuth2** — OAuth2 app with Sheets scope
- **Apify** — API token stored directly in the HTTP Request node URLs (no n8n credential needed)

### Step 3 — Configure Your Google Sheet

Create a Google Sheet named **Meta Ads Detector** with two tabs:

1. **`ads_db`** — with columns matching the schema above
2. **`Run Logs`** — with columns: `date`, `ads_scraped`, `new_ads_found`, `winning_ads`

### Step 4 — Configure Apify

1. Go to [Apify Console](https://console.apify.com) and sign up
2. Find the **Facebook Ads Library Scraper** actor by `curious_coder`
3. Copy your API token from **Settings → Integrations → API token**
4. Replace `YOUR_APIFY_API_TOKEN` in both HTTP Request nodes in the main workflow

### Step 5 — Set Your Target URLs

In the **Meta Ads URL** Set node, update the URL entries with your actual Meta Ads Library search URLs:

1. Go to [Meta Ads Library](https://www.facebook.com/ads/library/)
2. Search for your competitor keywords or page names
3. Copy the full URL from your browser and paste it into the node

You can add, remove, or rename as many URL entries as needed — the URL Merger node handles any number dynamically.

### Step 6 — Import Workflows

1. In n8n, go to **Workflows → Import from File**
2. Import `competitor-error-handler.json` first
3. Note the workflow ID assigned to the Error Handler (visible in the URL after opening it)
4. Replace `YOUR_ERROR_WORKFLOW_ID` in the main workflow JSON with that ID
5. Import `Competitor Product Intelligence System - META ADS.json`
6. Reconnect all credentials in each node

### Step 7 — Activate

Activate the **Error Handler** first, then the **Main Workflow**.

## How the Winner Detection Works

An ad is considered a **winning ad** when:
- It has been continuously active for **3 or more days** (`days_active >= 3`)
- It has `is_active: true`
- It has **not yet been flagged** as a winner in this session (`notified_winner` is not `true`)

When a new winner is detected:
1. A Telegram alert fires with the product name, ad count, creative type, and landing URL
2. The ad row in Sheets is updated with `notified_winner: true` to prevent duplicate alerts on future runs

The daily **Winning Ads Leaderboard** shows all currently active ads at 3+ days — regardless of when they first crossed the threshold — sorted by ad count descending with a strength indicator:

- `***` — 5+ ads running to the same landing page
- `**` — 3–4 ads
- `*` — 1–2 ads

## Notes

- The Apify scraper is set to fetch **10 ads per URL** by default. Increase the `count` value in the **URL Merger** node for broader coverage.
- The deduplication logic in the **Filter Node** allows the same ad to reappear on the day it was `first_seen`, then blocks it on all future days — preventing re-saves while allowing same-day re-runs.
- `days_active` is computed from `start_date` (the date the ad first ran on Meta), not from `first_seen` (when your scraper first picked it up).

## File Structure

```
/
├── workflows/
│   ├── Competitor Product Intelligence System - META ADS.json   # Main scraping, detection, and winner tracking workflow
│   └── competitor-error-handler.json                            # Error monitoring and Telegram alert workflow
└── README.md
```

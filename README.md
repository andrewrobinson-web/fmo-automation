# FMO Media — Automation Scripts

Internal automation tools for client data collection, scorecard generation, and viral content detection.

---

## Scripts

| Script | Purpose | Command |
|---|---|---|
| `viral_detector.py` | Scan all client social profiles for viral content + pull metrics | `python3 viral_detector.py --full` |
| `asana_to_sheets.py` | Pull all Asana comments from master client roster → Google Sheets | `python3 asana_to_sheets.py` |

---

## Setup

### Requirements

```bash
pip3 install requests gspread google-auth
```

### Credentials

All credentials are stored as environment variables — never hardcode them in scripts.

| Variable | What it is | Where to get it |
|---|---|---|
| `AGORAPULSE_API_KEY` | Agorapulse API key | Agorapulse → Profile → Personal Settings → API Keys |
| `ASANA_PAT` | Asana Personal Access Token | Asana → Profile → My Settings → Apps → Manage Developer Apps |
| `SLACK_WEBHOOK_URL` | Slack incoming webhook (optional) | Slack → Apps → Incoming Webhooks |
| `SLACK_BOT_TOKEN` | Slack bot token (optional, for thumbnails) | Slack → Apps → OAuth & Permissions |

### Google Sheets credentials

The scripts write to Google Sheets via a service account.

1. Go to **console.cloud.google.com**
2. Create a project → Enable **Sheets API** + **Drive API**
3. Create a **Service Account** → download the JSON key
4. Save it as `~/Desktop/google_creds.json`
5. Share any target Google Sheet with the service account email (`...@...iam.gserviceaccount.com`) — Editor access

---

## viral_detector.py

Connects to Agorapulse across all 5 FMO Media orgs, scans ~680 social profiles (Facebook, Instagram, TikTok), and either detects viral content or pulls aggregated metrics.

### Organization & Workspace IDs

```python
ORG_WORKSPACES = [
    ("290398", "190399"),   # FMO Media 4
    ("217688", "117689"),   # FMO Media 1
    ("510368", "410258"),   # FMO Media 2
    ("377130", "277088"),   # FMO Media 6
    ("352521", "252495"),   # FMO Media 5
]
```

### Commands

```bash
# Set credentials
export AGORAPULSE_API_KEY="your_key"
export ASANA_PAT="your_pat"

# First time setup — verify connections
python3 viral_detector.py --discover-orgs
python3 viral_detector.py --discover-profiles
python3 viral_detector.py --test-one

# Pull metrics + Asana comments → Google Sheets (monthly run)
python3 viral_detector.py --full

# Pull metrics only → Google Sheets
python3 viral_detector.py --metrics

# Scan for viral content (10K+ views) → HTML report + Slack alerts
python3 viral_detector.py --scan
```

### --full output

Writes a dated tab (e.g. `2026-03-17 Full Pull`) to the scorecard Google Sheet with:

- **54 metric columns** — T-90→T-60, T-60→T-30, T-30→Now × FB/IG/TT × Followers/Reach/Views/Clicks/Comments/Shares
- **1 Asana Comments column** — last 10 comments per client matched from Asana

### --scan output

| File | Contents |
|---|---|
| `viral_report_TIMESTAMP.html` | Interactive dashboard — leaderboard, charts, platform breakdown |
| `viral_showcase_TIMESTAMP.html` | 3D sphere visualization — sized by views, click to open post |
| `viral_videos_TIMESTAMP.csv` | All posts over 10K views |
| `all_videos_TIMESTAMP.csv` | Every post scanned |

### Viral thresholds

| Views | Label | Alert |
|---|---|---|
| 10,000+ | VIRAL | #viral-wins |
| 25,000+ | CASE STUDY | #viral-wins |
| 50,000+ | LEADERSHIP ALERT | #viral-wins + #leadership |
| 100,000+ | MEGA VIRAL | #viral-wins + #leadership |

### Rate limits

- Agorapulse: 500 requests per 30 minutes
- Script auto-pauses at 450 requests and waits for the window to clear
- Full run (~680 profiles): approximately 80-90 minutes

### Known platform quirks

| Platform | Notes |
|---|---|
| `FACEBOOK_INSTAGRAM` | Instagram's actual API type name (not `INSTAGRAM`) |
| TikTok | Uses `commentCount` / `shareCount` (not `commentsCount` / `sharesCount`) |
| `GOOGLE` | Not supported by Agorapulse Open API — returns error, skipped |
| `LINKEDIN_COMPANY` | Returns 0 posts — skipped |
| Followers | Pulled from audience report (not content report) |

---

## asana_to_sheets.py

Pulls all comments from every task in the **MASTER CLIENT ROSTER 2026** Asana project and writes them to a dated tab in Google Sheets.

### Configuration (edit at top of file)

```python
PAT          = os.environ["ASANA_PAT"]
PROJECT_GID  = "1201870248175720"       # MASTER CLIENT ROSTER 2026
SHEET_URL    = "https://docs.google.com/spreadsheets/d/1K7RoUhYuU3WfR3hDAzOGWamtS-jt6oE3-EI4E5qbuTg/edit"
CREDS_FILE   = os.path.expanduser("~/Desktop/google_creds.json")
```

### Run

```bash
export ASANA_PAT="your_pat"
python3 asana_to_sheets.py
```

### Output

Writes a dated tab (e.g. `2026-03-17`) to the Asana Comments Google Sheet with columns:

| Column | Description |
|---|---|
| Client | Asana task name |
| Date | Comment date |
| Author | Comment author |
| Comment | Comment text |

---

## Google Sheets

| Sheet | URL | Purpose |
|---|---|---|
| Asana Comments | [Link](https://docs.google.com/spreadsheets/d/1K7RoUhYuU3WfR3hDAzOGWamtS-jt6oE3-EI4E5qbuTg/edit) | Asana comments + full scorecard data pull output |
| FMO Master Client List | [Link](https://docs.google.com/spreadsheets/d/1B14D-IpMPSOvdCTenBBavSKY8MCPBQ0of_xEVTWu2GQ/edit) | Source for Agorapulse metrics (team tabs) |

---

## Monthly Scorecard Workflow

1. **Run the full pull** — pulls Agorapulse metrics + Asana comments for all clients
   ```bash
   AGORAPULSE_API_KEY="key" ASANA_PAT="pat" python3 viral_detector.py --full
   ```
2. **Add WhatsApp exports** — manually export from phone, drop `.txt` files in a folder (future automation TBD)
3. **Run scorecard analysis** — paste combined data into [Client Scorecard Analyst GPT](https://chatgpt.com/g/g-685d9b983d808191b2e91d400106f446-client-scorecard-analyst)
4. **Update health scores** — copy scores back to master Google Sheet

---

## Roadmap

- [ ] Checkpoint/resume for long API runs
- [ ] Auto-populate health scores to master sheet
- [ ] WhatsApp batch processor (folder of .txt exports)
- [ ] Automated daily cron for viral detection
- [ ] Slack → #viral-wins channel (currently posting to DM for testing)
- [ ] Thumbnail batching for large viral result sets (77+ videos)

---

## Troubleshooting

**401 Unauthorized**
- API key expired — regenerate in Agorapulse → Personal Settings → API Keys
- Make sure you're using `AGORAPULSE_API_KEY` not `AGORAPULSE_KEY`
- Pass credentials inline: `AGORAPULSE_API_KEY="key" python3 script.py` instead of `export`

**No Instagram profiles found**
- Instagram type is `FACEBOOK_INSTAGRAM` in Agorapulse API — not `INSTAGRAM`

**Followers showing 0**
- Followers come from the audience report, not the content report
- If audience report returns empty, the profile token may be expired in Agorapulse

**Script dies mid-run**
- Check `errors_TIMESTAMP.csv` for which profiles failed
- Re-run — already-written rows won't be duplicated (tab is cleared and rewritten)

**Google Sheets auth error**
- Confirm `google_creds.json` is on your Desktop
- Confirm the sheet is shared with the service account email

---

*Last updated: March 2026*

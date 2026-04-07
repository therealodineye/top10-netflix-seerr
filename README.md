# Netflix Top 10 → Seerr (n8n Workflow)

Automatically request the daily Netflix Top 10 movies and TV shows into your [Seerr](https://github.com/seerr-team/seerr) (or Overseerr) instance using [n8n](https://n8n.io). No paid APIs, no third-party accounts — just a daily curl to FlixPatrol and your own self-hosted stack.

## What it does

Every day at 08:00, the workflow:

1. Fetches the Netflix Top 10 for your chosen country from FlixPatrol
2. Parses the HTML to extract the top 10 movies and top 10 TV shows
3. Searches each title in Seerr to check if it already exists or has been requested
4. Skips anything already pending, processing, or available
5. For new titles, fetches full media details from Seerr to get season information
6. Logs in as a dedicated bot user (no admin privileges, no auto-approve)
7. Submits the request — movies as-is, TV shows requesting only the latest season

---

## Requirements

- [n8n](https://n8n.io) (self-hosted, tested on v2.15.0)
- [Seerr](https://github.com/seerr-team/seerr) or [Overseerr](https://github.com/sct/overseerr)
- CSRF Protection **disabled** in Seerr (Settings → General → uncheck Enable CSRF Protection)

---

## Setup

### 1. Create a bot user in Seerr

The workflow authenticates as a dedicated low-permission user so that requests are not auto-approved. This keeps you in control of what actually gets downloaded.

- Go to **Seerr → Users → Add User**
- Create a local account with an email and password
- Do **not** grant Auto-Approve permissions
- Note the email and password — you will need them below

### 2. Get your Seerr API key

- Go to **Seerr → Settings → General**
- Copy the API key shown at the top of the page

### 3. Disable CSRF Protection

- Go to **Seerr → Settings → General**
- Uncheck **Enable CSRF Protection**
- Save

This is required for the workflow to POST requests via the API without a browser session.

### 4. Import the workflow into n8n

- Open n8n
- Go to **Workflows → Import**
- Upload `Netflix-TOP10-Seerr.json`

### 5. Fill in your credentials

Open each of the nodes listed below and replace the placeholder values:

#### Search Seerr (node-005)
| Field | Value |
|-------|-------|
| URL | `http://YOUR_SEERR_HOST:5055/api/v1/search` |
| Header `X-Api-Key` | Your Seerr API key |

#### Fetch TV Seasons (node-010)
| Field | Value |
|-------|-------|
| URL | `http://YOUR_SEERR_HOST:5055/api/v1/...` |
| Header `X-Api-Key` | Your Seerr API key |

#### Login Bot User (node-009)
| Field | Value |
|-------|-------|
| URL | `http://YOUR_SEERR_HOST:5055/api/v1/auth/local` |
| Body `email` | Your bot user's email |
| Body `password` | Your bot user's password |

#### Request in Seerr (node-008)
| Field | Value |
|-------|-------|
| URL | `http://YOUR_SEERR_HOST:5055/api/v1/request` |

Replace `YOUR_SEERR_HOST` with the IP address or hostname of your Seerr instance, e.g. `192.168.1.100` or `seerr.yourdomain.com`. If you use HTTPS, change `http://` to `https://` in all URLs.

### 6. Activate the workflow

Toggle the workflow to **Active**. It will run automatically every day at 08:00.

---

## Changing the country

By default the workflow fetches the Netflix Top 10 for **Norway**. To change the country, open the **Fetch FlixPatrol** node and edit the URL:

```
https://flixpatrol.com/top10/netflix/norway/{{ $now.format('yyyy-MM-dd') }}/
```

Replace `norway` with your country's slug as it appears on FlixPatrol. Examples:

| Country | Slug |
|---------|------|
| United States | `united-states` |
| United Kingdom | `united-kingdom` |
| Germany | `germany` |
| France | `france` |
| Sweden | `sweden` |
| Denmark | `denmark` |

To find any country's slug, visit [flixpatrol.com/top10/netflix/](https://flixpatrol.com/top10/netflix/) and click your country. The slug is the last part of the URL.

---

## Changing the schedule

The workflow runs at 08:00 daily using the cron expression `0 8 * * *`. To change the time, open the **Daily 08:00** node and edit the expression. For example:

| Schedule | Cron |
|----------|------|
| Every day at 06:00 | `0 6 * * *` |
| Every day at midnight | `0 0 * * *` |
| Twice daily at 08:00 and 20:00 | `0 8,20 * * *` |

---

## Changing the streaming service

The workflow is built for Netflix but FlixPatrol supports other services. To switch, change `netflix` in the FlixPatrol URL to one of the following:

| Service | Slug |
|---------|------|
| Netflix | `netflix` |
| Amazon Prime | `amazon` |
| HBO / Max | `hbo` |
| Disney+ | `disney` |
| Apple TV+ | `apple-tv` |

---

## How the workflow handles duplicates

The **Check Status** node checks each title's `mediaInfo.status` in Seerr before requesting:

| Status | Meaning | Action |
|--------|---------|--------|
| Not in Seerr | Title unknown | Request it |
| 1 — Unknown | In Seerr but unhandled | Request it |
| 2 — Pending | Already queued | Skip |
| 3 — Processing | Being downloaded | Skip |
| 4 — Partially available | Some episodes available | Skip |
| 5 — Available | Already in your library | Skip |

This means running the workflow multiple times is safe — it will never submit duplicate requests.

---

## TV show behaviour

For TV shows, the workflow requests only the **latest season**. This avoids requesting an entire back catalogue for long-running shows that are trending. If you want all seasons instead, open the **Request in Seerr** node and change the `body` expression — replace the `.pop()` logic with `.map(s => s.seasonNumber)` to include all seasons.

---

## Node overview

| Node | Type | Purpose |
|------|------|---------|
| Daily 08:00 | Schedule Trigger | Fires the workflow once per day |
| Fetch FlixPatrol | HTTP Request | Downloads the FlixPatrol Top 10 HTML page |
| Parse Top10 Titles | Code | Extracts ranked titles from HTML using regex |
| Split Titles | Split Out | Splits the array into individual items for processing |
| Search Seerr | HTTP Request | Searches each title in Seerr by name |
| Check Status | Code | Checks media status and decides whether to request |
| Not Yet Requested? | IF | Filters out titles already in the system |
| Fetch TV Seasons | HTTP Request | Fetches full media details to get season list |
| Login Bot User | HTTP Request | Authenticates the bot user and retrieves a session cookie |
| Request in Seerr | HTTP Request | Submits the media request using the bot session |

---

## Troubleshooting

**Workflow fetches 0 titles**
FlixPatrol may have changed their HTML structure. Check the **Fetch FlixPatrol** node output and verify the page still contains ranked rows with the `table-hover:text-gray-400` class.

**403 on Request in Seerr**
Make sure CSRF Protection is disabled in Seerr settings.

**500 on Request in Seerr**
Usually means the seasons array is malformed. Check the **Fetch TV Seasons** node output and verify the `seasons` field is present.

**Titles are requested but not picked up by Radarr/Sonarr**
This is a Seerr configuration issue, not a workflow issue. Verify your Radarr/Sonarr connections are set up correctly in Seerr settings.

---

## Credits

Built with [n8n](https://n8n.io) and data from [FlixPatrol](https://flixpatrol.com).

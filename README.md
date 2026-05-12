# trmnl-ios-calendars

A private [TRMNL](https://usetrmnl.com) plugin that displays today's iOS Calendar events
on the e-ink display. Powered by the TRMNL iOS Companion app, which POSTs calendar data to a
webhook each morning.

## What it shows

Today's events sorted chronologically. All-day events appear first with a circle date badge;
timed events show a vertical accent bar, the event title, and start/end times — styled after
the iOS Calendar app. Days with no events show a quiet empty state.

## Local development

Requires [Docker Desktop](https://www.docker.com/products/docker-desktop/).

```powershell
# from the repo root
.\bin\serve.ps1
```

Then open `http://localhost:4567` in a browser. The server watches `src/` and `.trmnlp.yml`
for changes and reloads automatically.

### Fixture data

`.trmnlp.yml` contains sample events used during local preview. The real event data
(`resources/events.json`) is gitignored — it's written by the iOS Companion app and contains
personal calendar information.

The fixture also sets `trmnl.system.timestamp_utc` to anchor "today" for the dev server.
Update that value when testing on a different date — the comment in `.trmnlp.yml` shows the
formula.

## Deployment

This is a private TRMNL plugin using the **webhook** strategy. The iOS Companion app sends
calendar data directly to TRMNL's webhook endpoint. No hosted server is required.

### First-time setup

Copy `src/settings.example.yml` to `src/settings.yml` and set your plugin ID (the number at
the end of your plugin's URL on `trmnl.com`). This file is gitignored — it contains your
plugin ID and will be updated by `trmnlp push` with the full server response.

### Pushing markup changes

```powershell
# authenticate once (stores credentials in ~/.config/trmnlp)
docker run --rm -it `
  --volume "$env:USERPROFILE\.config\trmnlp:/root/.config/trmnlp" `
  trmnl/trmnlp login

# push markup to your TRMNL plugin
.\bin\push.ps1
```

Credentials are stored in `~/.config/trmnlp/config.yml` (outside the repo). Both
`src/settings.yml` and the credentials file are gitignored.

## Layouts

| Layout | Size | Description |
|---|---|---|
| Full | 800×480 | Up to 8 events, start + end times, description on timed events |
| Half Horizontal | 800×240 | 8 events across two columns, start time only |
| Half Vertical | 400×480 | 8 events, single column, start time only |
| Quadrant | 400×240 | 5 events, most compact |

## Data model

The iOS Companion app provides two Liquid variables:

**`{{ events }}`** — array of calendar event objects, sorted by `date_time`

| Field | Type | Notes |
|---|---|---|
| `summary` | string | Event title |
| `all_day` | boolean | `true` for all-day events |
| `start` | string | Local start time, `"HH:MM"` |
| `end` | string | Local end time, `"HH:MM"` |
| `date_time` | string | UTC ISO 8601 start timestamp |
| `start_full` | string | UTC ISO 8601 start timestamp (same as `date_time`) |
| `end_full` | string | UTC ISO 8601 end timestamp |
| `calname` | string | iOS calendar UUID |
| `status` | string | `"confirmed"`, `"tentative"`, etc. |
| `description` | string | Event notes, may be empty |

**`{{ trmnl }}`** — device and user context (see `resources/trmnl.json` for shape)

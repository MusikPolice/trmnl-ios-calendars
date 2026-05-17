# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```powershell
.\bin\serve.ps1   # start local dev server at http://localhost:4567 (auto-reloads on src/ changes)
.\bin\push.ps1    # push markup to your TRMNL plugin (requires credentials — see below)
```

First-time auth (stores credentials in `~/.config/trmnlp/config.yml`, outside the repo):
```powershell
docker run --rm -it --volume "$env:USERPROFILE\.config\trmnlp:/root/.config/trmnlp" trmnl/trmnlp login
```

**Fixture date**: `.trmnlp.yml` sets `trmnl.system.timestamp_utc` to anchor "today" in the dev server. When testing on a different date, update that value to the Unix timestamp of noon UTC on the target date (e.g. May 11 2026 12:00 UTC = `1746964800`).

## Project

TRMNL private plugin that renders iOS Calendar events on an 800×480 2-bit grayscale e-ink display.
The iOS Companion app POSTs calendar event data to a TRMNL webhook each morning. This repo contains
the Liquid markup that TRMNL renders into a screen image.

## Tech stack

- **Liquid** (Shopify-flavored, Ruby) — templating language for all markup
- **TRMNL CSS framework** — loaded automatically by the device; use its classes where possible
- **trmnlp** — local dev server, run via Docker (`.\bin\serve.ps1`)
- No build step, no package manager, no server — just `.liquid` files

## Key files

| File | Purpose |
|---|---|
| `src/full.liquid` | Full-screen layout (800×480) — primary view, date-grouped with headers |
| `src/half_horizontal.liquid` | 800×240 layout — two-column flat list |
| `src/half_vertical.liquid` | 400×480 layout — single column flat list |
| `src/quadrant.liquid` | 400×240 layout — most compact, title only |
| `src/shared.liquid` | CSS/markup included before all layout files |
| `src/settings.yml` | Plugin metadata — **gitignored**, contains plugin ID; updated by `trmnlp push` |
| `src/settings.example.yml` | Committed template — copy to `settings.yml` and set your plugin ID |
| `.trmnlp.yml` | Dev server config and fixture data |
| `bin/serve.ps1` | Docker runner for trmnlp |
| `resources/events.json` | Sample `{{ events }}` payload (gitignored, personal) |

## Data model

`{{ events }}` is an array sorted by `date_time` (UTC). Every item has:

| Field | Type | Notes |
|---|---|---|
| `summary` | string | Event title |
| `all_day` | boolean | True for all-day events |
| `start` | string | Local start time, "HH:MM" format |
| `end` | string | Local end time, "HH:MM" format |
| `date_time` | string | UTC ISO 8601 start timestamp |
| `start_full` | string | UTC ISO 8601 start timestamp (same as `date_time`) |
| `end_full` | string | UTC ISO 8601 end timestamp |
| `calname` | string | iOS calendar UUID (not human-readable) |
| `status` | string | "confirmed", "tentative", etc. |
| `description` | string | Event notes/body (may be empty) |

`{{ trmnl }}` key paths used in templates:
- `trmnl.system.timestamp_utc` — Unix timestamp of last refresh (integer)
- `trmnl.user.utc_offset` — seconds offset from UTC (e.g. -14400 for EDT)

## Timezone handling

Always compute today's local date by shifting the UTC timestamp before formatting:

```liquid
{% assign local_ts = trmnl.system.timestamp_utc | plus: trmnl.user.utc_offset %}
{% assign today    = local_ts | date: "%Y-%m-%d" %}
```

For each event's local date, apply the same shift to its `date_time` field:

```liquid
{% assign event_local_ts = e.date_time | date: "%s" | plus: trmnl.user.utc_offset %}
{% assign event_date     = event_local_ts | date: "%Y-%m-%d" %}
```

Never use `date: "%Y-%m-%d"` directly on UTC timestamps — Liquid formats in UTC (the server timezone),
which causes off-by-one day errors for users in negative UTC offsets in the evening hours.

## Display logic

All layouts filter to events where `event_date == today` — only today's events are shown.
The `| sort: "date_time"` filter orders all-day events first (midnight timestamp), then timed
events chronologically. ISO 8601 strings sort correctly with lexicographic comparison.

**iOS Calendar-inspired row structure:**
- **Timed events**: solid black 3px vertical bar on left, title + optional description left, start time + end time stacked right
- **All-day events**: bar hidden via `visibility: hidden` (preserves spacing), title left, "all-day" label right
- **Title bar**: dynamic date string computed from `local_ts | date: "%A, %b %-d"` (e.g. "Monday, May 11")

Time formatting uses `event_local_ts | date: "%-l:%M%P"` → e.g. "7:00pm", "10:30am"
End time uses `e.end_full | date: "%s" | plus: trmnl.user.utc_offset | date: "%-l:%M%P"`

## Display constraints

- 800×480 px, 4 shades of gray only (black, dark gray, light gray, white)
- No color — use weight, size, and shade to convey hierarchy
- TRMNL framework uses `flex--row` layout by default; use `flex flex--col` for vertical lists
- Font: Inter via Google Fonts — `<link>` tags and base `.screen` font-family are in `shared.liquid`

## Layout structure

```html
<div class="screen">
  <div class="view view--full">        <!-- or view--half_horizontal etc -->
    <div class="title_bar">
      <p class="title">...</p>
    </div>
    <div class="flex flex--col">
      <div class="list-wrap">
        <!-- rows here -->
      </div>
    </div>
  </div>
</div>
```

**Important — `flex--col` has `align-items: center`** in the TRMNL framework CSS. For wide
layouts (Full, Half Horizontal) this centers `.list-wrap` by default, which is what we want.
For narrow layouts (Half Vertical, Quadrant) where we want left-alignment, override with
`align-self: flex-start` on `.list-wrap` — margin alone will not work.

## Row cap per layout

- Full (800×480): 8 events — bar + title + description (if present) + start/end time
- Half Horizontal (800×240): 8 events across two columns of 4 — bar + title + start time only
- Half Vertical (400×480): 8 events, single column — bar + title + start time only
- Quadrant (400×240): 5 events — bar + title + start time only, tightest padding

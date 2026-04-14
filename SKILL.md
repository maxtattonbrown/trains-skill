---
name: trains
description: "Train Tracks: Check UK train commute times, disruptions, and add trains to Fantastical (or Apple Calendar if Fantastical not installed). Uses the Huxley2 API (National Rail Darwin proxy). Supports live departures, delay checking, baseline timetable caching, and .ics calendar export. Trigger on '/trains', or when user mentions train times, commute, or checking departures. Examples - '/trains' shows next departures, '/trains disruptions' checks for delays, '/trains add 08:15' adds a train to calendar, '/trains setup' configures stations."
---

# Train Tracks — Commute Tracker

Check live UK train departures, disruptions, and add trains to Apple Calendar.

## Config

- **Config file**: `~/.claude/trains/config.json`
- **Timetable cache**: `~/.claude/trains/timetable.json`

Read config on every invocation. If missing, run setup flow.

```json
{
  "home": {"name": "Leighton Buzzard", "crs": "LBZ"},
  "work": {"name": "London Euston", "crs": "EUS"},
  "theme": "fast",
  "sort": "depart",
  "filter": null,
  "countdown_mins": 60
}
```

| Config key | Values | Default | Purpose |
|------------|--------|---------|---------|
| `theme` | `fast`, `board`, `clean` | `fast` | Display theme |
| `sort` | `depart`, `arrive` | `depart` | Sort trains by departure or arrival time |
| `filter` | `null`, `fast`, `semi`, `stopping` | `null` | Only show trains of this type |
| `countdown_mins` | integer | `60` | Show countdown in status line when train is within this many minutes |

Pass as CLI flags too: `--sort arrive`, `--fast`, `--semi`, `--stopping`, `--board`

## API Reference

Base URL: `https://national-rail-api.davwheat.dev` — no API key needed.

| Endpoint | Purpose |
|----------|---------|
| `/departures/{from}/to/{to}?expand=true` | Live departures filtered to destination |
| `/delays/{from}/to/{to}/60` | Disruption check (server-side) |

Optional params: `timeOffset` (-120 to 119 mins), `timeWindow` (-120 to 120 mins).

## Fetching Data

Use the bundled fetch script to get parsed JSON:
```bash
curl -s "https://national-rail-api.davwheat.dev/departures/{from}/to/{to}?expand=true" \
  | python3 ~/.claude/skills/trains/scripts/fetch.py {DEST_CRS}
```

This outputs a compact JSON object with parsed services. Then render the board as **inline markdown text** (NOT Bash output) so it always displays fully in Claude Code without collapsing.

## CRITICAL: Rendering

**NEVER render the departures board inside a Bash tool call.** Claude Code collapses long Bash output behind a "click to expand" fold, which defeats the purpose.

Instead:
1. Use Bash to fetch data (via the fetch script or curl + inline python)
2. Render the departures board as **markdown text output** directly in your response

### Board format

Render as a markdown code block using box-drawing characters:

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│   {STATION NAME}                                             │
│   Departures to {destination}  ·  {HH:MM}                   │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  Time   Plat   Destination            Arr     Expected       │
├──────────────────────────────────────────────────────────────┤
│  07:15    3    {destination}          07:52   ✓ On time      │
│                via Watford Jn                                │
│  07:45    -    {destination}          08:22   ✗ Exp 07:48    │
│                all stations                                  │
│  08:15    -    {destination}          08:45   ✓ On time      │
│                ⚡ FAST                                        │
├──────────────────────────────────────────────────────────────┤
│  ● Good service on this route                                │
└──────────────────────────────────────────────────────────────┘
```

### Route type labels

Based on number of intermediate stops:
- 0 stops → `⚡ FAST` (direct service)
- 1-3 stops → `via {stop names}` (semi-fast)
- 4+ stops → `all stations` (stopper)

### Status formatting

- On time → `✓ On time`
- Delayed with expected time → `✗ Exp {time}`
- Cancelled → `✗ CANCELLED`
- Delayed (no estimate) → `✗ Delayed`

## Auto-refresh

Before using the timetable cache, check if it's stale. If ANY of these are true, refresh silently before answering:

1. **Age**: `captured` date in timetable.json is more than 7 days ago
2. **Day-type mismatch**: `day_type` is "sunday" but today is a weekday (or vice versa)
3. **Empty/missing**: timetable.json doesn't exist or has no services

**Refresh process:** Run the `/trains refresh` flow (query API at multiple time offsets, save to timetable.json). Then proceed with the fresh data. Don't ask — just do it. Mention briefly: "Refreshed timetable (was stale)."

This does NOT apply when live API data is available (i.e., querying today's departures). Live always takes priority; the timetable is only a fallback for future dates or when the API is down.

## Modes

### `/trains` or `/trains next`

1. Read config
2. Direction: before 12:00 → home-to-work, after → work-to-home
   - Override: `/trains to work`, `/trains to home`
3. If theme is `fast` (default): fetch data and render a compact inline summary only. No Terminal popup. Example:
   ```
   Trains: Euston → Leighton Buzzard
   Next: 14:56 (⚡ fast, plat 12) · 15:09 (all stations) · 15:23 (via Watford Jn)
   ● Good service
   ```
4. If theme is `board` (or user passes `--board`): open an animated departures board in a Terminal.app window using:
   ```bash
   osascript -e 'tell application "Terminal" to activate' && osascript -e 'tell application "Terminal" to do script "curl -s \"https://national-rail-api.davwheat.dev/departures/{from}/to/{to}?expand=true\" | python3 ~/.claude/skills/trains/scripts/departures.py {DEST_CRS} --theme board --animate; echo; echo \"Press any key to close\"; read -n1; exit"'
   ```
   Also render the compact inline summary in the conversation.
5. If theme is `clean`: render the box-drawing board as inline markdown text (no Terminal popup)
6. Fallback to baseline timetable if API unreachable

### `/trains setup`

1. Ask for home and work station names
2. Validate: `curl -s "https://national-rail-api.davwheat.dev/departures/{name}/1"` — check `locationName` and `crs` in response
3. Write config, then run `/trains refresh`

### `/trains disruptions`

1. Query delays both directions:
   ```
   curl -s "https://national-rail-api.davwheat.dev/delays/{home_crs}/to/{work_crs}/60"
   curl -s "https://national-rail-api.davwheat.dev/delays/{work_crs}/to/{home_crs}/60"
   ```
2. Report delayed/cancelled services or "Good service"

### `/trains add {time}` or `/trains add next` or `/trains add`

1. If no time specified, fetch departures and present an **AskUserQuestion** prompt with the top 3-4 options (prioritise fast trains). If a time is given, match by departure time. If `next`, use first upcoming.
2. Add the event to the calendar — use whichever method is available:

   **A) Fantastical MCP (preferred — use if `mcp__Fantastical__createCalendarItem` is available):**
   - Tool: `mcp__Fantastical__createCalendarItem`
   - `calendarId`: call `mcp__Fantastical__queryCalendars` first to find a trains/commute calendar; if none exists, omit and let Fantastical use the default
   - `description`: compact natural language, e.g. `"Train Leighton Buzzard to London Euston 10:47 arriving 11:17 platform 4 on 14 April 2026"`
   - `type`: `"event"`

   **B) .ics fallback (if Fantastical MCP is not available):**
   Write `/tmp/train-{date}-{time}.ics`, then:
   ```bash
   if [ -d "/Applications/Fantastical.app" ]; then
     open -a Fantastical /tmp/train-{date}-{time}.ics
   else
     open /tmp/train-{date}-{time}.ics
   fi
   ```
   ```
   BEGIN:VCALENDAR
   VERSION:2.0
   PRODID:-//Claude//Train Tracker//EN
   BEGIN:VEVENT
   DTSTART:{YYYYMMDD}T{HHMM}00
   DTEND:{YYYYMMDD}T{HHMM}00
   SUMMARY:Train: {from} → {to}
   LOCATION:{from} station{platform_info}
   DESCRIPTION:Operator: {operator}\nScheduled: {depart} → {arrive}
   END:VEVENT
   END:VCALENDAR
   ```

3. Write `~/.claude/trains/next.json` for status line countdown:
   ```json
   {
     "date": "2026-03-05",
     "depart": "16:26",
     "arrive": "16:55",
     "from": "London Euston",
     "to": "Leighton Buzzard",
     "route_type": "fast",
     "plat": "7"
   }
   ```
   The status line script reads this and shows a countdown chip when within `countdown_mins` of departure. Auto-clears after the train departs.
5. **Return journey offer**: If the added train is home→work (morning commute), offer to add a return:
   - Fetch evening departures from work→home
   - Use **AskUserQuestion** to present the next 3-4 fast/semi options as a pick list (include a "Skip" option)
   - If user picks one, add it to Calendar and save to `~/.claude/trains/return.json` (same format as `next.json`)
   - The status line script checks `return.json` after `next.json` has cleared (outbound departed), so the countdown seamlessly switches to the return train
   - If user declines, skip silently

   `return.json` format is identical to `next.json`. The status line script reads `next.json` first; if empty/departed, falls back to `return.json`.

### `/trains timetable`

Show cached baseline schedule from `~/.claude/trains/timetable.json`. If missing, prompt `/trains refresh`.

### `/trains refresh`

Capture baseline by querying multiple time offsets (covers ~4 hours per run). Save to timetable.json. Suggest running again at different times for full coverage.

## Response Parsing Reference

`trainServices[]`: `std`, `etd`, `platform`, `operator`, `operatorCode`, `isCancelled`, `subsequentCallingPoints[0].callingPoint[]` (each has `locationName`, `crs`, `st`, `et`).

Delays: `delayedTrains[]`, `totalTrainsDelayed`, `delays` boolean.

## Error Handling

- API unreachable → fall back to baseline + warn
- No services → suggest `/trains disruptions`
- Bad station name → suggest CRS code

## Gotchas

- **Rendering in Bash is the #1 failure mode.** Despite the CRITICAL section above, Claude still defaults to rendering the departures board inside a Bash tool call. The board then collapses behind "click to expand" and defeats the purpose. Always: fetch data with Bash, render the board as markdown text in your response.
- **The API proxy can go down without warning.** The davwheat Huxley2 proxy is a community project, not an official National Rail service. When it's down, fall back to timetable cache — but if the cache is empty (never refreshed), there's nothing to show. Suggest `/trains refresh` when the cache is empty and the API is healthy.
- **Direction auto-detect is crude.** Before noon = home→work, after noon = work→home. This is wrong on WFH days, weekends, or unusual schedules. If the direction seems wrong for context, ask rather than assuming.
- **Fantastical MCP is the preferred calendar method when available.** Check for `mcp__Fantastical__createCalendarItem` in the session's available tools — if present, use it. First call `queryCalendars` to find a dedicated trains/commute calendar; fall back to default calendar if none exists. Fall back to .ics export only when the MCP is not connected. The official binary lives at `~/Library/Application Support/Claude/Claude Extensions/ant.dir.gh.flexibits.fantastical-mcp/`.
- **Fantastical is preferred over Apple Calendar when installed.** Always check for `/Applications/Fantastical.app` before opening .ics files. Use `open -a Fantastical` — do not rely on default file association, which may still point to Apple Calendar even when Fantastical is installed.
- **The .ics export lacks VTIMEZONE.** Generally fine for UK local time, but could produce wrong calendar entries around DST transitions (last Sunday of March/October). If adding a train near a clock change, double-check the times.

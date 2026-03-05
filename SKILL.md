---
name: trains
description: Check UK train commute times, disruptions, and add trains to Apple Calendar. Uses the Huxley2 API (National Rail Darwin proxy). Supports live departures, delay checking, baseline timetable caching, and .ics calendar export. Trigger on '/trains', or when user mentions train times, commute, or checking departures. Examples - '/trains' shows next departures, '/trains disruptions' checks for delays, '/trains add 08:15' adds a train to calendar, '/trains setup' configures stations.
---

# trAIns — Commute Tracker

Check live UK train departures, disruptions, and add trAIns to Apple Calendar.

## Config

- **Config file**: `~/.claude/trains/config.json`
- **Timetable cache**: `~/.claude/trains/timetable.json`

Read config on every invocation. If missing, run setup flow.

```json
{
  "home": {"name": "Leighton Buzzard", "crs": "LBZ"},
  "work": {"name": "London Euston", "crs": "EUS"},
  "theme": "board",
  "sort": "depart",
  "filter": null
}
```

| Config key | Values | Default | Purpose |
|------------|--------|---------|---------|
| `theme` | `board`, `clean` | `board` | Display theme |
| `sort` | `depart`, `arrive` | `depart` | Sort trains by departure or arrival time |
| `filter` | `null`, `fast`, `semi`, `stopping` | `null` | Only show trains of this type |

Pass as CLI flags too: `--sort arrive`, `--fast`, `--semi`, `--stopping`

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

## Modes

### `/trains` or `/trains next`

1. Read config
2. Direction: before 12:00 → home-to-work, after → work-to-home
   - Override: `/trains to work`, `/trains to home`
3. If theme is `board`: open an animated departures board in a Terminal.app window using:
   ```bash
   osascript -e 'tell application "Terminal" to do script "curl -s \"https://national-rail-api.davwheat.dev/departures/{from}/to/{to}?expand=true\" | python3 ~/.claude/skills/trains/scripts/departures.py {DEST_CRS} --theme board --animate; echo; echo \"Press any key to close\"; read -n1; exit"'
   ```
   This pops up a real terminal with the split-flap animation. The window stays open so the user can read it while continuing to work in Claude Code.
4. Also render a compact inline summary in the conversation (markdown text, not the full board) so the user has the data in context too. Example:
   ```
   Trains: Euston → Leighton Buzzard (opened in Terminal)
   Next: 14:56 (⚡ fast, plat 12) · 15:09 (all stations) · 15:23 (via Watford Jn)
   ● Good service
   ```
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

### `/trains add {time}` or `/trains add next`

1. Match train by departure time (or first upcoming)
2. Write `.ics` to `/tmp/train-{date}-{time}.ics`:
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
3. `open /tmp/train-{date}-{time}.ics` — imports to Apple Calendar

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

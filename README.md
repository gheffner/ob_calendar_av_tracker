# Onboarding Calendar Availability Tracker

A drill-down dashboard for DoorLoop's **onboarding** Calendly availability — the seven
sales-handoff round-robins (by unit band) and every onboarding specialist's twelve managed
event types. Sibling of `sales_rep_av_tracker`; same deployment shape, same design tokens.

Live data is produced by the n8n workflow **`ob_calendly_next_availability_v2`**, which writes a
snapshot into `capacity.ob_calendar_rep_daily` and `capacity.ob_calendar_segment_daily` with
`calendar_set = 'ob_2026_07'`.

## The three levels

| Route | View | Shows |
|---|---|---|
| `#/` | Overview | KPI tiles · 7 round-robin unit-tier cards · specialist segment cards · trend chart · all-segments table |
| `#/tier/{unit_tier}` | Round-robin detail | That calendar's numbers and its last 28 runs |
| `#/segment/{rep_segment}` | **Manager view** | Every specialist in the segment, worst availability first, plus an event-type breakdown |
| `#/rep/{name}` | Specialist detail | That person's 12 calendars, each with next slot / 24h / 48h / status |

Routing is hash-based, so any drill-down state is a shareable link and the back button works.
Cards and table rows are clickable; a `›` chevron marks anything that drills.

The overview intentionally mirrors the sales tracker — four KPI tiles, then status-coded cards
with a next-slot line, three metrics and a 14-day sparkline, then a single-series trend chart with
a metric toggle, then a full table.

## Why specialist slots aren't summed

A specialist's twelve calendars all draw on **one underlying availability**. Adding their slot
counts together would overstate real capacity several times over. So:

- **Round-robin tiers** are one calendar each — slot counts are shown directly and do add up.
- **Specialist segments** report coverage and timing instead: specialists free today, event types
  with an open slot, average days out.

An on-page note states this so nobody reads the two card types as the same unit.

## What it shows

| Section | Meaning |
|---|---|
| KPI tiles | unit tiers available today, soonest slot anywhere, specialists available today, segments over the SLA threshold |
| Round-robin cards | per unit band: next slot, open slots in 24h / 48h, days out, 14-day sparkline |
| Specialist cards | per rep segment: specialists free today, event types open, avg days out, 14-day sparkline |
| Trend chart | one metric across the round-robins, per run, last 14 days |
| Tables | every segment / specialist / event type at the latest run |

Health / SLA threshold = next slot within **1 day**, matching the sales tracker and the RevOps
threshold alert. "Days to slot" is computed in the browser from `next_available_at` against
**Eastern** calendar days, so "today" always means today in ET.

## How the SQL proxy works

The dashboard never talks to AWS directly. It POSTs `{sql, params}` to `/api/sql` on its own
origin; nginx forwards to the AWS API Gateway and adds the `X-Identity` and `X-Internal-Secret`
headers from container env vars. **The credential never reaches the browser.** Numeric columns
come back as strings — the page wraps them in `Number()`.

## Deploy (deploybay)

Ships a `Dockerfile` that builds an nginx image serving `index.html` on port 80. Set these env
vars in deploybay (as a single bundled env block, then trigger a deploy):

| Env var | Value |
|---|---|
| `SQL_IDENTITY` | `gheffner` — sent as `X-Identity` |
| `SQL_SECRET` | the `X-Internal-Secret` for the AWS SQL proxy |

Verify injection from an authenticated browser: `fetch('/debug-env.json').then(r=>r.json()).then(console.log)`.

## Local development

```sh
python3 _local_preview.py   # serves on :8789 and proxies /api/sql with headers
# open http://localhost:8789/
```

`config.js` and `_local_preview.py` are gitignored (they carry the same-origin config / the secret
for local use). Note the port differs from the sales tracker's 8788 so both can run at once.

## Design notes

One data hue (`#2a78d6`) and **every chart is a single series**, so there is no categorical
palette and no legend — the panel title names the series. The status palette
(`#0ca30c` / `#fab219` / `#d03b3b`) is reserved for state and always ships with an icon **and** a
text label, never color alone; amber badge text uses the darkened `#9a6a00` for contrast. Every
view pairs its chart with a table.

## Status

Not yet verified against real data — both tables stay empty until
`ob_calendly_next_availability_v2` runs for the first time. The queries have been executed against
the live tables to confirm they parse and return the expected shape, but the page has not been
seen rendering live rows.

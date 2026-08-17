---
name: cheapest-dates
description: Find the cheapest date to fly a route, and the cheapest trip length, by scanning a whole departure date range of real-time Google Flights fares rather than pricing one fixed date, then judging the winner against Google's historical price_insights_low and price_insights_high band and the low|typical|high price_range_in_relation_to_other_periods verdict. Use when dates are flexible or not yet chosen: cheap flights to Rome in May, cheapest day, week, weekend, or month to fly, cheapest time of year, off-season or shoulder season, when should I go to Tokyo, lowest fare between two dates, cheapest departure date JFK to LIS, price calendar, fare calendar, date grid, flexible date search, would leaving a day earlier or moving the trip a week be cheaper, or cheapest 5, 6, or 7 nights in Lisbon. Returns the cheapest fare per date across the window plus the winning departure_date and return_date, price_as_number or total_price_as_number, the airline, stop count and duration for each leg, buy_link, and the low/typical/high deal verdict with its price band, all fetched live. For one fixed date tracked over time or a book-now-or-wait alert use fare-watch; for flights plus hotel totalled into one trip budget use trip-planner.
---

# Cheapest Dates

## Overview

Answers "when is it cheapest to fly X to Y" — and, for round trips, "how long a stay is cheapest" — by pricing a range of departure dates, and a set of trip lengths in nights, then judging the winner against Google's own historical band for that route and period. Use `travel-data-api` for auth, base URL, and endpoint details, and prefer the hosted MCP tools when a client is configured, because the MCP tools take a date range in a single call while the REST path costs one billed request per date. Every result item carries `price_range_in_relation_to_other_periods` (`low` | `typical` | `high`), `price_insights_low`, `price_insights_high`, `departure_date`, and `buy_link` — the verdict field is the differentiator and it leads the answer. The remaining fields differ by trip type: one-way items carry `price_as_number`, `airline`, `stops`, and `duration_seconds`, while round-trip items carry `total_price_as_number`, `total_stops`, `total_duration_seconds`, `return_date`, and per-leg fields such as `departure_flight_airline`, `departure_flight_stops`, `departure_flight_duration`, and their `return_flight_*` counterparts. Do not read `airline`, `stops`, or `duration_seconds` off a round-trip item; those keys are not there.

## Endpoints and Tools

| Job | REST endpoint (one call per date) | MCP tool (one call per range) |
|---|---|---|
| Cheapest one-way departure date | `POST /api/google_flights/oneway/v1` | `search_oneway_flights` |
| Cheapest round-trip departure date / trip length | `POST /api/google_flights/roundtrip/v1` | `search_roundtrip_flights` |

REST host: `google-flights-live-api.p.rapidapi.com`. MCP servers: `https://google-flights-mcp.flightpowers.com/mcp` (paid, ad-free, caller's own RapidAPI key, fan-out cap 30 per call and 60 max, reports `api_usage` in every response) and `https://google-flights-lulu.flightpowers.com/mcp` (free, ad-supported, no key, fan-out cap 15 per call).

## Workflow

1. Pin down origin IATA, destination IATA, the date window, and whether the trip is one-way or round-trip. For round-trip, get trip length in nights — that is what makes the MCP range search collapse to one call.
2. Check whether an MCP client is configured. If yes, issue **one** `search_oneway_flights` or `search_roundtrip_flights` call with `departure_date_from` and `departure_date_to` covering the whole window (and `nights` for round-trip, a number or a list like `[5,6,7]`). Never loop the MCP tools one date at a time.
3. If only REST is available, state the cost before calling: a 14-day one-way scan is 14 billed requests, and a round-trip scan is one request per departure/return **combination**, which multiplies fast. Get the user's go-ahead, or narrow the window first.
4. Collect the lowest `price_as_number` / `total_price_as_number` per date, sort client-side, and note the date and airline of the minimum.
5. Read `price_range_in_relation_to_other_periods` and the `price_insights_low`/`price_insights_high` band from the winning item, then lead the answer with that verdict — the cheapest date found is not automatically a good deal.

## Single Date Probe (REST)

Substitute your own key for `YOUR_RAPIDAPI_KEY`; nothing else needs editing.

```bash
export RAPIDAPI_KEY="YOUR_RAPIDAPI_KEY"

curl -s -X POST 'https://google-flights-live-api.p.rapidapi.com/api/google_flights/oneway/v1' \
  -H "x-rapidapi-key: $RAPIDAPI_KEY" \
  -H 'x-rapidapi-host: google-flights-live-api.p.rapidapi.com' \
  -H 'content-type: application/json' \
  -d '{
    "from_airport": "JFK",
    "to_airport": "LIS",
    "departure_date": "2026-11-03",
    "currency": "USD",
    "limit": 10
  }'
```

## Scanning a Range

**MCP — one call for the whole window.** Call `search_oneway_flights` with the origin, the destination list, `departure_date_from: "2026-11-01"` and `departure_date_to: "2026-11-14"`. For round-trip, call `search_roundtrip_flights` with the same range plus `nights` instead of a fixed `return_date`. The server expands the range internally, so fourteen dates cost one tool call, not fourteen.

**REST — N calls, one per date.** The seven dates in the loop below are seven billed requests; the same fourteen-day window done over REST would be fourteen. Say that number out loud before running it.

```bash
export RAPIDAPI_KEY="YOUR_RAPIDAPI_KEY"

for d in 2026-11-01 2026-11-02 2026-11-03 2026-11-04 2026-11-05 2026-11-06 2026-11-07; do
  printf '=== %s ===\n' "$d"
  curl -s -X POST 'https://google-flights-live-api.p.rapidapi.com/api/google_flights/oneway/v1' \
    -H "x-rapidapi-key: $RAPIDAPI_KEY" \
    -H 'x-rapidapi-host: google-flights-live-api.p.rapidapi.com' \
    -H 'content-type: application/json' \
    -d "{\"from_airport\":\"JFK\",\"to_airport\":\"LIS\",\"departure_date\":\"$d\",\"currency\":\"USD\",\"limit\":5}"
  printf '\n'
done
```

A round-trip scan over REST is one request per departure/return pair. This single pair costs one request; a 7-departure × 3-return grid costs 21.

```bash
export RAPIDAPI_KEY="YOUR_RAPIDAPI_KEY"

curl -s -X POST 'https://google-flights-live-api.p.rapidapi.com/api/google_flights/roundtrip/v1' \
  -H "x-rapidapi-key: $RAPIDAPI_KEY" \
  -H 'x-rapidapi-host: google-flights-live-api.p.rapidapi.com' \
  -H 'content-type: application/json' \
  -d '{
    "from_airport": "JFK",
    "to_airport": "LIS",
    "departure_date": "2026-11-03",
    "return_date": "2026-11-10",
    "currency": "USD",
    "limit": 10
  }'
```

## Common Pitfalls

- **The cheapest date found is not the same as a good deal.** A scan always produces a minimum. If `price_range_in_relation_to_other_periods` on that item is `typical` or `high`, say so plainly — "$209 is typical for this route, don't rush" is the useful answer, and it is the thing competing data sources cannot say.
- **An empty response is `[]` with HTTP 200, not a 404 or an error.** That means no flights on that route and date. Record the date as unavailable, move on, and do not retry it in a loop.
- **`sort_type` is accepted by the one-way schema but is not forwarded** — it is a known defect on `POST /api/google_flights/oneway/v1` and it does work on round-trip. Never rely on it to surface the cheapest one-way itinerary; sort client-side on `price_as_number`. Sorting the range across dates is always client-side work anyway.
- **Hard routes need `use_fallback: true`** (default `false`, slower but better on difficult routes) before you conclude a date genuinely has no flights. Try it on the specific empty dates, not on the whole scan, since each retry is another billed request.
- **Every date, and every departure/return combination, is a separate billed request.** State the request count before scanning, never after. One user intent should be one MCP call whenever an MCP client is configured.
- **Never reuse a fare from earlier in the conversation.** Fares go stale in minutes. If the user asks again, re-run the search and report when the data was fetched; do not cache or carry over a price from a previous scan.

## Output Standards

- Lead with the verdict, then the number: state whether the best fare is `low`, `typical`, or `high` for the route and period, quote the `price_insights_low`–`price_insights_high` band it sits in, then give the winning date, price, airline, stops, and duration, followed by the `buy_link`.
- Show the runner-up dates as a short cheapest-per-date list so the user can see how flat or spiky the window is, always stamp the answer with when the fares were fetched, and note that prices change by the minute and are not a booking or a held fare. This is an independent API that returns publicly available flight and hotel pricing; it is not affiliated with, endorsed by, or sponsored by Google or Booking.com.

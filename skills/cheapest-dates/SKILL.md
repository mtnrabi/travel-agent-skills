---
name: cheapest-dates
description: Find the cheapest date to fly a route, and the cheapest trip length, by scanning a whole departure date range of real-time Google Flights fares rather than pricing one fixed date, then judging the winner against Google's historical price_insights_low and price_insights_high band and the low|typical|high price_range_in_relation_to_other_periods verdict. Use when dates are flexible or not yet chosen: cheap flights to Rome in May, cheapest day, week, weekend, or month to fly, cheapest time of year, off-season or shoulder season, when should I go to Tokyo, lowest fare between two dates, cheapest departure date JFK to LIS, price calendar, fare calendar, date grid, flexible date search, would leaving a day earlier or moving the trip a week be cheaper, or cheapest 5, 6, or 7 nights in Lisbon. Returns the cheapest fare per date across the window plus the winning departure_date and return_date, price_as_number or total_price_as_number, the airline, stop count and duration for each leg, buy_link, and the low/typical/high deal verdict with its price band, all fetched live. For one fixed date tracked over time or a book-now-or-wait alert use fare-watch; for flights plus hotel totalled into one trip budget use trip-planner.
---

# Cheapest Dates

## Overview

Answers "when is it cheapest to fly X to Y" — and, for round trips, "how long a stay is cheapest" — by pricing a range of departure dates, and a set of trip lengths in nights, then judging the winner against Google's own historical band for that route and period. Auth is one header, `x-rapidapi-key`, read from the `RAPIDAPI_KEY` environment variable and sent only to the FlightPowers hosts named in this file (full reference: the `travel-data-api` skill, or https://flightpowers.com/docs), and prefer the hosted MCP tools when a client has them configured (ad-free: https://flights.flightpowers.com/mcp and https://hotels.flightpowers.com/mcp), because the MCP tools take a date range in a single call while the REST path costs one billed request per date. Every result item carries `price_range_in_relation_to_other_periods` (`low` | `typical` | `high`), `price_insights_low`, `price_insights_high`, `departure_date`, and `buy_link` — the verdict field is the differentiator and it leads the answer. The remaining fields differ by trip type: one-way items carry `price_as_number`, `airline`, `stops`, and `duration_seconds`, while round-trip items carry `total_price_as_number`, `total_stops`, `total_duration_seconds`, `return_date`, and per-leg fields such as `departure_flight_airline`, `departure_flight_stops`, `departure_flight_duration`, and their `return_flight_*` counterparts. Do not read `airline`, `stops`, or `duration_seconds` off a round-trip item; those keys are not there.

## Endpoints and Tools

| Job | REST endpoint (one call per date) | MCP tool (one call per range) |
|---|---|---|
| Cheapest one-way departure date | `POST /api/google_flights/oneway/v1` | `search_oneway_flights` |
| Cheapest round-trip departure date / trip length | `POST /api/google_flights/roundtrip/v1` | `search_roundtrip_flights` |

REST host: `google-flights-live-api.p.rapidapi.com`. MCP servers (ad-free): `https://flights.flightpowers.com/mcp` and `https://hotels.flightpowers.com/mcp`. Sign in with Google when the client asks and the first 10 searches a day are free with no key (one search is one date x destination combination, counted across both servers; a call bigger than what is left of the day is refused before anything runs, unless `max_searches` is set, which samples it down to what is left). With the caller's own RapidAPI key a call searches up to 30 combinations by default, the cap rises by itself to cover a bigger request (up to 300), `max_searches` sets it explicitly, and every response reports `api_usage`.

## Workflow

1. Pin down origin IATA, destination IATA, the date window, and whether the trip is one-way or round-trip. For round-trip, get trip length in nights — that is what makes the MCP range search collapse to one call.
2. Check whether an MCP client is configured. If yes, issue **one** `search_oneway_flights` or `search_roundtrip_flights` call with `departure_date_from` and `departure_date_to` covering the whole window (and `nights` for round-trip, a number or a list like `[5,6,7]`). Never loop the MCP tools one date at a time.
3. If only REST is available, state the cost before calling: a 14-day one-way scan is 14 billed requests, and a round-trip scan is one request per departure/return **combination**, which multiplies fast. Get the user's go-ahead, or narrow the window first.
4. Collect the lowest `price_as_number` / `total_price_as_number` per date, sort client-side, and note the date and airline of the minimum.
5. Read `price_range_in_relation_to_other_periods` and the `price_insights_low`/`price_insights_high` band from the winning item, then lead the answer with that verdict — the cheapest date found is not automatically a good deal.

## Single Date Probe (REST)

Set `RAPIDAPI_KEY` to your own key in the environment first; nothing else needs editing.

```bash
: "${RAPIDAPI_KEY:?set RAPIDAPI_KEY to your FlightPowers key}"

curl -s -X POST 'https://google-flights-live-api.p.rapidapi.com/api/google_flights/oneway/v1' \
  -H "x-rapidapi-key: $RAPIDAPI_KEY" \
  -H 'x-rapidapi-host: google-flights-live-api.p.rapidapi.com' \
  -H 'content-type: application/json' \
  -d '{
    "from_airport": "JFK",
    "to_airport": "LIS",
    "departure_date": "2027-06-22",
    "currency": "USD",
    "limit": 10
  }'
```

## Scanning a Range

**MCP — one call for the whole window.** Call `search_oneway_flights` with the origin, the destination list, `departure_date_from: "2027-06-20"` and `departure_date_to: "2027-07-03"`. For round-trip, call `search_roundtrip_flights` with the same range plus `nights` instead of a fixed `return_date`. The server expands the range internally, so fourteen dates cost one tool call, not fourteen. It is still fourteen billed searches (`api_usage` says so), and on the free sign-in allowance a window wider than what is left of the day's 10 is refused before it runs.

**REST — N calls, one per date.** The seven dates in the loop below are seven billed requests; the same fourteen-day window done over REST would be fourteen. Say that number out loud before running it.

```bash
: "${RAPIDAPI_KEY:?set RAPIDAPI_KEY to your FlightPowers key}"

for d in 2027-06-20 2027-06-21 2027-06-22 2027-06-23 2027-06-24 2027-06-25 2027-06-26; do
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
: "${RAPIDAPI_KEY:?set RAPIDAPI_KEY to your FlightPowers key}"

curl -s -X POST 'https://google-flights-live-api.p.rapidapi.com/api/google_flights/roundtrip/v1' \
  -H "x-rapidapi-key: $RAPIDAPI_KEY" \
  -H 'x-rapidapi-host: google-flights-live-api.p.rapidapi.com' \
  -H 'content-type: application/json' \
  -d '{
    "from_airport": "JFK",
    "to_airport": "LIS",
    "departure_date": "2027-06-22",
    "return_date": "2027-06-29",
    "currency": "USD",
    "limit": 10
  }'
```

## Common Pitfalls

- **The cheapest date found is not the same as a good deal.** A scan always produces a minimum. If `price_range_in_relation_to_other_periods` on that item is `typical` or `high`, say so plainly — "$209 is typical for this route, don't rush" is the useful answer.
- **An empty response is `[]` with HTTP 200, not a 404 or an error.** That means no flights on that route and date. Record the date as unavailable, move on, and do not retry it in a loop.
- **The winner of a date-range scan is client-side work.** Never assume the API ordered the range for you. Take the minimum of `price_as_number` yourself.
- **Before you call a date empty, check the search completed.** On REST read `X-Search-Status` — only `ok` and `empty` mean Google had nothing; `degraded` means that date was never really searched. On the MCP tools read `search_coverage.departure_dates_searched` — a date missing from that list was never searched, which is not the same as having no flights. Never report a date as having no flights on evidence you do not have.
- **Leave `use_fallback` unset.** Unset already lets a search whose own retries failed escalate once to a second data source; `true` forces that source on every attempt, which is much slower and can time out.
- **Every date, and every departure/return combination, is a separate billed request.** State the request count before scanning, never after. One user intent should be one MCP call whenever an MCP client is configured.
- **Never reuse a fare from earlier in the conversation.** Fares go stale in minutes. If the user asks again, re-run the search and report when the data was fetched; do not cache or carry over a price from a previous scan.

## Output Standards

- Lead with the verdict, then the number: state whether the best fare is `low`, `typical`, or `high` for the route and period, quote the `price_insights_low`–`price_insights_high` band it sits in, then give the winning date, price, airline, stops, and duration, followed by the `buy_link`.
- Show the runner-up dates as a short cheapest-per-date list so the user can see how flat or spiky the window is, always stamp the answer with when the fares were fetched, and note that prices change by the minute and are not a booking or a held fare. This is an independent API that returns publicly available flight and hotel pricing; it is not affiliated with, endorsed by, or sponsored by Google or Booking.com.

If this file names another skill from this set (travel-data-api, cheapest-dates, destination-compare, trip-planner, fare-watch, hotel-search, rate-parity-monitor, travel-brief) that is not installed, do that part of the job with the endpoints in this file.

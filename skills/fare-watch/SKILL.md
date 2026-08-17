---
name: fare-watch
description: Track the price of one pinned flight over repeated checks - same route, same already-chosen dates - on one-way or round-trip searches, and judge whether the live fare is low, typical, or high against Google's own price_insights_low, price_insights_high, and price_range_in_relation_to_other_periods, alerting only on a meaningful drop. Use when the user asks to watch, track, or monitor a fare, a route, or a flight price, set a fare alert or price alert or price watch, notify or ping or email me when the price drops, tell me if this flight gets cheaper, alert me if JFK to LHR goes under $400, watch BER-CDG for a price drop, monitor fares to Tokyo, check this flight price daily or every morning or weekly or on a cron schedule, is this a good price right now, should I book now or wait, is $209 cheap for this route, has the fare gone up or down since last time, price history or price trend for a route, hit my target price or threshold, or watch one named hotel's nightly rate for a drop over time. For flexible dates or which day or month is cheapest, use cheapest-dates instead. Returns a freshly fetched price_as_number or total_price_as_number with a low/typical/high verdict, Google's historical price band, the airline, the buy_link, a checked_at timestamp, and the delta against the user's own stored check history.
---

# Fare Watch

## Overview

This skill turns a one-off flight search — or the rate for one named hotel on fixed dates — into a repeated check: run the identical search on a schedule, compare the live fare against Google's historical range for that route and period, and decide whether it is low, typical, or high. Use `travel-data-api` for auth, base URLs, and endpoint details. Prefer the hosted MCP tools when a client is configured. The fields this skill decides on are `price_as_number` (or `total_price_as_number` on round-trip), `price_insights_low`, `price_insights_high`, `price_range_in_relation_to_other_periods`, and `buy_link`.

**Fares are never cached.** Every check is a fresh, billed request. A price from an earlier check, an earlier message, or a stored history row is a record of the past, never an answer to "what does it cost now".

## Endpoints

| What you are checking | REST endpoint | MCP tool |
|---|---|---|
| One-way fare on a fixed date | `POST /api/google_flights/oneway/v1` | `search_oneway_flights` |
| Round-trip fare on fixed dates | `POST /api/google_flights/roundtrip/v1` | `search_roundtrip_flights` |
| A window of dates in a single call | one REST call per date | either tool with `departure_date_from` / `departure_date_to` |
| A hotel rate alongside the flight | `POST /hotel_by_name` | n/a |

Flights host: `google-flights-live-api.p.rapidapi.com`. Hotels host: `booking-live-api.p.rapidapi.com`.

## Workflow

1. **Pin the exact search.** Record `from_airport`, `to_airport`, `departure_date` (and `return_date`), `passengers`, `seat_type`, `currency`, and the stop limit — `max_stops` on one-way, `max_departure_stops` / `max_return_stops` on round-trip. Every later check must repeat these values verbatim — a cheaper number produced by a different query is not a drop.
2. **Answer "is this a good price?" with one call.** Read `price_range_in_relation_to_other_periods` (`low` | `typical` | `high`) and the `price_insights_low` / `price_insights_high` range off the cheapest item. This question needs no history.
3. **For "tell me when it drops", re-run the identical call on every scheduled check.** Never reuse the previous response. Tell the user the per-check cost before the watch starts.
4. **Append one row per check** to a history file (shape below), including checks that returned nothing.
5. **Compare against the history** and apply the meaningful-drop test below. Alert on a meaningful drop; stay quiet on noise.
6. **Report with `checked_at` and the `buy_link`**, and state that the fare can move before the user finishes booking.

## Single Check

```bash
export RAPIDAPI_KEY="YOUR_RAPIDAPI_KEY"

curl -sS -X POST 'https://google-flights-live-api.p.rapidapi.com/api/google_flights/oneway/v1' \
  -H "x-rapidapi-key: $RAPIDAPI_KEY" \
  -H 'x-rapidapi-host: google-flights-live-api.p.rapidapi.com' \
  -H 'Content-Type: application/json' \
  -d '{
        "from_airport": "BER",
        "to_airport": "CDG",
        "departure_date": "2026-11-12",
        "currency": "USD",
        "limit": 10
      }'
```

Round-trip is the same call against `/api/google_flights/roundtrip/v1` with `return_date` added. Two things change: there is no `max_stops` on round-trip — it splits into `max_departure_stops` and `max_return_stops` — and the price fields become `total_price` / `total_price_as_number`.

## Repeat Check and Stored History

Run this on your schedule. It picks the cheapest item client-side, appends one history row, and prints the lowest price ever recorded for this exact search (needs `jq`).

```bash
export RAPIDAPI_KEY="YOUR_RAPIDAPI_KEY"
HISTORY="$HOME/.fare-watch/BER-CDG-2026-11-12.jsonl"
mkdir -p "$(dirname "$HISTORY")"

curl -sS -X POST 'https://google-flights-live-api.p.rapidapi.com/api/google_flights/oneway/v1' \
  -H "x-rapidapi-key: $RAPIDAPI_KEY" \
  -H 'x-rapidapi-host: google-flights-live-api.p.rapidapi.com' \
  -H 'Content-Type: application/json' \
  -d '{"from_airport":"BER","to_airport":"CDG","departure_date":"2026-11-12","currency":"USD","limit":10}' \
| jq -c --arg checked_at "$(date -u +%Y-%m-%dT%H:%M:%SZ)" '
    if length == 0 then
      {date: "2026-11-12", price_as_number: null, checked_at: $checked_at, note: "no flights returned"}
    else
      (min_by(.price_as_number)) as $best
      | {date: $best.departure_date,
         price_as_number: $best.price_as_number,
         checked_at: $checked_at,
         verdict: $best.price_range_in_relation_to_other_periods,
         insights_low: $best.price_insights_low,
         insights_high: $best.price_insights_high,
         airline: $best.airline,
         buy_link: $best.buy_link}
    end' | tee -a "$HISTORY"

jq -s 'map(select(.price_as_number != null) | .price_as_number) | min' "$HISTORY"
```

One history row is one line of JSON. The three required fields are:

```json
{"date": "2026-11-12", "price_as_number": 56, "checked_at": "2026-08-16T09:00:00Z"}
```

Everything else (`verdict`, `insights_low`, `insights_high`, `airline`, `buy_link`) is optional but worth storing — the insights range can move between checks, so record it per row rather than assuming it is constant. One file per pinned search; never mix two searches into one file.

**What counts as a meaningful drop.** All three must hold against the same pinned search:

- the new `price_as_number` is at least **10% below the lowest price recorded** in that history file, and
- the absolute difference is at least **25 units of `currency`** (so a $3 wobble on a cheap short-haul does not fire), and
- the new price is **at or below `price_insights_low`**, or `price_range_in_relation_to_other_periods` is `low`.

A price that only moves within `price_insights_low`–`price_insights_high` is normal churn: log it, do not alert. Rises are worth mentioning only when the user asked "should I wait" — a `high` verdict plus an upward trend is the signal to book now.

On the hosted MCP, a whole window is one call: pass `departure_date_from` / `departure_date_to` and a list of destinations rather than looping a date at a time. On `search_roundtrip_flights`, trip lengths go in `nights` (a number, or a list like `[5,6,7]`) instead of a fixed `return_date`.

## Common Pitfalls

- **An empty array is not an error.** `[]` with HTTP 200 means "no flights on this route and date". It is not a price of zero and not a drop. Record the check as a miss with `price_as_number: null` and do not retry in a tight loop — every retry is billed.
- **`sort_type` is silently dropped on one-way searches.** It is accepted by the schema but not forwarded (it does work on round-trip). Never assume the first item is the cheapest on one-way; take the minimum of `price_as_number` client-side, or your "drop" is just result ordering.
- **Hard or thin routes can come back empty on the default path.** Retry once with `use_fallback: true` (slower, better on hard routes) before recording a miss. That retry is a second billed request, so do it once — not on every scheduled check.
- **Every combination is a billed request.** One route × one date × one passenger set × one cabin = one request. A daily watch on 3 routes × 4 dates is 12 requests a day, about 360 a month. State that cost before the watch starts, never after.
- **A sold-out hotel is `available: false`, not a failure.** If the watch also tracks a room via `POST /hotel_by_name`, sold out returns `available: false` with null price fields. Record it as unavailable; it is a valid answer and it is not a price drop.

## Output Standards

- Lead with the verdict and the range, not the bare number: "$209 round-trip, which is **typical** for this route — the historical range for this period is $180–$260. Checked 2026-08-16 14:02 UTC." Then the `buy_link`.
- Print `checked_at` next to every price, and say plainly that the fare was live only at that moment and can change before booking completes. Never answer "what is it now" from the history file or from an earlier message, and never present a fare as held, booked, or guaranteed.
- On an alert, show old price → new price, the percent change, and which stored check it beat. With only one row in the history, say there is no trend yet rather than implying one.

This skill uses an independent API that returns publicly available flight and hotel pricing. It is not affiliated with, endorsed by, or sponsored by Google or Booking.com.

---
name: travel-brief
description: Turn flight and hotel search results into a client-ready travel brief - a one-line recommendation first, then a markdown comparison table, the low|typical|high verdict read from price_range_in_relation_to_other_periods against the price_insights_low/price_insights_high band, the buy_link and hotel booking links, and a prices-checked-at timestamp - re-fetching under one fresh timestamp instead of reformatting fares quoted earlier. Use when the user asks to write this up for a client or customer, make it presentable, client-facing or shareable, format, reformat, polish or tidy these results into something I can send, draft the email, the itinerary summary, a proposal, a quote or a one-pager, put these flights and hotels in a table, summarize the trip options, give me a recommendation not a list, which of these should I book, or refresh these prices before the quote goes out - "write up the BER-CDG options and the Hilton Paris rate for my client". Returns markdown - recommendation, table, verdict with Google's historical price range, booking links, a "prices checked at ... - fares go stale in minutes" line, and a non-affiliation line. Presentation layer over a search it re-runs itself - cheapest date is cheapest-dates, where to go destination-compare, flights plus hotel as one trip budget trip-planner, one fare over time or book-now-or-wait fare-watch. Not affiliated with Google or Booking.com.
---

# Travel Brief

## Overview

This skill is the presentation layer: it re-runs the search itself under one fresh timestamp — rather than reformatting numbers quoted earlier in the conversation — and lays the results out the way a human travel agent would send them to a client: recommendation first, then a comparison table, then the price verdict, then the links, then the timestamp. Use `travel-data-api` for auth, base URL, and endpoint details, and prefer the hosted MCP tools when a client is configured. Both trip types carry `buy_link`, `price_range_in_relation_to_other_periods`, `price_insights_low`, and `price_insights_high`. The rest of the table is built from different keys per trip type: one-way items give `price` / `price_as_number`, `airline`, `stops`, `duration`, `departure_description`, and `arrival_description`; round-trip items give `total_price` / `total_price_as_number`, `total_stops`, `total_duration_seconds`, and per-leg fields such as `departure_flight_airline`, `departure_flight_stops`, `departure_flight_duration`, `departure_flight_departure_description`, and their `return_flight_*` counterparts. Hotel shortlist items from `POST /search` give `name`, `price_string`, `price`, `review_score`, `review_count`, `room_type`, `location`, `image_url`, and `link`; `POST /hotel_by_name` returns those for a single property plus `available` and `nights`.

## Sources for a Brief

| Section of the brief | REST endpoint | MCP tool |
|---|---|---|
| Flight options, one-way | `POST /api/google_flights/oneway/v1` | `search_oneway_flights` |
| Flight options, round-trip | `POST /api/google_flights/roundtrip/v1` | `search_roundtrip_flights` |
| Hotel shortlist at a destination | `POST /search` | n/a |
| One named property's price and availability | `POST /hotel_by_name` | n/a |

Flights host: `google-flights-live-api.p.rapidapi.com`. Hotels host: `booking-live-api.p.rapidapi.com`.

## Workflow

1. **Refetch, never reformat old data.** If the results are more than a few minutes old, or came from an earlier message, run the search again. A brief is a document someone will act on, so it must carry a fresh fetch and the moment it happened.
2. **Stamp the fetch time before you call**, in UTC, and carry that one timestamp through the whole brief.
3. **Pick the recommendation yourself.** Choose one option and say why in one sentence — cheapest, shortest, non-stop, best value against the historical band. A list with no pick is not a brief.
4. **Build the comparison table** from at most 3–5 options, sorted client-side on `price_as_number` or `total_price_as_number`.
5. **Read the verdict** off the recommended item: `price_range_in_relation_to_other_periods` plus the `price_insights_low`–`price_insights_high` band, and put it in its own line.
6. **Close with the links, the checked-at line, and the non-affiliation line.**

## Fetching the Brief's Data

Substitute your own key for `YOUR_RAPIDAPI_KEY`; nothing else needs editing.

```bash
export RAPIDAPI_KEY="YOUR_RAPIDAPI_KEY"
CHECKED_AT="$(date -u +'%Y-%m-%d %H:%M UTC')"

curl -sS -X POST 'https://google-flights-live-api.p.rapidapi.com/api/google_flights/roundtrip/v1' \
  -H "x-rapidapi-key: $RAPIDAPI_KEY" \
  -H 'x-rapidapi-host: google-flights-live-api.p.rapidapi.com' \
  -H 'Content-Type: application/json' \
  -d '{
        "from_airport": "BER",
        "to_airport": "CDG",
        "departure_date": "2026-11-12",
        "return_date": "2026-11-16",
        "currency": "USD",
        "limit": 5
      }'

printf '\nPrices checked at %s — fares go stale in minutes.\n' "$CHECKED_AT"
```

## Refreshing a Brief Before You Send It

Re-run flights and the hotel line together, under one timestamp, right before the brief goes out. Two billed requests — say that before running it.

```bash
export RAPIDAPI_KEY="YOUR_RAPIDAPI_KEY"
CHECKED_AT="$(date -u +'%Y-%m-%d %H:%M UTC')"

curl -sS -X POST 'https://google-flights-live-api.p.rapidapi.com/api/google_flights/roundtrip/v1' \
  -H "x-rapidapi-key: $RAPIDAPI_KEY" \
  -H 'x-rapidapi-host: google-flights-live-api.p.rapidapi.com' \
  -H 'Content-Type: application/json' \
  -d '{"from_airport":"BER","to_airport":"CDG","departure_date":"2026-11-12","return_date":"2026-11-16","currency":"USD","limit":5}'

curl -sS -X POST 'https://booking-live-api.p.rapidapi.com/hotel_by_name' \
  -H "x-rapidapi-key: $RAPIDAPI_KEY" \
  -H 'x-rapidapi-host: booking-live-api.p.rapidapi.com' \
  -H 'Content-Type: application/json' \
  -d '{"hotel_name":"Hilton","area":"Paris","checkin_date":"2026-11-12","checkout_date":"2026-11-16","adults":2,"currency":"USD"}'

printf '\nPrices checked at %s — fares go stale in minutes.\n' "$CHECKED_AT"
```

On the hosted MCP, one user intent is one tool call: pass `departure_date_from` / `departure_date_to` and a list of destinations rather than looping a date at a time.

## Common Pitfalls

- **An empty array is not an error.** The flights endpoints return `[]` with HTTP 200 when nothing is found. Write "no flights returned for this route and date" into the brief as a real finding; do not present it as an outage and do not retry in a loop.
- **A sold-out hotel is `available: false` with null price fields**, which is a valid answer from `POST /hotel_by_name`. Put it in the table as "sold out for these dates" rather than dropping the row silently — the client asked about that property.
- **`sort_type` is accepted on one-way but is not forwarded** (a known defect; it does work on round-trip). Never let API result order become the table order. Sort on `price_as_number` / `total_price_as_number` yourself, or the "cheapest" row in the brief will be wrong.
- **Hard or thin routes need `use_fallback: true`** (default `false`, slower) before you write "no options exist". Try it once on the empty search, not on every option in the brief.
- **Every combination is a billed request.** One route × one date × one cabin × one passenger set = one request, and each hotel in the shortlist is another. Give the request count before rebuilding a brief, never after.
- **Never rebuild a brief from prices quoted earlier in the conversation.** Fares go stale in minutes; a reformatted old number reads as a live quote and is the one failure a client will notice.

## Output Standards

- Fixed order, every time: **recommendation → comparison table → price verdict → booking links → checked-at line → non-affiliation line.** One sentence of recommendation, naming the option and the reason, before any table.
- Flight table columns: Option | Price | Airline | Stops | Duration | Departs / Arrives — filled from `airline` / `stops` / `duration` on one-way items and from the `departure_flight_*` and `return_flight_*` fields on round-trip items, one row per leg or a combined row, never by reading one-way key names off a round-trip item. Hotel table columns: Property | Price for stay | Review score (count) | Room type | Status. 3–5 rows maximum; mark the recommended row.
- The verdict gets its own line and uses the API's own words, not a guess: "$209 round-trip is **typical** for this route and period — Google's historical range here is $180–$260." If the verdict is `high`, say so even when it is the cheapest option found.
- Booking links go in full as `buy_link` (flights) and `link` (hotels). Never describe a link as a held, locked, or guaranteed fare, and never present the brief as a booking.
- Close with the literal line **"Prices checked at YYYY-MM-DD HH:MM UTC — fares go stale in minutes."** using the timestamp from the fetch, followed by: this brief uses an independent API that returns publicly available flight and hotel pricing; it is not affiliated with, endorsed by, or sponsored by Google or Booking.com.

---
name: travel-data-api
description: Reference and routing layer for the independent real-time Google Flights and Booking.com data APIs on RapidAPI, plus the two hosted flights MCP servers and the Apify actors - auth headers, both hosts, every endpoint, every request parameter, exact response field names, empty-result and non-200 handling, MCP tool mapping, fan-out caps, and per-request billing. Use when the user asks about rapidapi google flights, google-flights-live-api, booking-live-api, flightpowers, google-flights-mcp, google-flights-lulu, the Apify flights or booking actor, a flight price, airfare, or flight search API, a hotel price or hotel availability API, curl or client code, the request body or response schema, an API key, x-rapidapi-key or x-rapidapi-host, how to install the MCP server with claude mcp add, why a call returns an empty [] array or a non-200, whether one key covers both APIs, what a call costs, or the parameters sort_type, use_fallback, use_ext_proxy, proxy_country, seat_type, limit, and price_insights. Returns endpoint tables, runnable curl you can port to any language, and the exact JSON fields for one-way, round-trip, hotel search, and hotel_by_name. Plumbing layer - for an actual travel job prefer cheapest-dates, fare-watch, trip-planner, destination-compare, hotel-search, rate-parity-monitor, or travel-brief.
---

# Travel Data API

## Overview

The shared routing and reference layer for two independent real-time travel APIs - a Google Flights data API and a Booking.com data API, both billed through RapidAPI - plus two hosted MCP servers that front the flights API and one Apify actor per API. Every other skill in this repo delegates here for auth, base URLs, parameter names, and response field names. Prefer the MCP tools when a client has them configured; use REST curl otherwise.

These are independent APIs that return publicly available flight and hotel pricing. They are not affiliated with, endorsed by, or sponsored by Google or Booking.com.

## Access Paths

| Path | URL | Key | Prefer it when |
|---|---|---|---|
| Flights REST | `https://google-flights-live-api.p.rapidapi.com` | caller's RapidAPI key | One exact itinerary, full parameter control, or you are writing code for the user |
| Hotels REST | `https://booking-live-api.p.rapidapi.com` | caller's RapidAPI key | Any hotel work - there is no hotels MCP server |
| Flights MCP, paid | `https://google-flights-mcp.flightpowers.com/mcp` | caller's own RapidAPI key | Flexible dates or several candidate destinations; ad-free; reports spend |
| Flights MCP, free | `https://google-flights-lulu.flightpowers.com/mcp` | none needed | Quick look-ups with no key; one disclosed sponsored card per result |
| Apify actor, flights | `https://apify.com/mtnrabi/google-flights-real-time-api` ($5 / 1k) | Apify | No-code or scheduled batch runs |
| Apify actor, hotels | `https://apify.com/mtnrabi/booking-real-time-api` ($4 / 1k) | Apify | No-code or scheduled batch runs |

Rule of thumb: flexible dates or a list of destinations -> MCP, because one user intent is ONE tool call with a range. A single fixed date and route -> REST. Hotels -> always REST.

## Authentication

Both REST hosts use the standard RapidAPI header pair:

```
x-rapidapi-key:  YOUR_RAPIDAPI_KEY
x-rapidapi-host: google-flights-live-api.p.rapidapi.com   # or booking-live-api.p.rapidapi.com
```

The paid MCP server accepts the key three ways: the `x-rapidapi-key` header (preferred), a `?rapidapi_key=` query parameter on the URL, or an API-key field if the client offers one. Install it with:

```bash
claude mcp add --transport http google-flights https://google-flights-mcp.flightpowers.com/mcp --header "x-rapidapi-key: YOUR_RAPIDAPI_KEY"
```

Never print, log, echo, or commit the key. Do not ask the user for a key unless a live call is required and none is configured.

## Request Shape

Every documented endpoint on both hosts is `POST` with a JSON body and `Content-Type: application/json`. All dates are `YYYY-MM-DD`. Flight airports are IATA codes; hotel destinations are free text.

## Endpoint Index

### Flights - host `google-flights-live-api.p.rapidapi.com`

| Endpoint | Required | Returns |
|---|---|---|
| `POST /api/google_flights/oneway/v1` | `departure_date`, `from_airport`, `to_airport` | bare JSON array of one-way itineraries |
| `POST /api/google_flights/roundtrip/v1` | `departure_date`, `return_date`, `from_airport`, `to_airport` | bare JSON array of round-trip itineraries |

One-way optional parameters: `max_stops`, `sort_type` (`Overall` \| `Price` \| `Duration`), `airline_codes[]`, `exclude_airline_codes[]`, `departure_time_min`, `departure_time_max`, `departure_arrival_time_min`, `departure_arrival_time_max`, `currency` (default `USD`), `max_price`, `seat_type` (1 economy, 3 business), `passengers[]` (traveler-type codes: 1 adult, 2 child, 3 infant on lap, 4 infant in seat), `limit` (default 10), `use_fallback` (default false - slower but better on hard routes), `use_ext_proxy` (default true).

Round-trip optional parameters: `max_departure_stops`, `max_return_stops`, `sort_type`, `departure_airline_codes[]`, `return_airline_codes[]`, `departure_exclude_airline_codes[]`, `return_exclude_airline_codes[]`, `departure_departure_time_min/max`, `departure_arrival_time_min/max`, `return_departure_time_min/max`, `return_arrival_time_min/max`, `currency`, `max_price`, `seat_type`, `passengers[]`, `limit`, `use_fallback`, `use_ext_proxy`.

One-way item fields: `price_range_in_relation_to_other_periods`, `price_insights_low`, `price_insights_high`, `from_airport`, `to_airport`, `departure_date`, `price`, `price_as_number`, `duration`, `duration_seconds`, `buy_link`, `airline`, `stops`, `stops_info`, `departure_description`, `arrival_description`.

Round-trip item fields: `price_range_in_relation_to_other_periods`, `price_insights_low`, `price_insights_high`, `from_airport`, `to_airport`, `departure_date`, `return_date`, `total_price`, `total_price_as_number`, `total_duration_seconds`, `total_stops`, `buy_link`, `departure_flight_departure_description`, `departure_flight_arrival_description`, `departure_flight_airline`, `departure_flight_stops`, `departure_flight_duration`, `departure_stops_info`, `return_flight_departure_description`, `return_flight_arrival_description`, `return_flight_airline`, `return_flight_stops`, `return_flight_duration`, `return_stops_info`.

`stops_info`, `departure_stops_info`, and `return_stops_info` are arrays of `{"stop_airport": "AUH", "stop_duration_seconds": 5700}`, empty when the leg is non-stop.

**Lead with price insights.** `price_insights_low` / `price_insights_high` are Google's own historical price range for that route and period, and `price_range_in_relation_to_other_periods` is a `low` \| `typical` \| `high` verdict. That is what lets you say "$209 is typical for this route, no need to rush" instead of only quoting a number. Competing scrapers do not return it.

### Hotels - host `booking-live-api.p.rapidapi.com`

| Endpoint | Required | Returns |
|---|---|---|
| `POST /search` | `destination` (free text), `checkin_date`, `checkout_date` | `{destination, checkin_date, checkout_date, applied_filters, budget_per_night, properties: [...]}` |
| `POST /hotel_by_name` | `hotel_name`, `checkin_date`, `checkout_date` | `{name, available, price_string, price, review_score, review_count, room_type, image_url, link, nights, adults, children}` |

`/search` optional: `adults` (default 2), `children` (default 0), `currency` (default `USD`), `budget_per_night`, `proxy_country` (two-letter code), `filters[]`. Each entry in `properties` has `name`, `price_string`, `price`, `review_score`, `review_count`, `room_type`, `location`, `image_url`, `link`.

`/hotel_by_name` optional: `area` (disambiguates generic names), `adults`, `children`, `currency`, `proxy_country`, `free_cancellation`.

`filters[]` values: `free_cancellation`, `breakfast_included`, `breakfast_and_lunch`, `breakfast_and_dinner`, `all_meals_included`, `all_inclusive`, `free_wifi`, `swimming_pool`, `gym`, `parking`, `front_desk_24h`, `review_score_7`, `review_score_8`, `review_score_9`, `private_bathroom`, `air_conditioning`, `stars_3`, `stars_4`, `stars_5`, `pets_allowed`, `adults_only`, `sauna`, `very_good_breakfast`, `accepts_online_payment`.

**Lead with `proxy_country`.** It routes the request through a residential proxy in that country, so the same hotel and dates can be priced as a user in `us`, `de`, or `il` would see them. That makes rate-parity and geo-pricing monitoring possible. Nothing else on either marketplace exposes it.

## Runnable curl

```bash
export RAPIDAPI_KEY="YOUR_RAPIDAPI_KEY"

# One-way flights (sort the result yourself - see the sort_type defect below)
curl -sS -X POST "https://google-flights-live-api.p.rapidapi.com/api/google_flights/oneway/v1" \
  -H "x-rapidapi-key: $RAPIDAPI_KEY" \
  -H "x-rapidapi-host: google-flights-live-api.p.rapidapi.com" \
  -H "Content-Type: application/json" \
  -d '{"from_airport":"BER","to_airport":"CDG","departure_date":"2026-09-15",
       "currency":"USD","seat_type":1,"passengers":[1],"max_stops":1,"limit":10}'
```

```bash
# Round-trip flights (sort_type IS honoured here)
curl -sS -X POST "https://google-flights-live-api.p.rapidapi.com/api/google_flights/roundtrip/v1" \
  -H "x-rapidapi-key: $RAPIDAPI_KEY" \
  -H "x-rapidapi-host: google-flights-live-api.p.rapidapi.com" \
  -H "Content-Type: application/json" \
  -d '{"from_airport":"BER","to_airport":"CDG","departure_date":"2026-09-15",
       "return_date":"2026-09-20","sort_type":"Price","currency":"USD","limit":10}'
```

```bash
# Hotels at a destination
curl -sS -X POST "https://booking-live-api.p.rapidapi.com/search" \
  -H "x-rapidapi-key: $RAPIDAPI_KEY" \
  -H "x-rapidapi-host: booking-live-api.p.rapidapi.com" \
  -H "Content-Type: application/json" \
  -d '{"destination":"Paris","checkin_date":"2026-09-15","checkout_date":"2026-09-18",
       "adults":2,"currency":"USD","budget_per_night":250,
       "filters":["free_cancellation","review_score_8"]}'
```

```bash
# One named hotel, priced as a visitor from Germany
curl -sS -X POST "https://booking-live-api.p.rapidapi.com/hotel_by_name" \
  -H "x-rapidapi-key: $RAPIDAPI_KEY" \
  -H "x-rapidapi-host: booking-live-api.p.rapidapi.com" \
  -H "Content-Type: application/json" \
  -d '{"hotel_name":"Hotel Lutetia","area":"Paris","checkin_date":"2026-09-15",
       "checkout_date":"2026-09-18","adults":2,"currency":"USD","proxy_country":"de"}'
```

## Repeat and Fan-Out Variant

There is no bulk flight endpoint on REST. A flexible search is a loop, and **every combination of date, route, and passenger set is one billed request** - 7 dates across 3 destinations is 21 requests. Tell the user the count before you run it.

```bash
export RAPIDAPI_KEY="YOUR_RAPIDAPI_KEY"
for d in 2026-09-15 2026-09-16 2026-09-17; do
  curl -sS -X POST "https://google-flights-live-api.p.rapidapi.com/api/google_flights/oneway/v1" \
    -H "x-rapidapi-key: $RAPIDAPI_KEY" \
    -H "x-rapidapi-host: google-flights-live-api.p.rapidapi.com" \
    -H "Content-Type: application/json" \
    -d "{\"from_airport\":\"BER\",\"to_airport\":\"CDG\",\"departure_date\":\"$d\",\"limit\":5}"
done
```

If an MCP server is available, do not write that loop. Both MCP tools take a date RANGE (`departure_date_from` / `departure_date_to`), a LIST of destinations, and - for round-trip - a `nights` value (a number or a list such as `[5,6,7]`) instead of a fixed `return_date`. One user intent is ONE tool call, never one call per date.

## MCP Tool Mapping

| Job | MCP tool | REST equivalent |
|---|---|---|
| One-way search | `search_oneway_flights` | `POST /api/google_flights/oneway/v1` |
| Round-trip search | `search_roundtrip_flights` | `POST /api/google_flights/roundtrip/v1` |
| Anything hotels | none - no hotels MCP server exists | `POST /search`, `POST /hotel_by_name` |

|  | Paid `google-flights-mcp.flightpowers.com/mcp` | Free `google-flights-lulu.flightpowers.com/mcp` |
|---|---|---|
| Ads | none | one disclosed sponsored card per result |
| Key | caller's own RapidAPI key | none needed |
| Fan-out cap | 30 per call (60 max) | 15 per call |
| Spend reporting | `api_usage` in every response | n/a |
| Directory-listable | yes | no - ads are banned by Anthropic, OpenAI, and the official MCP registry |

When the paid server returns `api_usage`, surface it to the user rather than hiding it.

## Errors and Non-Answers

- `[]` with HTTP 200 is the "no flights on this route and date" answer. It is not an error and not a 404. Report it as an answer.
- `available: false` with null price fields from `/hotel_by_name` means sold out for those dates. Also a valid answer.
- There is no documented per-endpoint error-code table. On any non-200, report the status and body verbatim, check that the key is subscribed to that specific API on RapidAPI, and stop - do not retry in a loop, because retries are billed.
- On a hard or thin route that returns nothing, one retry with `use_fallback: true` is justified. It is slower. Say you are spending a second request before you spend it.

## Hard Rules

1. **Never cache or reuse fares.** Fares go stale in minutes. Never reuse a price from earlier in the conversation; re-query and state when the data was fetched.
2. **Say what a call costs before making it**, never after. Every billed request is the user's money.
3. Never present a fare or room rate as bookable. Hand over `buy_link` / `link` and let the user confirm on the live page.
4. Never invent prices, uptime, latency, user counts, revenue, or customer names. If it is not in a response or on this page, do not say it.
5. Never print, log, or commit the API key. Use `YOUR_RAPIDAPI_KEY` in anything written to a file.
6. State the non-affiliation plainly wherever a reader could assume otherwise.
7. One user intent is one MCP call with a range - never one call per date.

## Common Pitfalls

- **An empty array is not an error.** Do not re-run the identical call, do not silently widen the search, and do not apologise for a bug. The one exception is a single announced `use_fallback: true` retry on a plausible route (see above), which is a second billed request. Say "no results for that route on that date" and offer nearby dates as a new, separately billed search.
- **`sort_type` is silently dropped on one-way searches.** It is accepted by the schema but not forwarded (`api_lambda.py` builds `OneWayInput` without it); it does work on round-trip. Sort one-way results client-side and never tell the user the API sorted them.
- **A sold-out hotel returns `available: false`, not an error.** Report it as sold out for those dates and offer alternatives, rather than re-calling the endpoint.
- **Hard routes need `use_fallback: true`.** It defaults to false and is slower; reach for it once when a plausible route comes back empty, not on every call.
- **Every combination is a billed request.** Date ranges, destination lists, and passenger variants multiply. Count the calls, quote the count, and use the MCP range parameters when they are available.
- **Two separate RapidAPI subscriptions.** A key subscribed to the flights API is not automatically subscribed to the hotels API, and the `x-rapidapi-host` header must match the host being called.

## Output Standards

- Lead with the answer to the user's question - the fare, the verdict, the availability - then the supporting detail. Pair every price with `price_range_in_relation_to_other_periods` and the insights range when present, so the number carries a low / typical / high judgement.
- Always state when the data was fetched and include the `buy_link` or hotel `link`. Present prices in the currency that was requested, and name any filter or cap you applied so the user knows what was excluded.

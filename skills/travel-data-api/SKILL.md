---
name: travel-data-api
description: Reference and routing layer for the independent real-time Google Flights and Booking.com data APIs on RapidAPI, plus the hosted MCP servers for flights and hotels and the Apify actors - auth headers, both hosts, every endpoint, every request parameter, exact response field names, empty-result and non-200 handling, MCP tool mapping, fan-out caps, and per-request billing. Use when the user asks about rapidapi google flights, google-flights-live-api, booking-live-api, flightpowers, google-flights-mcp, flights.flightpowers.com, hotels.flightpowers.com, the Apify flights or booking actor, a flight price, airfare, or flight search API, a hotel price or hotel availability API, curl or client code, the request body or response schema, an API key, x-rapidapi-key or x-rapidapi-host, how to install the MCP server with claude mcp add, the free sign-in allowance, why a call returns an empty [] array or a non-200, whether one key covers both APIs, what a call costs, what an X-Search-Status of degraded means, or the parameters sort_type, use_fallback, strict, use_ext_proxy, proxy_country, price_as_seen_from, seat_type, limit, and price_insights. Returns endpoint tables, runnable curl you can port to any language, and the exact JSON fields for one-way, round-trip, hotel search, and hotel_by_name. Plumbing layer - for an actual travel job prefer cheapest-dates, fare-watch, trip-planner, destination-compare, hotel-search, rate-parity-monitor, or travel-brief.
---

# Travel Data API

## Overview

The shared routing and reference layer for two independent real-time travel APIs - a Google Flights data API and a Booking.com data API, both billed through RapidAPI - plus two hosted, ad-free MCP servers - one for flights, one for hotels - and one Apify actor per API. Every other skill in this repo delegates here for auth, base URLs, parameter names, and response field names. Prefer the MCP tools when a client has them configured; use REST curl otherwise.

These are independent APIs that return publicly available flight and hotel pricing. They are not affiliated with, endorsed by, or sponsored by Google or Booking.com.

## Access Paths

| Path | URL | Key | Prefer it when |
|---|---|---|---|
| Flights REST | `https://google-flights-live-api.p.rapidapi.com` | caller's RapidAPI key | One exact itinerary, full parameter control, or you are writing code for the user |
| Hotels REST | `https://booking-live-api.p.rapidapi.com` | caller's RapidAPI key | Full parameter control, `proxy_country`, and the 24 `filters[]` values |
| Flights MCP, ad-free | `https://flights.flightpowers.com/mcp` | Google sign-in (10 free searches a day), then the caller's own RapidAPI key | Flexible dates or several candidate destinations; ad-free; reports spend. Also answers on `google-flights-mcp.flightpowers.com/mcp` |
| Hotels MCP, ad-free | `https://hotels.flightpowers.com/mcp` | Google sign-in (the same 10 a day), then the caller's own RapidAPI key | Hotel work from an MCP client; adds `price_as_seen_from` and `filters` |
| Apify actor, flights | `https://apify.com/mtnrabi/google-flights-real-time-api` ($3.00 per 1,000 flight lookups) | Apify | No-code or scheduled batch runs |
| Apify actor, hotels | `https://apify.com/mtnrabi/booking-real-time-api` ($3.70 per 1,000 hotel lookups) | Apify | No-code or scheduled batch runs |

Rule of thumb: flexible dates or a list of destinations -> MCP, because one user intent is ONE tool call with a range. A single fixed date and route -> REST. Hotels -> the hotels MCP server from an MCP client, REST otherwise; `proxy_country` and the 24 `filters[]` values are on both (`proxy_country` is `price_as_seen_from` on MCP).

## Authentication

Both REST hosts use the standard RapidAPI header pair:

```
x-rapidapi-key:  YOUR_RAPIDAPI_KEY
x-rapidapi-host: google-flights-live-api.p.rapidapi.com   # or booking-live-api.p.rapidapi.com
```

Both MCP servers take the same URL either way in:

```bash
claude mcp add --transport http flights https://flights.flightpowers.com/mcp
claude mcp add --transport http hotels https://hotels.flightpowers.com/mcp
```

In a client that supports MCP authorization (Claude, Claude Code, Cursor, ChatGPT and others), a request with no credential is answered with a sign-in: the user signs in with Google and gets **10 free searches a day with no key**, counted per Google account across both servers, renewing at 00:00 UTC. A search is one date x destination combination, so a call with a three-day range spends three; a call bigger than what is left of the day is refused before anything runs, and past the allowance the tools answer `search_status: "trial_exhausted"` with no results (not an error, and retrying does not help). To go past it, the user pastes their RapidAPI key once on the `/connect` page after signing in (`https://flights.flightpowers.com/connect`, `https://hotels.flightpowers.com/connect`); the key is then used on every call and nothing goes in the client config.

A client without MCP authorization sends the key itself: the `x-rapidapi-key` header (preferred), a `?rapidapi_key=` query parameter on the URL, or an API-key field if the client offers one:

```bash
claude mcp add --transport http flights https://flights.flightpowers.com/mcp --header "x-rapidapi-key: YOUR_RAPIDAPI_KEY"
```

A key on the request always wins over a stored one. With no key and no allowance left, the tools return `needs_api_key: true` with the signup URL and spend nothing.

Never print, log, echo, or commit the key. Do not ask the user for a key unless a live call is required and none is configured.

## Request Shape

Every documented endpoint on both hosts is `POST` with a JSON body and `Content-Type: application/json`. All dates are `YYYY-MM-DD`. Flight airports are IATA codes; hotel destinations are free text.

## Endpoint Index

### Flights - host `google-flights-live-api.p.rapidapi.com`

| Endpoint | Required | Returns |
|---|---|---|
| `POST /api/google_flights/oneway/v1` | `departure_date`, `from_airport`, `to_airport` | bare JSON array of one-way itineraries |
| `POST /api/google_flights/roundtrip/v1` | `departure_date`, `return_date`, `from_airport`, `to_airport` | bare JSON array of round-trip itineraries |

One-way optional parameters: `max_stops`, `sort_type` (`Overall` \| `Price` \| `Duration`), `airline_codes[]`, `exclude_airline_codes[]`, `departure_time_min`, `departure_time_max`, `departure_arrival_time_min`, `departure_arrival_time_max`, `currency` (default `USD`), `max_price`, `seat_type` (1 economy, 3 business), `passengers[]` (traveler-type codes: 1 adult, 2 child, 3 infant on lap, 4 infant in seat), `limit` (default 10), `use_fallback` (default unset; see Common Pitfalls), `strict` (default false - return HTTP 503 instead of a misleading `[]` when the search did not complete), `use_ext_proxy` (default true).

Round-trip optional parameters: `max_departure_stops`, `max_return_stops`, `sort_type`, `departure_airline_codes[]`, `return_airline_codes[]`, `departure_exclude_airline_codes[]`, `return_exclude_airline_codes[]`, `departure_departure_time_min/max`, `departure_arrival_time_min/max`, `return_departure_time_min/max`, `return_arrival_time_min/max`, `currency`, `max_price`, `seat_type`, `passengers[]`, `limit`, `use_fallback` (see Common Pitfalls), `strict`, `use_ext_proxy`.

One-way item fields: `price_range_in_relation_to_other_periods`, `price_insights_low`, `price_insights_high`, `from_airport`, `to_airport`, `departure_date`, `price`, `price_as_number`, `duration`, `duration_seconds`, `buy_link`, `airline`, `stops`, `stops_info`, `departure_description`, `arrival_description`.

Round-trip item fields: `price_range_in_relation_to_other_periods`, `price_insights_low`, `price_insights_high`, `from_airport`, `to_airport`, `departure_date`, `return_date`, `total_price`, `total_price_as_number`, `total_duration_seconds`, `total_stops`, `buy_link`, `departure_flight_departure_description`, `departure_flight_arrival_description`, `departure_flight_airline`, `departure_flight_stops`, `departure_flight_duration`, `departure_stops_info`, `return_flight_departure_description`, `return_flight_arrival_description`, `return_flight_airline`, `return_flight_stops`, `return_flight_duration`, `return_stops_info`.

`stops_info`, `departure_stops_info`, and `return_stops_info` are arrays of `{"stop_airport": "AUH", "stop_duration_seconds": 5700}`, empty when the leg is non-stop.

**Lead with price insights.** `price_insights_low` / `price_insights_high` are Google's own historical price range for that route and period, and `price_range_in_relation_to_other_periods` is a `low` \| `typical` \| `high` verdict. That is what lets you say "$209 is typical for this route, no need to rush" instead of only quoting a number.

### Hotels - host `booking-live-api.p.rapidapi.com`

| Endpoint | Required | Returns |
|---|---|---|
| `POST /search` | `destination` (free text), `checkin_date`, `checkout_date` | `{destination, checkin_date, checkout_date, applied_filters, budget_per_night, properties: [...]}` |
| `POST /hotel_by_name` | `hotel_name`, `checkin_date`, `checkout_date` | `{name, available, price_string, price, review_score, review_count, room_type, image_url, link, nights, adults, children}` |

`/search` optional: `adults` (default 2), `children` (default 0), `currency` (default `USD`), `budget_per_night`, `proxy_country` (two-letter code), `filters[]`. Each entry in `properties` has `name`, `price_string`, `price`, `review_score`, `review_count`, `room_type`, `location`, `image_url`, `link`.

`/hotel_by_name` optional: `area` (disambiguates generic names), `adults`, `children`, `currency`, `proxy_country`, `free_cancellation`.

`filters[]` values: `free_cancellation`, `breakfast_included`, `breakfast_and_lunch`, `breakfast_and_dinner`, `all_meals_included`, `all_inclusive`, `free_wifi`, `swimming_pool`, `gym`, `parking`, `front_desk_24h`, `review_score_7`, `review_score_8`, `review_score_9`, `private_bathroom`, `air_conditioning`, `stars_3`, `stars_4`, `stars_5`, `pets_allowed`, `adults_only`, `sauna`, `very_good_breakfast`, `accepts_online_payment`.

**Lead with `proxy_country`.** It routes the request through a residential proxy in that country, so the same hotel and dates can be priced as a resident of `de`, `jp`, or `il` would be quoted them. That makes rate-parity and geo-pricing monitoring possible. Two things to keep straight: it is an INPUT parameter only, and the response carries no country field, so record which country you asked for yourself; and rates move between identical calls, so sample each country a few times before calling a gap real. `rate-parity-monitor` has the full procedure.

## Runnable curl

```bash
: "${RAPIDAPI_KEY:?set RAPIDAPI_KEY to your FlightPowers key}"

# One-way flights
curl -sS -X POST "https://google-flights-live-api.p.rapidapi.com/api/google_flights/oneway/v1" \
  -H "x-rapidapi-key: $RAPIDAPI_KEY" \
  -H "x-rapidapi-host: google-flights-live-api.p.rapidapi.com" \
  -H "Content-Type: application/json" \
  -d '{"from_airport":"BER","to_airport":"CDG","departure_date":"2027-06-22",
       "currency":"USD","seat_type":1,"passengers":[1],"max_stops":1,"limit":10}'
```

```bash
# Round-trip flights
curl -sS -X POST "https://google-flights-live-api.p.rapidapi.com/api/google_flights/roundtrip/v1" \
  -H "x-rapidapi-key: $RAPIDAPI_KEY" \
  -H "x-rapidapi-host: google-flights-live-api.p.rapidapi.com" \
  -H "Content-Type: application/json" \
  -d '{"from_airport":"BER","to_airport":"CDG","departure_date":"2027-06-22",
       "return_date":"2027-06-27","sort_type":"Price","currency":"USD","limit":10}'
```

```bash
# Hotels at a destination
curl -sS -X POST "https://booking-live-api.p.rapidapi.com/search" \
  -H "x-rapidapi-key: $RAPIDAPI_KEY" \
  -H "x-rapidapi-host: booking-live-api.p.rapidapi.com" \
  -H "Content-Type: application/json" \
  -d '{"destination":"Paris","checkin_date":"2027-06-22","checkout_date":"2027-06-25",
       "adults":2,"currency":"USD","budget_per_night":250,
       "filters":["free_cancellation","review_score_8"]}'
```

```bash
# One named hotel, priced as a visitor from Germany
curl -sS -X POST "https://booking-live-api.p.rapidapi.com/hotel_by_name" \
  -H "x-rapidapi-key: $RAPIDAPI_KEY" \
  -H "x-rapidapi-host: booking-live-api.p.rapidapi.com" \
  -H "Content-Type: application/json" \
  -d '{"hotel_name":"Hotel Lutetia","area":"Paris","checkin_date":"2027-06-22",
       "checkout_date":"2027-06-25","adults":2,"currency":"USD","proxy_country":"de"}'
```

## Repeat and Fan-Out Variant

There is no bulk flight endpoint on REST. A flexible search is a loop, and **every combination of date, route, and passenger set is one billed request** - 7 dates across 3 destinations is 21 requests. Tell the user the count before you run it.

```bash
: "${RAPIDAPI_KEY:?set RAPIDAPI_KEY to your FlightPowers key}"
for d in 2027-06-22 2027-06-23 2027-06-24; do
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
| Hotel search | `search_hotels` | `POST /search` |
| One named hotel | `find_hotel_by_name` | `POST /hotel_by_name` |

| `flights.` / `hotels.flightpowers.com/mcp` | Google sign-in, no key | Caller's own RapidAPI key |
|---|---|---|
| Ads | none | none |
| Daily budget | 10 searches per Google account per UTC day, across both servers | the caller's own RapidAPI plan |
| Per-call search cap | what is left of the day; a bigger call is refused before it runs | 30 date x destination combinations by default, raised by itself to cover a bigger request (up to 300); `max_searches` sets it explicitly, up to 300. Past the cap the server samples evenly and sets `search_coverage.truncated` |
| Spend reporting | a `trial` block with the count and the cap | `api_usage` in every response |
| Hotels | search and by-name, plus `price_as_seen_from` and `filters` | the same |

When a response carries `api_usage` or a `trial` block, surface it to the user rather than hiding it.

## Errors and Non-Answers

- `[]` with HTTP 200 means "no flights on this route and date" **only when the search completed**. Read the `X-Search-Status` response header first: `ok` and `empty` are answers, `degraded` means the search did not complete and the empty array says nothing about flight availability, and `partial` means the list is real but knowingly short. See "Search Outcome Headers" below.
- `available: false` with null price fields from `/hotel_by_name` means sold out for those dates. Also a valid answer.
- There is no documented per-endpoint error-code table. On any non-200, report the status and body verbatim, check that the key is subscribed to that specific API on RapidAPI, and stop - do not retry in a loop, because retries are billed.
- A `degraded` search is worth one announced retry - that is a second billed request, so say so before spending it. A genuine `empty` is not worth retrying: the answer will not change.

## Search Outcome Headers

Every flights response carries the outcome of the search in headers, so an empty body is
never ambiguous.

| Header | Meaning |
|---|---|
| `X-Search-Status` | `ok` \| `empty` \| `partial` \| `degraded` |
| `X-Search-Reason` | what went wrong on some attempt - `blocked_page`, `unrecognized_page`, `unreadable_prices`, `upstream_timeout`, `upstream_connection`, `parse_error`, `search_truncated`, `upstream_status_<code>` and others. It records the FIRST failure, so it can appear on a response a retry recovered. Read `X-Search-Status`, not this, to decide whether you got an answer |
| `X-Search-Results`, `X-Search-Attempts`, `X-Search-Combinations` | how much work the search did |

- **`ok`** - results returned and the array is complete.
- **`empty`** - the search completed and Google genuinely has nothing for that route and
  date. The empty array IS the answer; retrying is a wasted billed request.
- **`partial`** - real, usable results, knowingly short. Some rows or some combinations
  could not be delivered.
- **`degraded`** - the search did not complete. The empty array says nothing about flight
  availability. Do not report "no flights". Retry once; nothing was booked or changed.

Unreadable pages are retried automatically, and a page that genuinely says "no flights" is
never retried, so a real empty result comes back as fast as it always did. Sending
`"strict": true` turns a `degraded` search into an HTTP 503 with an explanatory body
instead of a misleading `[]`.

The MCP servers read these headers on the caller's behalf: when part of a search did not
complete, the tool result says so in words rather than returning a short list that looks
complete.

## Hard Rules

1. **Never cache or reuse fares.** Fares go stale in minutes. Never reuse a price from earlier in the conversation; re-query and state when the data was fetched.
2. **Say what a call costs before making it**, never after. Every billed request is the user's money.
3. Never present a fare or room rate as bookable. Hand over `buy_link` / `link` and let the user confirm on the live page.
4. Never invent prices, uptime, latency, user counts, revenue, or customer names. If it is not in a response or on this page, do not say it.
5. Never print, log, or commit the API key. Use `YOUR_RAPIDAPI_KEY` in anything written to a file.
6. State the non-affiliation plainly wherever a reader could assume otherwise.
7. One user intent is one MCP call with a range - never one call per date.

## Common Pitfalls

- **An empty array is not automatically "no flights".** Check `X-Search-Status` before you report a route as empty. On `ok`/`empty`, say "no results for that route on that date" and offer nearby dates as a new, separately billed search - do not re-run the identical call. On `degraded`, the search did not complete: never tell the user there are no flights, and retry once, announced.
- **Leave `use_fallback` unset.** Unset already lets a search whose own retries failed escalate once, automatically, to a second, independent flight-data source. `true` forces that source on every attempt, which is much slower and can time out; `false` turns it off, the automatic escalation included. It never made the API "wait longer for Google". Automatic retries and the outcome headers are always on.
- **Never assume the array is sorted the way you asked.** Take the minimum of `price_as_number` / `total_price_as_number` yourself before you call anything the cheapest.
- **A sold-out hotel returns `available: false`, not an error.** Report it as sold out for those dates and offer alternatives, rather than re-calling the endpoint.
- **Every combination is a billed request.** Date ranges, destination lists, and passenger variants multiply. Count the calls, quote the count, and use the MCP range parameters when they are available.
- **Two separate RapidAPI subscriptions.** A key subscribed to the flights API is not automatically subscribed to the hotels API, and the `x-rapidapi-host` header must match the host being called.

## Output Standards

- Lead with the answer to the user's question - the fare, the verdict, the availability - then the supporting detail. Pair every price with `price_range_in_relation_to_other_periods` and the insights range when present, so the number carries a low / typical / high judgement.
- Always state when the data was fetched and include the `buy_link` or hotel `link`. Present prices in the currency that was requested, and name any filter or cap you applied so the user knows what was excluded.

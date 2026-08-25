---
name: trip-planner
description: Plan and price a whole trip for a named destination by combining a real-time round-trip Google Flights search with a Booking.com hotel search over the same dates, aligning checkin_date and checkout_date to the flight dates, deriving the night count, and totalling airfare plus lodging into one all-in per-person budget with filters like free_cancellation and review_score_8. Use when the user asks to plan a trip, vacation, holiday, honeymoon, city break, getaway or long weekend, or asks for flight and hotel together, a flight + hotel package, total trip cost, all-in cost, trip budget, or a per-person cost breakdown — "plan a trip to Rome for a week in May under $2000", "how much would 7 nights in Lisbon cost", "can I do Tokyo for under $3k", "10 days in Japan for a family of four", "JFK to FCO plus a hotel, budget €1500", "long weekend in Barcelona for two, £800 all in", or fit an itinerary inside a price cap in any currency. Returns the aligned checkin_date/checkout_date and night count, round-trip flights with total_price_as_number and buy_link, matching hotels with price, review_score and a booking link, and the combined per-person total against the cap. Flight-only date-range scanning is cheapest-dates, hotels alone hotel-search, picking where to go destination-compare, writing results up for a client travel-brief.
---

# Trip Planner

## Overview

This skill turns a fuzzy request like "a week in Rome in May under $2000" into two live searches — one round-trip flight search and one hotel search on the *same* dates — and one combined number the user can act on. Use `travel-data-api` for auth, base URL, and endpoint details. Prefer the hosted MCP tools when a client is configured — both legs are available over MCP: flights on `flights.flightpowers.com/mcp`, hotels on `hotels.flightpowers.com/mcp`, and both on the free server.

The fields you will actually read: from flights, `total_price_as_number`, `total_price`, `buy_link`, `price_range_in_relation_to_other_periods`, `price_insights_low`, `price_insights_high`, and the `departure_flight_arrival_description` / `return_flight_departure_description` strings that tell you which calendar day each end of the trip lands on. From hotels, `properties[].name`, `price`, `price_string`, `review_score`, `review_count`, `room_type`, `location`, and `link`, plus the echoed `checkin_date`, `checkout_date`, `applied_filters`, and `budget_per_night`.

This is an independent API that returns publicly available flight and hotel pricing; it is not affiliated with, endorsed by, or sponsored by Google or Booking.com.

## Endpoints

| Leg | Endpoint / MCP tool |
|---|---|
| Round-trip flights | `POST /api/google_flights/roundtrip/v1` on `google-flights-live-api.p.rapidapi.com` |
| Round-trip flights (MCP) | `search_roundtrip_flights` — takes `departure_date_from`/`departure_date_to`, a list of destinations, and `nights` |
| One-way legs (open-jaw) | `POST /api/google_flights/oneway/v1` on `google-flights-live-api.p.rapidapi.com` |
| One-way legs (MCP) | `search_oneway_flights` — takes `departure_date_from`/`departure_date_to` and a list of destinations |
| Hotels for the stay | `POST /search` on `booking-live-api.p.rapidapi.com` — REST only, no MCP tool exists |
| One named hotel | `POST /hotel_by_name` on `booking-live-api.p.rapidapi.com` |

## Workflow

1. **Resolve the request into dates and airports before calling anything.** Convert the destination to an IATA airport code (Rome → `FCO`). Turn "a week in May" into a concrete `departure_date` and `return_date`. Ask for the origin airport if the user did not give one — never guess it, a wrong origin makes every number downstream wrong.
2. **Derive nights from the dates, not from the word "week".** `nights = checkout_date − checkin_date` in days. `2027-05-08` → `2027-05-15` is **7 nights** (8 days on the ground). A "week-long trip" is 7 nights only if the user means 7 sleeps; if they said "7 days", confirm whether that is 6 or 7 nights before you price anything.
3. **State the cost before you spend it.** Every search is a billed request against the user's own key. Say "this is 2 calls — one flight search, one hotel search" up front. If you are comparing 3 date windows × 2 cities, that is 6 flight calls plus 6 hotel calls; get agreement first.
4. **Search flights first.** The flight is the larger and least flexible number, and it sets the hotel envelope. Pass `max_price` equal to the per-person flight ceiling you are willing to spend, `currency`, `passengers[]` (1 adult, 2 child, 3 infant on lap, 4 infant in seat), and `sort_type: "Price"`.
5. **Align `checkin_date` / `checkout_date` to the flight, not to the request.** Read `departure_flight_arrival_description` (e.g. `"8:50 AM on Mon, Jun 1"`). On a red-eye or long-haul the outbound lands the day *after* `departure_date` — in that case `checkin_date` is the arrival day, not the departure day. Likewise if `return_flight_departure_description` is an early-morning slot, the traveller still needs the previous night, so `checkout_date` stays the return day. Recompute `nights` after this adjustment.
6. **Compute the hotel envelope, then search hotels.** With a per-person budget `B`, a flight total `F` = `total_price_as_number`, `N` nights and `A` adults sharing one room: `room_budget = (B − F) × A` and `budget_per_night = room_budget ÷ N`. Send that as `budget_per_night`, with `filters: ["free_cancellation", "review_score_8"]` so the shortlist is refundable and well-reviewed.
7. **Combine and present.** Per-person total = `F + (hotel_stay_cost ÷ A)`. Show the gap to budget, the `price_range_in_relation_to_other_periods` verdict against `price_insights_low`/`price_insights_high`, both live links, and the timestamp of the search.

## Search the Round-Trip Flight

```bash
export RAPIDAPI_KEY="YOUR_RAPIDAPI_KEY"

curl -sS -X POST \
  "https://google-flights-live-api.p.rapidapi.com/api/google_flights/roundtrip/v1" \
  -H "x-rapidapi-key: ${RAPIDAPI_KEY}" \
  -H "x-rapidapi-host: google-flights-live-api.p.rapidapi.com" \
  -H "Content-Type: application/json" \
  -d '{
    "from_airport": "JFK",
    "to_airport": "FCO",
    "departure_date": "2027-05-08",
    "return_date": "2027-05-15",
    "passengers": [1, 1],
    "seat_type": 1,
    "currency": "USD",
    "max_price": 900,
    "sort_type": "Price",
    "max_departure_stops": 1,
    "max_return_stops": 1,
    "limit": 10
  }'
```

## Search Hotels for Exactly Those Nights

`checkin_date` and `checkout_date` are the flight dates from the call above. The date span *is* the night count — 2027-05-08 to 2027-05-15 is 7 nights.

```bash
export RAPIDAPI_KEY="YOUR_RAPIDAPI_KEY"

curl -sS -X POST \
  "https://booking-live-api.p.rapidapi.com/search" \
  -H "x-rapidapi-key: ${RAPIDAPI_KEY}" \
  -H "x-rapidapi-host: booking-live-api.p.rapidapi.com" \
  -H "Content-Type: application/json" \
  -d '{
    "destination": "Rome",
    "checkin_date": "2027-05-08",
    "checkout_date": "2027-05-15",
    "adults": 2,
    "children": 0,
    "currency": "USD",
    "budget_per_night": 120,
    "filters": ["free_cancellation", "review_score_8"]
  }'
```

Check the echoed `checkin_date`, `checkout_date` and `applied_filters` in the response body against what you sent before you do any arithmetic on `properties[]`.

## Repeat Variant — Several Windows or Several Cities

There is no bulk trip endpoint. Comparing windows means repeating the pair of calls, and **each iteration is a separate billed request** — quote the total first.

```bash
export RAPIDAPI_KEY="YOUR_RAPIDAPI_KEY"

for WINDOW in "2027-05-01|2027-05-08" "2027-05-08|2027-05-15" "2027-05-15|2027-05-22"; do
  DEP="${WINDOW%%|*}"
  RET="${WINDOW##*|}"
  echo "=== depart ${DEP} / return ${RET} ==="
  curl -sS -X POST \
    "https://google-flights-live-api.p.rapidapi.com/api/google_flights/roundtrip/v1" \
    -H "x-rapidapi-key: ${RAPIDAPI_KEY}" \
    -H "x-rapidapi-host: google-flights-live-api.p.rapidapi.com" \
    -H "Content-Type: application/json" \
    -d "{\"from_airport\":\"JFK\",\"to_airport\":\"FCO\",\"departure_date\":\"${DEP}\",\"return_date\":\"${RET}\",\"passengers\":[1,1],\"currency\":\"USD\",\"sort_type\":\"Price\",\"limit\":5}"
  curl -sS -X POST \
    "https://booking-live-api.p.rapidapi.com/search" \
    -H "x-rapidapi-key: ${RAPIDAPI_KEY}" \
    -H "x-rapidapi-host: booking-live-api.p.rapidapi.com" \
    -H "Content-Type: application/json" \
    -d "{\"destination\":\"Rome\",\"checkin_date\":\"${DEP}\",\"checkout_date\":\"${RET}\",\"adults\":2,\"currency\":\"USD\",\"filters\":[\"free_cancellation\",\"review_score_8\"]}"
done
```

When an MCP client is configured, do the flight half of this in **one** `search_roundtrip_flights` call: pass `departure_date_from: "2027-05-01"`, `departure_date_to: "2027-05-15"` and `nights: 7`. The range bounds the *departure* date, not the return, so the last window in the loop above departs 2027-05-15 and returns 2027-05-22 — setting `departure_date_to` to the return date would silently price an extra window. One user intent is one tool call — never one call per date. The `nights` value you pass there is the same night count you feed to `checkin_date`/`checkout_date` on the hotel call.

## Common Pitfalls

- **An empty flight result is `[]` with HTTP 200, not an error.** It means "no flights on this route for these dates" only when `X-Search-Status` is `ok` or `empty` — then report it plainly and offer adjacent dates or a nearby airport, without retrying the identical call. On `degraded` the search did not complete: do not tell the user the trip is impossible, and do not price a trip around a flight leg you never actually searched.
- **A sold-out hotel is `available: false`, not a failure.** On `/hotel_by_name` a sold-out property returns `available: false` with null price fields — that is a valid, correct answer, so say "not available for those nights" rather than reporting an error. On `/search` the equivalent is an empty `properties` array; loosen `budget_per_night` or drop a filter and say that you did.
- **`sort_type` is honoured on both endpoints.** If you split a trip into two one-ways (open-jaw), you still combine and rank the two halves yourself.
- **`use_fallback` currently does nothing.** It is accepted but selects a second data source that is not switched on for this API. On a long-haul or thin route, an empty result set that is often the search failing rather than genuine unavailability is exactly what `X-Search-Status: degraded` now tells you — retry on that signal, not on `use_fallback`.
- **Every combination is a billed request.** Three date windows across two cities is twelve calls, not one. Price the plan out loud before running it and let the user cut the matrix down.
- **Never reuse or cache a fare.** Fares and room rates go stale in minutes. Do not carry a price forward from earlier in the conversation, from an example, or from a previous window — re-search and stamp the result with the time it was fetched.
- **`/search` returns no `nights` field.** Derive nights from the dates you sent, and read `price_string` before you multiply anything. Do not silently turn a per-night `price` into a stay total; if the basis is ambiguous, quote `price_string` verbatim next to your own arithmetic and label which is which.
- **Keep the passenger set consistent across both legs.** `passengers[]` on the flight call and `adults`/`children` on the hotel call must describe the same travellers, or the per-person total is meaningless. State the split you used (e.g. "2 adults, one shared room").

## Output Standards

- Lead with the verdict against the budget — "7 nights in Rome, 2027-05-08 to 2027-05-15: about $X per person, $Y under your $2000" — then the flight line, the hotel line, and the arithmetic that joins them, so the user can check it. Always show the derived night count and the exact `checkin_date`/`checkout_date` you searched.
- Stamp every total with when it was fetched, quote the `price_range_in_relation_to_other_periods` verdict alongside `price_insights_low`/`price_insights_high` so "$209 is typical for this route, don't rush" is available instead of a bare number, and hand over the live `buy_link` and hotel `link` for booking. Never present a fare or rate as booked or guaranteed — prices are live search results and can change before the user finishes clicking.

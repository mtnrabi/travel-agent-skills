---
name: hotel-search
description: Find and price places to stay with real-time Booking.com data. POST /search takes a free-text destination (city, neighborhood, landmark, airport area), checkin_date, checkout_date, budget_per_night and 24 filters - stars_3/4/5, review_score_7/8/9, free_cancellation, breakfast_included, all_inclusive, swimming_pool, parking, pets_allowed, adults_only. POST /hotel_by_name prices one named property (a Hilton, an Ibis) disambiguated by area. Use when the user asks to find or book a hotel, hostel, resort, apartment or guesthouse, where to stay in Paris or Shibuya, the cheapest hotel or a room under $200 a night, a 4-star with good reviews, somewhere pet-friendly, adults-only or with a pool or gym, accommodation or lodging, or whether a hotel is available or sold out and what a room costs. Returns name, price_string, review_score, room_type, location and link, or available plus nights. Hotel only - flights plus hotel is trip-planner, proxy_country parity is rate-parity-monitor. Not affiliated with Booking.com.
---

# Hotel Search

## Overview

Two endpoints on the hotels REST API answer almost every accommodation question: `POST /search` for "what can I stay in at this destination", and `POST /hotel_by_name` for "is this exact property available and what does it cost". Use `travel-data-api` for auth, base URL, and endpoint details; prefer the hosted MCP tools when a client is configured, but note the hosted MCP servers expose flight search only, so hotel work always runs over REST with the RapidAPI header pair. Response fields you will actually see are `properties[].name`, `price`, `price_string`, `review_score`, `review_count`, `room_type`, `location`, `image_url`, `link` from `/search`, and `name`, `available`, `price`, `price_string`, `review_score`, `review_count`, `room_type`, `image_url`, `link`, `nights`, `adults`, `children` from `/hotel_by_name`.

## Endpoints

| Job the user has | Endpoint | Required | Optional |
|---|---|---|---|
| "Find me a hotel in X" | `POST /search` | `destination` (free text), `checkin_date`, `checkout_date` (YYYY-MM-DD) | `adults` (default 2), `children` (default 0), `currency` (default `USD`), `budget_per_night`, `proxy_country` (two-letter code), `filters[]` |
| "Is the Hotel X available / how much?" | `POST /hotel_by_name` | `hotel_name`, `checkin_date`, `checkout_date` | `area` (disambiguates generic names), `adults`, `children`, `currency`, `proxy_country`, `free_cancellation` |

Host: `booking-live-api.p.rapidapi.com`. Both endpoints are `POST` with a JSON body.

## Workflow

1. Decide which endpoint the question is. A named property ("the Gracery Shinjuku", "Hilton Midtown") is `/hotel_by_name`. Anything shaped like a destination, area, or "somewhere near X" is `/search`.
2. Pin down dates before calling. Both endpoints require `checkin_date` and `checkout_date` in `YYYY-MM-DD`. If the user said "three nights in September", ask or state the dates you assumed — do not guess silently, because a wrong date is a wasted billed request.
3. Translate the user's soft constraints into hard parameters. "Under $200 a night" is `budget_per_night`, "4-star with good reviews and free cancellation" is `filters: ["stars_4", "review_score_8", "free_cancellation"]`. Only send filters the user actually implied; each one narrows the result set.
4. For `/hotel_by_name`, always send `area` when the name is generic ("Hilton", "Ibis Budget", "Grand Hotel"). Without it the API can match a same-named property in another city.
5. Read the result honestly. An empty `properties` array and `available: false` are both valid answers, not errors — see Common Pitfalls. Report the price with the timestamp of the call.

## Search a Destination

```bash
export RAPIDAPI_KEY="YOUR_RAPIDAPI_KEY"

curl -sS -X POST "https://booking-live-api.p.rapidapi.com/search" \
  -H "x-rapidapi-key: $RAPIDAPI_KEY" \
  -H "x-rapidapi-host: booking-live-api.p.rapidapi.com" \
  -H "Content-Type: application/json" \
  -d '{
    "destination": "Tokyo Shibuya",
    "checkin_date": "2026-09-12",
    "checkout_date": "2026-09-15",
    "adults": 2,
    "children": 0,
    "currency": "USD",
    "budget_per_night": 220,
    "filters": ["stars_4", "review_score_8", "free_cancellation"]
  }'
```

The response echoes `destination`, `checkin_date`, `checkout_date`, `applied_filters`, and `budget_per_night` alongside `properties[]` — check `applied_filters` to confirm the constraints you meant to apply were the ones the API used.

## Price a Named Hotel

```bash
export RAPIDAPI_KEY="YOUR_RAPIDAPI_KEY"

curl -sS -X POST "https://booking-live-api.p.rapidapi.com/hotel_by_name" \
  -H "x-rapidapi-key: $RAPIDAPI_KEY" \
  -H "x-rapidapi-host: booking-live-api.p.rapidapi.com" \
  -H "Content-Type: application/json" \
  -d '{
    "hotel_name": "Hotel Gracery Shinjuku",
    "area": "Tokyo",
    "checkin_date": "2026-09-12",
    "checkout_date": "2026-09-15",
    "adults": 2,
    "currency": "USD",
    "free_cancellation": true
  }'
```

`nights` in the response is the length of the stay the quote covers, and `adults` / `children` echo the occupancy that was priced — check them against what the user asked for. Report `price_string` exactly as it comes back rather than deriving a per-night or total figure the response does not state.

## Filters

`filters` is an array of strings on `POST /search`. These are the only accepted values:

| Category | Values |
|---|---|
| Cancellation and payment | `free_cancellation`, `accepts_online_payment` |
| Meals | `breakfast_included`, `breakfast_and_lunch`, `breakfast_and_dinner`, `all_meals_included`, `all_inclusive`, `very_good_breakfast` |
| Star rating | `stars_3`, `stars_4`, `stars_5` |
| Review score | `review_score_7`, `review_score_8`, `review_score_9` |
| Amenities | `free_wifi`, `swimming_pool`, `gym`, `parking`, `front_desk_24h`, `private_bathroom`, `air_conditioning`, `sauna` |
| Property policy | `pets_allowed`, `adults_only` |

Anything not on this list will not work — do not invent filter names such as `spa` or `beachfront`. Note `free_cancellation` is a filter value on `/search` but a standalone optional boolean parameter on `/hotel_by_name`.

## Repeat Searches Across a Shortlist

Neither `POST /search` nor `POST /hotel_by_name` takes a batch parameter, so with these two endpoints a shortlist of named properties is one call per property. Loop them, and tell the user the call count before you start.

```bash
export RAPIDAPI_KEY="YOUR_RAPIDAPI_KEY"

for HOTEL in "Hotel Gracery Shinjuku" "Shibuya Stream Excel Hotel Tokyu" "Cerulean Tower Tokyu Hotel"; do
  curl -sS -X POST "https://booking-live-api.p.rapidapi.com/hotel_by_name" \
    -H "x-rapidapi-key: $RAPIDAPI_KEY" \
    -H "x-rapidapi-host: booking-live-api.p.rapidapi.com" \
    -H "Content-Type: application/json" \
    -d "$(printf '{"hotel_name":"%s","area":"Tokyo","checkin_date":"2026-09-12","checkout_date":"2026-09-15","adults":2}' "$HOTEL")"
done
```

The same shape applies to comparing date windows or currencies: vary one field per call, and count the calls out loud first.

## Common Pitfalls

- **An empty result is an answer, not an error.** `POST /search` returning `properties: []` on HTTP 200 means nothing at that destination matched those dates, that `budget_per_night`, and those `filters`. Do not retry the identical call. Say what came back empty, then offer to drop the tightest constraint (usually `budget_per_night` or a `stars_5` / `review_score_9` filter) and search again — that is a second billed request, so ask first.
- **A sold-out hotel is `available: false` with null prices, and that is valid.** `POST /hotel_by_name` returns `available: false` and null `price` / `price_string` when the property has no rooms for those dates. Report "no availability on those dates" plainly. Do not call it a failure, do not retry, and do not fall back to quoting a price you saw earlier.
- **Every parameter combination is a separate billed request.** Three properties × two date windows is six calls of the user's money. State the cost before making the calls, never after — this applies to `proxy_country` comparisons and currency sweeps too.
- **Never reuse or cache a rate.** Hotel prices and availability go stale in minutes. Every price you present must come from the call you just made, with the time it was fetched attached. If a rate came from earlier in the conversation, re-fetch it rather than repeating it.
- **Two flight-side defects to know if the same trip also needs flights** (hand that work to the flight skills, but do not be surprised by these): `sort_type` is accepted by the schema on the one-way flight endpoint but is not forwarded, so one-way results must be sorted client-side — it does work on round-trip; and `use_fallback` defaults to false, with `use_fallback: true` documented as slower but better on hard routes.

## Output Standards

- Lead with the answer to the question asked — the two or three properties that fit the stated budget and constraints, each with `price_string`, `review_score` out of the `review_count`, `room_type`, and the booking `link` — then offer the full list. Always state when the prices were fetched, and treat every quote as indicative until the user opens the link; a rate is not booked until Booking.com confirms it.
- Say what you filtered on and what you assumed (dates, `adults`, `currency`), so the user can correct a wrong assumption before paying for another search. This is an independent API that returns publicly available flight and hotel pricing; it is not affiliated with, endorsed by, or sponsored by Google or Booking.com — say so whenever a reader could assume the results are an official Booking.com channel.

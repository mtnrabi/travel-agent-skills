---
name: rate-parity-monitor
description: Detect hotel geo-pricing and rate-parity gaps by pricing the same property, the same dates, and the same currency as a resident of different countries would see them, using the proxy_country residential-proxy parameter on real-time Booking.com data, then comparing every market side by side at one moment. Use when the user asks about rate parity or price parity, a parity breach or parity violation, rate-parity monitoring or auditing, geo-pricing, geographic, regional, country-based or location-based pricing, IP-based price discrimination, whether a hotel is cheaper booked from another country or through a VPN or an incognito window, why the same hotel shows a different price from abroad, comparing one hotel across the US, Germany, Israel, the UK or any two-letter country code, OTA undercutting, distribution or rate audits, market-by-market rate shopping, or competitive price monitoring for a hotel, a chain, or a portfolio. Returns a per-market comparison table with proxy_country, available, room_type, price and price_string, nights, a per-night rate normalized to one currency, the spread versus the cheapest market, and a breach-versus-currency-noise verdict with the timestamp of every call. This is many markets at one instant: to track one hotel or one fare over time use fare-watch, and to find or price hotels in a single market use hotel-search.
---

# Rate Parity Monitor

## Overview

`proxy_country` on the hotels REST API routes the request through a residential proxy in that country, so the same hotel and the same dates can be priced exactly as a user in `us`, `de`, or `il` would see them. That one parameter turns a hotel lookup into a rate-parity and geo-pricing audit, which is a commercial job — revenue managers, distribution teams, and competitive-pricing analysts — not a hobbyist one. Use `travel-data-api` for auth, base URL, and endpoint details, and prefer the hosted MCP tools when a client is configured; note that the hosted MCP servers expose flight search only (`search_oneway_flights`, `search_roundtrip_flights`), so parity work always runs over REST on `booking-live-api.p.rapidapi.com` with the RapidAPI header pair. The fields you compare across markets are `available`, `price`, `price_string`, `room_type`, and `nights` from `POST /hotel_by_name`.

## Endpoints

| Job the user has | Endpoint | Required | Parity-relevant optional |
|---|---|---|---|
| "Is this hotel priced differently in other countries?" | `POST /hotel_by_name` | `hotel_name`, `checkin_date`, `checkout_date` (YYYY-MM-DD) | `proxy_country` (two-letter code), `currency`, `area`, `adults`, `children`, `free_cancellation` |
| "Does the whole market look different from abroad?" | `POST /search` | `destination`, `checkin_date`, `checkout_date` | `proxy_country`, `currency` (default `USD`), `adults` (default 2), `children` (default 0), `budget_per_night`, `filters[]` |

Host: `booking-live-api.p.rapidapi.com`. Both are `POST` with a JSON body. `proxy_country` takes a two-letter country code (`us`, `de`, `il`); it is the only parameter that changes which market you are shopping from, and it exists on both endpoints.

## Workflow

1. Fix everything except the country. A parity comparison is only valid if `hotel_name`, `area`, `checkin_date`, `checkout_date`, `adults`, `children`, `free_cancellation`, and `currency` are byte-identical across every call, and only `proxy_country` varies. One drifting field and the result is noise.
2. Pin `currency` to a single value for the whole sweep — normally `USD`. Do not let each market answer in its own currency and then convert; that measures the FX rate, not the hotel's pricing.
3. Choose the market set and state the call count first. Four countries is four billed requests for one hotel and one date window; four countries × three hotels is twelve. Say the number before you spend it.
4. Run the sweep back to back, not spread over the session. Rates move in minutes, so a `us` call from ten minutes ago is not comparable to a `de` call made now — re-run the whole set rather than reusing any earlier figure.
5. Normalize before comparing: divide `price` by `nights` for a per-night rate, and check that `room_type` is the same string in every market. A different `room_type` is a different product, not a price gap.
6. Classify the spread as a breach or as noise using the rules below, then report the table with the fetch timestamp attached to every row.

## Price One Hotel in One Market

```bash
export RAPIDAPI_KEY="YOUR_RAPIDAPI_KEY"

curl -sS -X POST "https://booking-live-api.p.rapidapi.com/hotel_by_name" \
  -H "x-rapidapi-key: $RAPIDAPI_KEY" \
  -H "x-rapidapi-host: booking-live-api.p.rapidapi.com" \
  -H "Content-Type: application/json" \
  -d '{
    "hotel_name": "Hotel Gracery Shinjuku",
    "area": "Tokyo",
    "checkin_date": "2026-11-10",
    "checkout_date": "2026-11-13",
    "adults": 2,
    "children": 0,
    "currency": "USD",
    "proxy_country": "us"
  }'
```

This is the control call. Everything in the sweep below is this exact body with one field changed.

## Sweep a Market Set

There is no multi-country parameter: one country is one call. Loop `proxy_country` over the market set, hold every other field constant, and keep the responses side by side.

```bash
export RAPIDAPI_KEY="YOUR_RAPIDAPI_KEY"

for CC in us de il gb; do
  printf '%s ' "$CC"
  curl -sS -X POST "https://booking-live-api.p.rapidapi.com/hotel_by_name" \
    -H "x-rapidapi-key: $RAPIDAPI_KEY" \
    -H "x-rapidapi-host: booking-live-api.p.rapidapi.com" \
    -H "Content-Type: application/json" \
    -d "$(printf '{"hotel_name":"Hotel Gracery Shinjuku","area":"Tokyo","checkin_date":"2026-11-10","checkout_date":"2026-11-13","adults":2,"children":0,"currency":"USD","proxy_country":"%s"}' "$CC")"
  printf '\n'
done
```

To audit a portfolio, nest a hotel loop outside the country loop — and multiply the call count out loud before running it, because hotels × countries is the whole bill.

```bash
export RAPIDAPI_KEY="YOUR_RAPIDAPI_KEY"

for HOTEL in "Hotel Gracery Shinjuku" "Cerulean Tower Tokyu Hotel"; do
  for CC in us de il; do
    printf '%s | %s ' "$HOTEL" "$CC"
    curl -sS -X POST "https://booking-live-api.p.rapidapi.com/hotel_by_name" \
      -H "x-rapidapi-key: $RAPIDAPI_KEY" \
      -H "x-rapidapi-host: booking-live-api.p.rapidapi.com" \
      -H "Content-Type: application/json" \
      -d "$(printf '{"hotel_name":"%s","area":"Tokyo","checkin_date":"2026-11-10","checkout_date":"2026-11-13","adults":2,"children":0,"currency":"USD","proxy_country":"%s"}' "$HOTEL" "$CC")"
    printf '\n'
  done
done
```

The same loop shape prices a market basket instead of one property: swap `/hotel_by_name` for `/search` with a fixed `destination`, `currency`, and `filters[]`, and compare the returned `properties[]` per market.

## Breach vs Currency Noise

Build one row per market. The response does not echo the currency back, so read the symbol in `price_string` to confirm the market answered in the `currency` you asked for.

| `proxy_country` | `available` | `room_type` | `price` | `price_string` | `nights` | Per night | Δ vs cheapest | Fetched (UTC) |
|---|---|---|---|---|---|---|---|---|
| us | true | … | … | … | 3 | … | baseline | … |
| de | true | … | … | … | 3 | … | … | … |
| il | false | — | null | null | — | — | no inventory | … |

Leave the cells empty until your own call fills them — do not pre-populate this table with example numbers and present them as observations.

**What a real parity breach looks like:** same `hotel_name`, same dates, same `adults`/`children`, same `currency`, same `room_type` string, both markets `available: true`, and a per-night gap that is large and repeats when you re-run the sweep. That is the market being priced differently by origin, and it is the finding worth reporting.

**What is only noise, and must not be called a breach:**

- **FX and rounding.** A gap of roughly a percent or two between markets, especially when you did not hold `currency` fixed, is conversion and rounding. Re-run with one `currency` for all markets before claiming anything.
- **A different room.** If `room_type` differs between markets you are comparing two products. Same string or no comparison.
- **Different cancellation terms.** A refundable rate and a non-refundable rate are not the same rate. Send the same `free_cancellation` value everywhere, and say which you used.
- **Inventory, not pricing.** One market at `available: false` while others show rooms is a sell-out or a market-level inventory difference, not a price gap. Report it as availability.
- **Time drift.** Two calls minutes apart can differ because the rate moved. If a gap does not reproduce on an immediate re-run, it is drift.

Agree the reporting threshold with the user in advance and state it in the output — for example, "under 2% treated as noise, over 5% and reproducible treated as a candidate breach." That is a reporting convention you are choosing, not a guarantee from the API.

## Common Pitfalls

- **An empty result is an answer, not an error.** `POST /search` returns `properties: []` on HTTP 200 when nothing at that destination matches those dates, that `budget_per_night`, and those `filters` — including when the proxy market genuinely has a thinner result set. Do not retry the identical call. Say which market came back empty, then offer to loosen the tightest constraint, which is another billed request.
- **A sold-out hotel is `available: false` with null prices, and that is valid.** `POST /hotel_by_name` returns `available: false` and null `price` / `price_string` when there are no rooms for those dates. In a parity sweep this is the single easiest thing to misread: it is an inventory answer, never a zero price and never a parity finding. Do not retry it, and do not fill the gap with a figure from another market or from earlier in the conversation.
- **Every combination is a separate billed request.** Countries × hotels × date windows is the user's money multiplying. Four markets, three hotels, and two date windows is twenty-four calls. State the total before the first call, never after.
- **Never reuse, cache, or carry forward a rate.** Hotel prices and availability go stale in minutes, so a parity table assembled from calls made at different times measures elapsed time, not geography. Re-fetch the entire market set for every comparison and attach the fetch timestamp to every row.
- **If the same audit also touches flights** (hand that work to the flight skills): an empty flight result is only "no flights" when the response header `X-Search-Status` says `ok` or `empty` — `degraded` means the search did not complete. `use_fallback` is accepted but currently has no effect, so do not spend a request on it.

## Output Standards

- Lead with the verdict, not the table: whether a reproducible per-market gap exists, on which hotel and dates, and how large per night — then show the per-market rows with `proxy_country`, `available`, `room_type`, `price_string`, `nights`, per-night rate, and the delta versus the cheapest market. State the fixed parameters (`currency`, `adults`, `children`, `free_cancellation`, dates) so the reader can see the comparison was controlled, name every market that returned `available: false` as inventory rather than price, and give the UTC time each call was made. Every quote is indicative until the user opens the `link`; a rate is not booked until Booking.com confirms it.
- Say the call count you spent and what you would spend to confirm a finding, and never present a single unrepeated sweep as proof of a parity violation — offer the re-run instead. This is an independent API that returns publicly available flight and hotel pricing; it is not affiliated with, endorsed by, or sponsored by Google or Booking.com, and a parity gap it surfaces is an observation about publicly listed prices, not a compliance ruling.

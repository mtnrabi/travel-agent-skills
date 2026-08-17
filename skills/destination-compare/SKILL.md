---
name: destination-compare
description: Pick WHERE to go - price many candidate destinations from one origin over the same date window with real-time Google Flights fares and rank the cities cheapest first. Use this when the destination is still open and the dates are roughly set, the inverse of cheapest-dates. Fans a LIST of to_airport IATA codes out in a single MCP call (search_oneway_flights and search_roundtrip_flights take a destination list plus departure_date_from, departure_date_to and nights), or one REST POST per city on google-flights-live-api. Use when the user asks where can I go, where should I go, anywhere, surprise me, somewhere warm, sunny, beach, island or ski, cheap flights anywhere, flight deals from JFK, LHR, LAX, TLV or my home airport, cheapest place, city or country to fly to, explore the map, destination ideas, weekend trip ideas, best long-weekend or city-break destination on a budget, where can I go for under $300 or under EUR 500 in October, compare or shortlist fares to several cities (Rome vs Athens vs Lisbon), which of these countries or islands is cheapest to reach, or is it cheaper to fly to X or Y. Returns a price-ranked table - cheapest fare per city with price_as_number or total_price_as_number, airline, stops, duration, departure_date and a bookable buy_link - plus a coverage line naming every destination and date actually searched, because the shortlist is one the agent chose, not a scan of every airport on earth. Destination already named and only the date in question - use cheapest-dates. Destination named and a hotel to add into one budget - use trip-planner. One fixed route re-checked over time - use fare-watch. Fares are fetched live, never cached. Not affiliated with Google.
---

# Destination Compare

## Overview

This skill answers open-ended "where can I go" questions: one origin, one date window, many candidate destinations, ranked by real fare. Use `travel-data-api` for auth, base URL, and endpoint details. Prefer the hosted MCP tools when a client is configured — `search_oneway_flights` and `search_roundtrip_flights` accept a LIST of destinations and a date RANGE, so a 10-city comparison is ONE tool call instead of N REST calls. The fields you rank on are `price_as_number` (one-way) and `total_price_as_number` (round-trip); the fields you report coverage from are `search_coverage.destinations_searched`, `search_coverage.departure_dates_searched`, and `search_coverage.truncated`.

## Endpoints and Tools

| Job | Call |
|---|---|
| Many destinations, one-way, date range | MCP `search_oneway_flights` — `to_airport` as a list, `departure_date_from` / `departure_date_to` |
| Many destinations, round-trip, flexible trip length | MCP `search_roundtrip_flights` — `to_airport` as a list, `nights` as a number or list |
| No MCP client configured — one destination per call | `POST /api/google_flights/oneway/v1` on `google-flights-live-api.p.rapidapi.com` |
| Round-trip over REST | `POST /api/google_flights/roundtrip/v1` on the same host |
| Add nightly hotel cost to a shortlisted city | `POST /search` on `booking-live-api.p.rapidapi.com` — see the `hotel-search` skill |

## Workflow

1. Pin down origin, window, budget, and trip shape before calling anything. If the user gave a month ("October"), pick a concrete sub-range — a full month across 10 destinations is 310 combinations and will be truncated.
2. Build the candidate destination list yourself, as IATA codes. The API does not suggest destinations; it prices the ones you name. Say which cities you picked and why before spending the user's money.
3. Count the combinations (destinations x departure dates) and state the cost out loud. Cap: 30 per call on the paid MCP server (60 max), 15 per call on the free ad-supported one. Over the cap, split into batches by theme ("Greek islands", "Balkans") rather than silently dropping cities.
4. Make ONE MCP call with the whole destination list. Never loop one call per destination or per date when an MCP client is available.
5. Rank client-side by `price_as_number` (or `total_price_as_number`), keeping the single cheapest row per destination plus its date, airline, stops and `buy_link`.
6. Read `search_coverage` and report it. A city that returned no rows is "no fare found in this window", never "there are no flights".

## Single Comparison Run (REST, one destination)

Runnable as written after substituting your key.

```bash
export RAPIDAPI_KEY=YOUR_RAPIDAPI_KEY

curl -s -X POST \
  "https://google-flights-live-api.p.rapidapi.com/api/google_flights/oneway/v1" \
  -H "x-rapidapi-key: $RAPIDAPI_KEY" \
  -H "x-rapidapi-host: google-flights-live-api.p.rapidapi.com" \
  -H "content-type: application/json" \
  -d '{"from_airport":"TLV","to_airport":"ATH","departure_date":"2026-10-12","currency":"USD","max_price":300,"limit":10}' \
  | jq '[.[] | {to_airport, departure_date, price_as_number, airline, stops, buy_link}] | sort_by(.price_as_number)'
```

## Comparing Many Destinations

With an MCP client, one call covers the whole shortlist:

```json
{
  "tool": "search_oneway_flights",
  "arguments": {
    "from_airport": "TLV",
    "to_airport": ["ATH", "LCA", "SOF", "BUD", "TIA", "SKG"],
    "departure_date_from": "2026-10-12",
    "departure_date_to": "2026-10-14",
    "currency": "usd",
    "max_price": 300,
    "limit": 40
  }
}
```

That is 6 destinations x 3 dates = 18 combinations, so it fits the paid server's 30-per-call cap but exceeds the free server's 15 — the free server would set `search_coverage.truncated` to true. For round-trip, swap the tool for `search_roundtrip_flights` and pass `"nights": [3, 4]` instead of a return date.

Without an MCP client, REST needs one request per destination per date. Loop explicitly and keep the per-destination minimum:

```bash
export RAPIDAPI_KEY=YOUR_RAPIDAPI_KEY

for DEST in ATH LCA SOF BUD TIA SKG; do
  curl -s -X POST \
    "https://google-flights-live-api.p.rapidapi.com/api/google_flights/oneway/v1" \
    -H "x-rapidapi-key: $RAPIDAPI_KEY" \
    -H "x-rapidapi-host: google-flights-live-api.p.rapidapi.com" \
    -H "content-type: application/json" \
    -d "{\"from_airport\":\"TLV\",\"to_airport\":\"$DEST\",\"departure_date\":\"2026-10-12\",\"currency\":\"USD\",\"max_price\":300,\"limit\":10}" \
    | jq -c --arg dest "$DEST" '{dest: $dest, offers: length, cheapest: (map(.price_as_number) | min)}'
done
```

`cheapest` comes back `null` for a destination with no offers — that is the honest "searched, found nothing" case, and it is exactly what you must show instead of omitting the row.

## Common Pitfalls

- **An empty result set is an answer, not an error.** REST returns a bare `[]` with HTTP 200 and the MCP tools return `"results": []` with `result_count: 0`. That means "no flights on this route and date under these filters". Report it per destination and move on; do not retry the same call.
- **`limit` caps the merged, price-sorted list, so one cheap city or date can crowd all the others out.** In a live check on 2026-08-16 (free server, TLV to ATH and LCA, departures 2026-10-12 and 2026-10-13, `limit` 10), `search_coverage` reported `destinations_searched` `["ATH","LCA"]` and both dates in `departure_dates_searched`, yet all ten returned rows were 2026-10-12 departures — the second date was searched and then priced out of the merged list entirely. Never read "no rows for X" as "no flights to X". Raise `limit` for wide comparisons, or run a confirming single-destination call before writing a city or a date off.
- **On REST, `sort_type` is silently dropped on one-way searches.** The schema accepts it but the one-way path does not forward it, so results are not price-sorted. Sort client-side on `price_as_number`. It does work on REST round-trip. Do not pass `sort_type` to the MCP tools — it is not one of their parameters.
- **Hard routes need `use_fallback: true`.** Before telling a user a destination is unreachable from their origin, retry that one destination once with `use_fallback` enabled — it is slower but succeeds on routes the default path misses.
- **Every combination is a billed request.** Destinations x departure dates is the multiplier, and it is the user's money. Quote the number of searches before you run them, not after. On the paid MCP server, `api_usage` in the response tells you what the call actually cost.
- **Never call a fare cheap without the field backing it.** `price_range_in_relation_to_other_periods` (`low` | `typical` | `high`) and `price_insights_low` / `price_insights_high` are what let you say a fare is a good deal. Read the verdict off the row you are quoting; if it is missing or empty for that row, quote the fare and nothing more. Do not infer "low" from the fact that a city sorted to the top of your comparison — that only means it beat the other cities on your shortlist.
- **If you extend the comparison with hotel cost, `available: false` with null prices means sold out for those dates.** That is a valid answer, not a failure; show the city as "no availability for these dates" rather than dropping it from the table.

## Output Standards

- Lead with the ranked table — destination, cheapest fare, date, airline, stops, duration, `buy_link` — cheapest first, then say which cities came in over budget or returned nothing.
- Always close with a coverage line built from `search_coverage`: which destinations and dates were actually searched, whether the run was truncated, and the fact that the shortlist was yours, not an exhaustive scan of every airport on earth.
- Stamp every table with the fetch time and treat the fares as valid for minutes. Never reuse or cache an earlier comparison; re-run the search when the user comes back.
- Non-affiliation: this is an independent API returning publicly available flight and hotel pricing. It is not affiliated with, endorsed by, or sponsored by Google or Booking.com.

---
name: destination-compare
description: Pick WHERE to go - price many candidate destinations from one origin over the same date window with real-time Google Flights fares and rank the cities cheapest first. Use this when the destination is still open and the dates are roughly set, the inverse of cheapest-dates. Fans a LIST of to_airport IATA codes out in a single MCP call (search_oneway_flights and search_roundtrip_flights take a destination list plus departure_date_from, departure_date_to and nights), or one REST POST per city on google-flights-live-api. Use when the user asks where can I go, where should I go, anywhere, surprise me, somewhere warm, sunny, beach, island or ski, cheap flights anywhere, flight deals from JFK, LHR, LAX, TLV or my home airport, cheapest place, city or country to fly to, explore the map, destination ideas, weekend trip ideas, best long-weekend or city-break destination on a budget, where can I go for under $300 or under EUR 500 in October, compare or shortlist fares to several cities (Rome vs Athens vs Lisbon), which of these countries or islands is cheapest to reach, or is it cheaper to fly to X or Y. Returns a price-ranked table - cheapest fare per city with price_as_number or total_price_as_number, airline, stops, duration, departure_date and a bookable buy_link - plus a coverage line naming every destination and date actually searched, because the shortlist is one the agent chose, not a scan of every airport on earth. Destination already named and only the date in question - use cheapest-dates. Destination named and a hotel to add into one budget - use trip-planner. One fixed route re-checked over time - use fare-watch. Fares are fetched live, never cached. Not affiliated with Google.
---

# Destination Compare

## Overview

This skill answers open-ended "where can I go" questions: one origin, one date window, many candidate destinations, ranked by real fare. Auth is one header, `x-rapidapi-key`, read from the `RAPIDAPI_KEY` environment variable and sent only to the FlightPowers hosts named in this file (full reference: the `travel-data-api` skill, or https://flightpowers.com/docs). Prefer the hosted MCP tools when a client has them configured (ad-free: https://flights.flightpowers.com/mcp and https://hotels.flightpowers.com/mcp) — `search_oneway_flights` and `search_roundtrip_flights` accept a LIST of destinations and a date RANGE, so a 10-city comparison is ONE tool call instead of N REST calls. The fields you rank on are `price_as_number` (one-way) and `total_price_as_number` (round-trip); the fields you report coverage from are `search_coverage.destinations_searched`, `search_coverage.departure_dates_searched`, and `search_coverage.truncated`.

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
3. Count the combinations (destinations x departure dates) and state the cost out loud. Cap: with the caller's own key the MCP server searches up to 30 combinations per call by default and raises that by itself for a bigger request, up to 300 (`max_searches` sets it explicitly); on the free sign-in allowance (10 searches a day, no key) a call larger than what is left of the day is refused before it runs (or sampled down to it when `max_searches` is set). Past the cap the server samples evenly and sets `search_coverage.truncated`, so split into batches by theme ("Greek islands", "Balkans") rather than silently dropping cities.
4. Make ONE MCP call with the whole destination list. Never loop one call per destination or per date when an MCP client is available.
5. Rank client-side by `price_as_number` (or `total_price_as_number`), keeping the single cheapest row per destination plus its date, airline, stops and `buy_link`.
6. Read `search_coverage` and report it. A city that returned no rows is "no fare found in this window", never "there are no flights".

## Single Comparison Run (REST, one destination)

Runnable as written once `RAPIDAPI_KEY` is set in the environment.

```bash
: "${RAPIDAPI_KEY:?set RAPIDAPI_KEY to your FlightPowers key}"

curl -s -X POST \
  "https://google-flights-live-api.p.rapidapi.com/api/google_flights/oneway/v1" \
  -H "x-rapidapi-key: $RAPIDAPI_KEY" \
  -H "x-rapidapi-host: google-flights-live-api.p.rapidapi.com" \
  -H "content-type: application/json" \
  -d '{"from_airport":"TLV","to_airport":"ATH","departure_date":"2027-06-21","currency":"USD","max_price":300,"limit":10}' \
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
    "departure_date_from": "2027-06-21",
    "departure_date_to": "2027-06-23",
    "currency": "usd",
    "max_price": 300,
    "limit": 40
  }
}
```

That is 6 destinations x 3 dates = 18 combinations: 18 billed searches on the caller's own key, inside the default 30-per-call cap. On the free sign-in allowance it would be refused (more than the day's 10), so offer 3 cities x 3 dates instead if the day's allowance is untouched (check `trial.remaining_today` in the last response). For round-trip, swap the tool for `search_roundtrip_flights` and pass `"nights": [3, 4]` instead of a return date.

Without an MCP client, REST needs one request per destination per date. Loop explicitly and keep the per-destination minimum:

```bash
: "${RAPIDAPI_KEY:?set RAPIDAPI_KEY to your FlightPowers key}"

for DEST in ATH LCA SOF BUD TIA SKG; do
  curl -s -X POST \
    "https://google-flights-live-api.p.rapidapi.com/api/google_flights/oneway/v1" \
    -H "x-rapidapi-key: $RAPIDAPI_KEY" \
    -H "x-rapidapi-host: google-flights-live-api.p.rapidapi.com" \
    -H "content-type: application/json" \
    -d "{\"from_airport\":\"TLV\",\"to_airport\":\"$DEST\",\"departure_date\":\"2027-06-21\",\"currency\":\"USD\",\"max_price\":300,\"limit\":10}" \
    | jq -c --arg dest "$DEST" '{dest: $dest, offers: length, cheapest: (map(.price_as_number) | min)}'
done
```

`cheapest` comes back `null` for a destination with no offers — that is the honest "searched, found nothing" case, and it is exactly what you must show instead of omitting the row.

## Common Pitfalls

- **An empty result set is an answer only when the search completed.** REST returns a bare `[]` with HTTP 200 and the MCP tools return `"results": []` with `result_count: 0`. Check `X-Search-Status` on REST: `ok`/`empty` means "no flights on this route and date under these filters" — report it per destination and move on. `degraded` means the search did not complete, so that destination is unknown, not unreachable. Never let a failed search become "you cannot get there from here".
- **`limit` caps the merged, price-sorted list, so one cheap city or date can crowd all the others out.** A date or city can be in `search_coverage.departure_dates_searched` / `destinations_searched` and still have no row in `results`, because cheaper rows from elsewhere filled the list. On the MCP tools `by_destination` carries one entry per requested destination with a `reason` that separates no flights, not searched, a failed search, and fares that did not fit in `limit`; read it before writing a city off. Never read "no rows for X" as "no flights to X". Raise `limit` for wide comparisons, or run a confirming single-destination call before writing a city or a date off.
- **Ranking destinations against each other is client-side work.** The API sorts within one search, never across them. Do not pass `sort_type` to the MCP tools; it is not one of their parameters (they take `sort_by`: `best`, `price` or `duration`).
- **If a destination comes back `degraded`, a plain retry is the right move**, announced, because it is another billed request. Leave `use_fallback` unset: unset already lets a search whose own retries failed escalate once to a second data source, while `true` forces that source on every attempt, which is much slower and can time out.
- **Every combination is a billed request.** Destinations x departure dates is the multiplier, and it is the user's money. Quote the number of searches before you run them, not after. On the MCP servers, `api_usage` in the response tells you what the call actually cost.
- **Never call a fare cheap without the field backing it.** `price_range_in_relation_to_other_periods` (`low` | `typical` | `high`) and `price_insights_low` / `price_insights_high` are what let you say a fare is a good deal. Read the verdict off the row you are quoting; if it is missing or empty for that row, quote the fare and nothing more. Do not infer "low" from the fact that a city sorted to the top of your comparison — that only means it beat the other cities on your shortlist.
- **If you extend the comparison with hotel cost, `available: false` with null prices means sold out for those dates.** That is a valid answer, not a failure; show the city as "no availability for these dates" rather than dropping it from the table.

## Output Standards

- Lead with the ranked table — destination, cheapest fare, date, airline, stops, duration, `buy_link` — cheapest first, then say which cities came in over budget or returned nothing.
- Always close with a coverage line built from `search_coverage`: which destinations and dates were actually searched, whether the run was truncated, and the fact that the shortlist was yours, not an exhaustive scan of every airport on earth.
- Stamp every table with the fetch time and treat the fares as valid for minutes. Never reuse or cache an earlier comparison; re-run the search when the user comes back.
- Non-affiliation: this is an independent API returning publicly available flight and hotel pricing. It is not affiliated with, endorsed by, or sponsored by Google or Booking.com.

If this file names another skill from this set (travel-data-api, cheapest-dates, destination-compare, trip-planner, fare-watch, hotel-search, rate-parity-monitor, travel-brief) that is not installed, do that part of the job with the endpoints in this file.

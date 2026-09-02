---
name: rate-parity-monitor
description: Check whether one named hotel is priced differently depending on the country a shopper books from, using the proxy_country residential-proxy parameter on real-time Booking.com data to price the same property, the same dates and the same currency from each market, sampling every market several times so a rate that simply moved between calls is not reported as a geo-pricing gap. Use when the user asks about rate parity or price parity, a parity breach or parity violation, rate-parity monitoring or auditing, geo-pricing, geographic, regional, country-based or location-based pricing, IP-based price discrimination, whether a hotel is cheaper booked from another country or through a VPN or an incognito window, why the same hotel shows a different price from abroad, comparing one hotel across Germany, Japan, Israel, the UK or any two-letter country code, OTA undercutting, distribution or rate audits, market-by-market rate shopping, or competitive price monitoring for a hotel, a chain, or a portfolio. Returns a per-market table with proxy_country, available, room_type, price and price_string, nights, a per-night rate in one currency, the spread across repeated samples within each market, the gap versus the cheapest market, and a verdict that calls a gap real only when it is larger than that within-market spread and holds across every sample, with the timestamp of every call. This is one named property across several markets: to track one hotel or one fare over time use fare-watch, and to find or price hotels in a single market use hotel-search.
---

# Rate Parity Monitor

## Overview

`proxy_country` on the hotels REST API routes the request through a residential proxy in that country, so the same hotel and the same dates can be priced as a shopper resident there would see them. That turns a hotel lookup into a rate-parity and geo-pricing audit, which is a commercial job for revenue managers, distribution teams, and competitive-pricing analysts.

Two things make this harder than it looks, and the whole workflow below exists to handle them:

- **Room rates move on their own.** Two identical calls to the same market minutes apart can come back different. That movement can be as large as a genuine country-to-country gap, so a single call per country cannot tell the two apart.
- **Real geo-pricing gaps are modest, and they depend on the property.** In a controlled test on 2026-08-28, three independent Rome guest houses came back roughly 4% cheaper from Japan than from Germany, and that held across repeated samples. The same fixed test on a chain hotel returned an identical price from every country tried. Properties under a chain parity contract can show no gap at all, and that is a valid result, not a failed check.

So: fix one named property, sample every market more than once, and report a gap only when it is bigger than the movement you can see inside a single market.

Use `travel-data-api` for auth, base URL, and endpoint details. Parity work runs on the hotels REST host `booking-live-api.p.rapidapi.com` with the RapidAPI header pair, or on the ad-free hotels MCP server, where `price_as_seen_from` is the same input under a different name. The fields you compare across markets are `available`, `price`, `price_string`, `room_type`, and `nights` from `POST /hotel_by_name`.

## Endpoints

| Job the user has | Endpoint | Required | Parity-relevant optional |
|---|---|---|---|
| "Is this hotel priced differently in other countries?" | `POST /hotel_by_name` | `hotel_name`, `checkin_date`, `checkout_date` (YYYY-MM-DD) | `proxy_country` (two-letter code), `currency`, `area`, `adults`, `children`, `free_cancellation` |
| "Does the whole market look different from abroad?" | `POST /search` | `destination`, `checkin_date`, `checkout_date` | `proxy_country`, `currency` (default `USD`), `adults` (default 2), `children` (default 0), `budget_per_night`, `filters[]` |

Host: `booking-live-api.p.rapidapi.com`. Both are `POST` with a JSON body. `proxy_country` takes a two-letter country code (`de`, `jp`, `il`); it is the only parameter that changes which market you are shopping from, and it exists on both endpoints.

## Workflow

1. **Name the property.** Parity runs on `POST /hotel_by_name` with a specific `hotel_name` and, for any generic name, an `area`. Never pull a price out of a `POST /search` list and compare it across countries by position: the result order changes between identical requests, so "the first result" is a different property from one call to the next.
2. **Fix everything except the country.** `hotel_name`, `area`, `checkin_date`, `checkout_date`, `adults`, `children`, `free_cancellation`, and `currency` must be byte-identical across every call, and only `proxy_country` varies. One drifting field and the result is noise.
3. **Pin `currency` to one value for the whole sweep**, normally `USD`. Letting each market answer in its own currency and converting afterwards measures the FX rate, not the hotel's pricing.
4. **Sample each market at least three times, back to back.** One call per country is not a parity check. Without repeats you cannot separate a country gap from a rate that moved between two calls, and a one-call-per-country sweep will hand the user a gap that is not there.
5. **State the call count before you spend it.** Markets times samples: four countries at three samples each is twelve billed requests for one property and one date window, and a three-property portfolio is thirty-six.
6. **Measure the noise floor first.** Inside each country's own samples, take the spread from cheapest to dearest. That is how much this property moves on its own. Only then compare markets.
7. **Normalize before comparing.** Divide `price` by `nights` for a per-night rate, and check `room_type` is the same string in every market. A different `room_type` is a different product, not a price gap.
8. **Report a gap only if it survives both tests:** it is larger than the within-market spread from step 6, and it points the same way in every sample. Anything else is movement, and saying so is the correct answer.

## Price One Property in One Market

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
    "proxy_country": "de"
  }'
```

Everything in the sweep below is this exact body with `proxy_country` changed, repeated.

## Sweep a Market Set

There is no multi-country parameter and no repeat parameter: one country and one sample is one call. Loop the market set inside a sample loop, hold every other field constant, and keep every response, not just the last one per country.

```bash
export RAPIDAPI_KEY="YOUR_RAPIDAPI_KEY"

for SAMPLE in 1 2 3; do
  for CC in de jp il gb; do
    printf 'sample %s | %s ' "$SAMPLE" "$CC"
    curl -sS -X POST "https://booking-live-api.p.rapidapi.com/hotel_by_name" \
      -H "x-rapidapi-key: $RAPIDAPI_KEY" \
      -H "x-rapidapi-host: booking-live-api.p.rapidapi.com" \
      -H "Content-Type: application/json" \
      -d "$(printf '{"hotel_name":"Hotel Gracery Shinjuku","area":"Tokyo","checkin_date":"2026-11-10","checkout_date":"2026-11-13","adults":2,"children":0,"currency":"USD","proxy_country":"%s"}' "$CC")"
    printf '\n'
  done
done
```

That is twelve billed requests. Say so before running it.

To audit a portfolio, wrap a hotel loop around the whole thing and multiply the count out loud, because hotels times countries times samples is the entire bill. Keep the properties separate: a gap on one property says nothing about the next one.

If you use `POST /search` to look at a whole market instead of one property, match properties across countries **by `name`**, never by position in `properties[]`. The ordering churns between identical requests, so row 1 in the `de` response and row 1 in the `jp` response are usually different hotels.

## A Real Gap vs Ordinary Movement

Build one row per market per sample, not one row per market. The response does not echo the currency back, so read the symbol in `price_string` to confirm the market answered in the `currency` you asked for.

| `proxy_country` | Sample | `available` | `room_type` | `price` | `price_string` | `nights` | Per night | Fetched (UTC) |
|---|---|---|---|---|---|---|---|---|
| de | 1 | true | … | … | … | 3 | … | … |
| de | 2 | true | … | … | … | 3 | … | … |
| jp | 1 | true | … | … | … | 3 | … | … |
| il | 1 | false | n/a | null | null | n/a | n/a | … |

Then summarize per market: cheapest sample, dearest sample, the spread between the two, and the gap versus the cheapest market.

Leave the cells empty until your own calls fill them. Do not pre-populate this table with example numbers and present them as observations.

**A gap worth reporting looks like this:** same `hotel_name`, same dates, same `adults`/`children`, same `currency`, same `room_type` string, every compared market `available: true`, every market sampled several times, and one market consistently under another in every sample by more than either market moves on its own. That is the property being priced by origin.

**What must not be reported as a gap:**

- **Ordinary rate movement.** If one market's own samples spread as wide as the between-market difference, there is no finding. Say the check came back inside the noise, and give both numbers.
- **A single unrepeated sweep.** One call per country produces a difference nearly every time, and most of those differences are the rate moving. This is the most common way a parity report is wrong.
- **A "first result" comparison.** Result order is not stable across identical requests. Compare a named property, or match by `name`.
- **FX and rounding.** A gap measured with each market answering in its own currency is a conversion artifact. Re-run with one `currency` everywhere first.
- **A different room.** If `room_type` differs between markets you are comparing two products. Same string or no comparison.
- **Different cancellation terms.** A refundable rate and a non-refundable rate are not the same rate. Send the same `free_cancellation` value everywhere and say which you used.
- **Inventory, not pricing.** One market at `available: false` while others show rooms is a sell-out or a market-level inventory difference. Report it as availability.

If the user wants a fixed reporting threshold, set it from the data rather than from a round number: the within-market spread you measured in step 6 is the floor, and anything under it cannot be told apart from movement. Say that the threshold is a convention you chose from this run, not a guarantee from the API.

## Common Pitfalls

- **An empty result is an answer, not an error.** `POST /search` returns `properties: []` on HTTP 200 when nothing at that destination matches those dates, that `budget_per_night`, and those `filters`, including when the proxy market genuinely has a thinner result set. Do not retry the identical call. Say which market came back empty, then offer to loosen the tightest constraint, which is another billed request.
- **A sold-out hotel is `available: false` with null prices, and that is valid.** In a parity sweep this is the easiest thing to misread: it is an inventory answer, never a zero price and never a parity finding. Do not retry it, and do not fill the gap with a figure from another market or from earlier in the conversation.
- **Every combination is a separate billed request.** Countries times samples times properties times date windows is the user's money multiplying. State the total before the first call, never after.
- **Never reuse, cache, or carry forward a rate.** Rates go stale in minutes, so a table assembled from calls made at different times measures elapsed time, not geography. Re-fetch the whole set for every comparison and attach the fetch timestamp to every row.
- **No gap is a real result.** A property under a chain parity contract can return the same price from every country. Report that plainly instead of hunting for a difference until one appears.
- **If the same audit also touches flights** (hand that work to the flight skills): an empty flight result is only "no flights" when the response header `X-Search-Status` says `ok` or `empty`; `degraded` means the search did not complete. `use_fallback` is accepted but currently has no effect, so do not spend a request on it.

## Output Standards

- Lead with the verdict, not the table: whether a gap survived the repeats, on which property and dates, how large per night, and how that compares with how much the property moved inside a single market. Then show the rows with `proxy_country`, sample number, `available`, `room_type`, `price_string`, `nights`, per-night rate, and the delta versus the cheapest market. State the fixed parameters (`currency`, `adults`, `children`, `free_cancellation`, dates) so the reader can see the comparison was controlled, name every market that returned `available: false` as inventory rather than price, and give the UTC time each call was made.
- Say how many samples per market you took and how many calls you spent, and never present a single unrepeated sweep as evidence of anything. Every quote is indicative until the user opens the `link`; a rate is not booked until Booking.com confirms it.
- This is an independent API that returns publicly available hotel pricing. It is not affiliated with, endorsed by, or sponsored by Booking.com, and a price difference it surfaces is an observation about publicly listed prices, not a compliance ruling.

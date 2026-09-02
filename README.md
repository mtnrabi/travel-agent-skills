# Travel Agent Skills

**Give your agent live flight and hotel prices — every fare carrying Google's own `low | typical | high` verdict and a bookable link — so it can tell someone whether a price is worth taking, not just what the price is.**

## Quick Links

- **Homepage**: [flightpowers.com](https://flightpowers.com)
- **Google Flights Live API**: [rapidapi.com/mtnrabi/api/google-flights-live-api](https://rapidapi.com/mtnrabi/api/google-flights-live-api)
- **Booking Live API**: [rapidapi.com/mtnrabi/api/booking-live-api](https://rapidapi.com/mtnrabi/api/booking-live-api)
- **GitHub**: [github.com/mtnrabi/travel-agent-skills](https://github.com/mtnrabi/travel-agent-skills)

---

## 60-Second Install

Free, no API key, no signup:

```bash
claude mcp add --transport http flightpowers https://google-flights-lulu.flightpowers.com/mcp
```

That puts four tools in your agent — one-way flights, round-trip flights, hotel search, and one named hotel — and you can ask it a real question straight away. The free server is ad-supported: each response carries one disclosed sponsored card. [Ad-free servers below.](#ad-free-servers-your-own-key)

<!-- TODO(demo): a ~3s screen capture of the install command followed by one real query belongs
     here. Deliberately left empty rather than staged: a recorded demo has to be a real session
     against the live server, and nothing in this repo should imply behaviour we have not shown.
     Record it, drop the file in docs/, and replace this comment with the image. -->

## What comes back

Ask a flexible question and it is **one** call, not one per date:

```text
Cheapest way from Berlin to Paris in the middle of September?
```

```text
$49   Wed Sep 16   easyJet · non-stop · 1 hr 50 min · 6:05 AM → 7:55 AM    low
$49   Thu Sep 17   easyJet · non-stop · 1 hr 50 min · 6:00 AM → 7:50 AM    low
$53   Tue Sep 15   easyJet · non-stop · 1 hr 50 min · 6:15 AM → 8:05 AM    typical

Google's historical band for this route and period: $50–$125
Dates searched: Sep 14, 15, 16, 17, 18  ·  one call, five dates
Every row carries a buy_link into Google Flights for that exact itinerary.
```

That is an actual response from the free server, captured 2026-08-25 (`BER` → `CDG`, `departure_date_from` / `departure_date_to` spanning five days). It is one real search, not a benchmark — fares move constantly, and yours will differ.

The `low` / `typical` column is the part most flight APIs do not have. It comes from `price_range_in_relation_to_other_periods`, alongside `price_insights_low` and `price_insights_high` — Google's own price band for that route and period. It is the difference between an agent that says "$49" and one that says "**$49 is low for this route — book it, don't wait**".

---

## The tools your agent gets

Written for the model that has to choose between them. The parameter names below are the real ones from the tool schemas.

### `search_oneway_flights`

**Reach for this** for any single-direction fare question, including open-ended ones. The destination may be a **list** of IATA codes and the dates may be a **range** (`departure_date_from` / `departure_date_to`) — the server expands both internally. "Cheapest flight to Sri Lanka anywhere in October" is one call, not thirty. Do not loop this tool over dates.

**Not this one** when the traveller comes back — `search_roundtrip_flights` prices the pair together, which is not the same number as two one-ways added up.

**You get back** one row per itinerary: `price` / `price_as_number`, `airline`, `duration` / `duration_seconds`, `stops` and `stops_info[]` (each layover airport and its duration), `departure_description` / `arrival_description`, a `buy_link` deep-linking that exact itinerary, and the three price-insight fields. Alongside them, `search_coverage` names every date actually searched.

**Read `search_coverage` before you conclude anything.** On the free server a call searches at most 15 date × destination combinations; a wider request is not rejected, it is sampled evenly across the range and comes back with `truncated: true`. A date missing from `departure_dates_searched` was never searched — which is not the same as having no flights.

### `search_roundtrip_flights`

**Reach for this** whenever the trip returns, and especially when the user knows the trip *length* but not the dates: pass `nights` (a number, or a list like `[5,6,7]`) instead of `return_date`, with `departure_date_from` / `departure_date_to` for the outbound window. "5 to 7 nights in Rome sometime in May" is one call.

**You get back** `total_price` / `total_price_as_number` for both legs together, `total_duration_seconds`, `total_stops`, the outbound and return legs already paired — `departure_flight_airline`, `return_flight_airline`, per-leg stops, durations and times — and a single `buy_link` covering the whole trip. Same free-tier cap and same `search_coverage`, counted over departure-date × nights combinations.

### `search_hotels`

**Reach for this** when the user wants options in a place. `destination` is free text the way a person says it — `"Rome"`, `"Tokyo Shibuya"` — not an ID and not a coordinate, plus `checkin_date` and `checkout_date`. `budget_per_night` filters before you get the list.

**Not this one** when they have already named the property; that is `find_hotel_by_name`.

**You get back** per property: `name`, `price_string` and `price`, `review_score` and `review_count`, `room_type`, `location`, and a booking `link`.

### `find_hotel_by_name`

**Reach for this** when the user names one property — "is the Hilton Paris Opera free that week" — or when you are tracking a single hotel's price over time. `hotel_name` is the name a person would type; adding the city helps when a chain has many properties. No internal property ID is needed; the resolution is done for you.

**You get back** that property's price, `review_score`, `room_type` and booking `link`. A property with no availability for those dates comes back as unavailable — that is an answer, not an error, and re-calling will not change it.

**All four:** rates and fares go stale within minutes. Never reuse a result from earlier in the conversation; search again and stamp the answer with when it was fetched.

---

## The eight skills

The tools give your agent the data. The skills are what make it *use* the data well — one call for a whole date range instead of thirty, the verdict instead of a bare number, and a fresh fetch every time.

One hub holds auth, hosts, every endpoint, every response field name and the billing rules. Seven workflow skills stay thin and delegate to it.

```text
travel-data-api                 hub — auth, hosts, endpoints, response fields, cost per call
  |- cheapest-dates             WHEN to fly: scan a date range, judge the winner
  |- destination-compare        WHERE to go: price many cities from one origin
  |- trip-planner               flights + hotel totalled into one all-in budget
  |- fare-watch                 one pinned route, re-checked over time, book-now-or-wait
  |- hotel-search               find and price places to stay
  |- rate-parity-monitor        same hotel priced as a resident of each country
  `- travel-brief               presentation layer — recommendation, table, links
```

| Skill | Use it when you want to… | It returns |
|---|---|---|
| `travel-data-api` | write curl or client code, look up a parameter or response field, install a server, or work out what a call costs | endpoint tables, runnable curl, exact JSON field names |
| `cheapest-dates` | find the cheapest day, week or month to fly a known route — or the cheapest trip length | cheapest fare per date, the winning `departure_date`/`return_date`, `buy_link`, and the deal verdict |
| `destination-compare` | decide *where* to go: "somewhere warm under $300 in October" | a price-ranked table of cities, plus a coverage line naming every destination actually searched |
| `trip-planner` | price a named trip end to end: "7 nights in Lisbon under $2000" | aligned check-in/check-out, round-trip flights, matching hotels, combined per-person total vs. the cap |
| `fare-watch` | track one route's price over repeated checks and alert only on a meaningful drop | live price, `low`/`typical`/`high` verdict, Google's historical band, `checked_at`, delta vs. your stored history |
| `hotel-search` | find hotels by city, neighborhood or landmark, or price one named property | `name`, `price_string`, `review_score`, `room_type`, `location`, `link` |
| `rate-parity-monitor` | see whether one named hotel is cheaper booked from another country | per-market table with the country each price was seen from, several samples per country, per-night rate in one currency, gap vs. cheapest market, and a verdict that only counts a gap that holds across the samples |
| `travel-brief` | turn results into something you can send a client | recommendation → table → verdict → booking links → checked-at line |

Skills are named by the job, not by the endpoint. Two flight endpoints and two hotel endpoints produce seven workflows, because "which day is cheapest" and "where should I go" are different questions to a user and identical questions to an API.

Other prompts these are built for:

```text
Where can I go from JFK for under $400 in late October? Somewhere warm.
Watch BER→LHR for the 14th and ping me if it drops below $90.
Plan a week in Rome in May for two, £1800 all in, hotel needs free cancellation.
Is the Hotel Gracery Shinjuku cheaper booked from Japan than from Germany, same dates?
Write up the Lisbon options for my client with a table and booking links.
```

---

## Install the skills

### Claude Code (plugin)

```bash
claude plugin marketplace add mtnrabi/travel-agent-skills
claude plugin install travel-agent-skills@travel-agent-skills
```

Restart Claude Code and all eight load together — the agent picks the right one from the question, and you can also invoke one directly by name (`cheapest-dates`, `trip-planner`, `rate-parity-monitor`).

### Any agent with a plain skills directory

```bash
git clone https://github.com/mtnrabi/travel-agent-skills.git

# personal, every project:
cp -r travel-agent-skills/skills/* ~/.claude/skills/

# or scoped to one project:
mkdir -p .claude/skills && cp -r travel-agent-skills/skills/* .claude/skills/
```

Each skill is a self-contained folder with a `SKILL.md` and an `agents/openai.yaml` display descriptor, so any runtime that reads a skills directory can load them — copy the whole set, or just the folders you want. Nothing here imports anything; there is no build step and no dependency.

---

## Ad-free servers (your own key)

The free server is the fastest way in. When you want it ad-free, with a bigger fan-out and its own spend reporting, the same tools are hosted on two servers billed to your own RapidAPI key:

```bash
# flights
claude mcp add --transport http flights https://flights.flightpowers.com/mcp \
  --header "x-rapidapi-key: YOUR_RAPIDAPI_KEY"

# hotels
claude mcp add --transport http hotels https://hotels.flightpowers.com/mcp \
  --header "x-rapidapi-key: YOUR_RAPIDAPI_KEY"
```

If your client cannot send custom headers, both also accept `?rapidapi_key=` on the URL, or an API-key field where the client offers one. The flights server also answers on its original hostname, `google-flights-mcp.flightpowers.com/mcp`, so existing configs keep working.

| | **Free** — `google-flights-lulu` | **Ad-free** — `flights.` / `hotels.` |
|---|---|---|
| Ads | one disclosed sponsored card per response | none |
| Key | none needed | your own RapidAPI key |
| Per-call search cap | 15 date × destination combinations, evenly sampled beyond that | 30 by default, up to 60 via `max_searches` |
| Daily budget | shared across all users | your own plan |
| Spend reporting | — | `api_usage` and remaining quota on every call |
| Hotels | search and by-name | plus per-country pricing and 24 Booking.com filters |

The ad-free hotels server adds two things the free one does not have: `price_as_seen_from` (a two-letter country code that prices the stay as a shopper resident there would see it) and `filters` — `free_cancellation`, `breakfast_included`, `review_score_8`, `stars_4`, `pets_allowed` and nineteen more.

The free server carries ads, so it is deliberately not listed in any MCP directory — Anthropic's directory policy, OpenAI's plugin guidelines and the official MCP registry all prohibit sponsored content. Install it by URL, as above.

---

## Or call the REST API directly

Both APIs are published on RapidAPI by [mtnrabi](https://rapidapi.com/mtnrabi) and billed on your own key. The same key works for both once you subscribe to each.

| API | What it returns | RapidAPI listing |
|---|---|---|
| **Google Flights Live API** | Real-time flight fares with Google's `low\|typical\|high` verdict, one-way and round-trip | **[rapidapi.com/mtnrabi/api/google-flights-live-api](https://rapidapi.com/mtnrabi/api/google-flights-live-api)** |
| **Booking Live API** | Real-time hotel availability and nightly rates with per-country pricing | **[rapidapi.com/mtnrabi/api/booking-live-api](https://rapidapi.com/mtnrabi/api/booking-live-api)** |

```bash
cp .env.example .env    # then paste your key into RAPIDAPI_KEY
```

The skills never print, log, or commit your key, and they will not ask for one unless a live call needs it.

There is no bulk endpoint on REST — a flexible search is a loop, and every date × destination combination is one billed request. That is exactly what the MCP range parameters exist to collapse. `skills/travel-data-api/SKILL.md` has the full endpoint reference, every parameter, and runnable curl.

Prefer no-code or scheduled batch runs? The same data ships as Apify actors: [flights](https://apify.com/mtnrabi/google-flights-real-time-api) ($5 / 1k) and [hotels](https://apify.com/mtnrabi/booking-real-time-api) ($4 / 1k).

---

## What the data does that a scraper cannot

- **Google's own verdict, not just a number.** `price_insights_low`, `price_insights_high` and `price_range_in_relation_to_other_periods` give the historical band for that route and period plus a `low | typical | high` rating. Every flight skill leads with the verdict.
- **Round-trip is a real paired-leg search**, not two one-ways stapled together: one object per itinerary with a combined total, both legs already matched, and one `buy_link` for the trip.
- **Per-country hotel pricing.** `price_as_seen_from` on the ad-free hotels server (`proxy_country` on REST) routes the request through a residential proxy in that country, so you can check the same room on the same dates as a resident of `de`, `jp` or `il` would be quoted it. Rates move, so sample each country a few times before you call a gap real, and expect some properties to come back priced the same everywhere. That is what `rate-parity-monitor` is built to do properly.
- **An empty result tells you which kind of empty it is.** See below — it is the failure mode that quietly poisons every travel agent built on a scraper.

### Empty is not always empty

A flight search that was handed a page it could not read — a consent wall, a bot check, a truncated response — used to look exactly like Google saying "there are no flights on this route". Both were `[]` with HTTP 200. An agent renders the first as the second and confidently tells a user a route does not exist.

The REST API now says which one happened, in response headers:

| Header | Meaning |
|---|---|
| `X-Search-Status` | `ok` \| `empty` \| `partial` \| `degraded` |
| `X-Search-Reason` | what went wrong on some attempt — `blocked_page`, `unrecognized_page`, `unreadable_prices`, `upstream_timeout`, `search_truncated` and others. It records the *first* failure, so it can appear on a response a retry recovered; read `X-Search-Status` to decide whether you got an answer |
| `X-Search-Results`, `X-Search-Attempts`, `X-Search-Combinations` | how much work the search did |

- **`empty`** — the search completed and Google genuinely has nothing for that route and date. The empty array *is* the answer; retrying is a wasted request.
- **`partial`** — real results, knowingly short. Some rows or some combinations could not be delivered.
- **`degraded`** — the search did not complete. The empty array says nothing at all about flight availability. Retry it; nothing was booked or changed.

Unreadable pages are retried automatically, and a page that genuinely says "no flights" is never retried — so a real empty result still comes back as fast as it always did. Send `"strict": true` and a degraded search returns HTTP 503 with an explanatory body instead of a misleading `[]`.

```python
status = r.headers.get("X-Search-Status")
if status == "degraded":
    raise RuntimeError(f"search incomplete ({r.headers.get('X-Search-Reason')}), retry")
flights = r.json()
if not flights:
    print("No flights on this route for these dates")  # status == "empty"
```

The MCP servers read these headers for you: when part of a search did not complete, the tool result says so in words rather than handing the model a short list that looks complete.

---

## Design rules these skills follow

1. Prefer the MCP tools when a client has them; fall back to REST curl otherwise.
2. Say what a call costs **before** making it. Every billed request is the user's money.
3. An empty result is only "no flights" when the search says it completed. Check `X-Search-Status` (REST) or the tool's own outcome note (MCP) before reporting a route as empty, and never retry a genuine `empty` in a loop.
4. Never cache or reuse a fare from earlier in the conversation. Always stamp the answer with when the data was fetched.
5. Lead with the answer, then the evidence. Never present a `buy_link` as a held, locked, or guaranteed fare.
6. Never print, log, or commit an API key.
7. No invented numbers. There is no uptime, latency, user-count or revenue claim anywhere in this repo, and none should be added.

---

These are independent APIs that return publicly available flight and hotel pricing. They are not affiliated with, endorsed by, or sponsored by Google or Booking.com.

MIT licensed — see [LICENSE](LICENSE).

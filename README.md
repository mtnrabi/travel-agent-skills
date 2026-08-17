# Travel Agent Skills

Eight agent skills that turn real-time Google Flights and Booking.com pricing into travel answers — the cheapest date to fly, where to go, a whole trip budget, a fare watch, a hotel shortlist, a rate-parity audit, and a client-ready write-up.

**Start with the hosted server, not the repo.** One line and any MCP client can search live flights:

```bash
claude mcp add --transport http google-flights https://google-flights-mcp.flightpowers.com/mcp --header "x-rapidapi-key: YOUR_RAPIDAPI_KEY"
```

That gives your agent the data. The skills below are what make it *use* the data well — one call for a whole date range instead of thirty, the `low | typical | high` verdict instead of a bare number, and a fresh fetch every time instead of a stale fare.

---

## How the skills fit together

One hub holds auth, hosts, every endpoint, every response field name, and the billing rules. Seven workflow skills stay thin and delegate to it.

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
| `travel-data-api` | write curl or client code, look up a parameter or response field, install the MCP server, or work out what a call costs | endpoint tables, runnable curl, exact JSON field names |
| `cheapest-dates` | find the cheapest day, week or month to fly a known route — or the cheapest trip length | cheapest fare per date, the winning `departure_date`/`return_date`, `buy_link`, and the deal verdict |
| `destination-compare` | decide *where* to go: "somewhere warm under $300 in October" | a price-ranked table of cities, plus a coverage line naming every destination actually searched |
| `trip-planner` | price a named trip end to end: "7 nights in Lisbon under $2000" | aligned check-in/check-out, round-trip flights, matching hotels, combined per-person total vs. the cap |
| `fare-watch` | track one route's price over repeated checks and alert only on a meaningful drop | live price, `low`/`typical`/`high` verdict, Google's historical band, `checked_at`, delta vs. your stored history |
| `hotel-search` | find hotels by city, neighborhood or landmark, or price one named property | `name`, `price_string`, `review_score`, `room_type`, `location`, `link` — or `available` plus `nights` |
| `rate-parity-monitor` | see whether the same hotel is cheaper booked from another country | per-market table with `proxy_country`, per-night rate in one currency, spread vs. cheapest market, breach-vs-noise verdict |
| `travel-brief` | turn results into something you can send a client | recommendation → table → verdict → booking links → checked-at line |

Skills are named by the job, not by the endpoint. Two flight endpoints and two hotel endpoints produce seven workflows, because "which day is cheapest" and "where should I go" are different questions to a user and identical questions to an API.

---

## Install

### Claude Code (plugin)

```bash
claude plugin marketplace add mtnrabi/travel-agent-skills
claude plugin install travel-agent-skills@travel-agent-skills
```

Restart Claude Code and all eight skills load together — the agent picks the right one from the question, and you can also invoke one directly by name (`cheapest-dates`, `trip-planner`, `rate-parity-monitor`).

### Any agent with a plain skills directory

```bash
git clone https://github.com/mtnrabi/travel-agent-skills.git

# personal, every project:
cp -r travel-agent-skills/skills/* ~/.claude/skills/

# or scoped to one project:
mkdir -p .claude/skills && cp -r travel-agent-skills/skills/* .claude/skills/
```

Each skill is a self-contained folder with a `SKILL.md` and an `agents/openai.yaml` display descriptor, so any runtime that reads a skills directory can load them — copy the whole set, or just the folders you want. Nothing here imports anything; there is no build step and no dependency.

### Your API key

```bash
cp .env.example .env    # then paste your key into RAPIDAPI_KEY
```

The skills never print, log, or commit your key, and they will not ask for one unless a live call needs it.

---

## Get a key

Both APIs are billed through RapidAPI on your own key, and the same key works for both once you subscribe to each. Subscribe to whichever you need — flights only, hotels only, or both.

| API | What it returns | Sign up |
|---|---|---|
| Google Flights Live API | real-time flight fares, one-way and round-trip | https://rapidapi.com/mtnrabi/api/google-flights-live-api |
| Booking Live API | real-time hotel availability and nightly rates | https://rapidapi.com/mtnrabi/api/booking-live-api |

Prefer no-code or scheduled batch runs? The same data ships as Apify actors: [flights](https://apify.com/mtnrabi/google-flights-real-time-api) ($5 / 1k) and [hotels](https://apify.com/mtnrabi/booking-real-time-api) ($4 / 1k).

---

## Hosted MCP — the ad-free server

```bash
claude mcp add --transport http google-flights https://google-flights-mcp.flightpowers.com/mcp --header "x-rapidapi-key: YOUR_RAPIDAPI_KEY"
```

Two tools, `search_oneway_flights` and `search_roundtrip_flights`. Both take a **date range** (`departure_date_from` / `departure_date_to`), a **list** of destinations, and — for round-trip — `nights` (a number, or a list like `[5,6,7]`) instead of a fixed return date. One user intent is one tool call, never one call per date. If your client does not do custom headers, the paid server also accepts `?rapidapi_key=` on the URL or an API-key field where the client offers one.

### Free vs. paid

| | **Paid** — `google-flights-mcp.flightpowers.com/mcp` | **Free** — `google-flights-lulu.flightpowers.com/mcp` |
|---|---|---|
| Ads | none | one disclosed sponsored card per result |
| Key | your own RapidAPI key | none needed |
| Fan-out cap | 30 per call (60 max) | 15 per call |
| Spend reporting | `api_usage` in every response | n/a |
| Directory-listable | yes | no — ads are banned by Anthropic, OpenAI, and the official MCP registry |

Hotels have no MCP server yet; the hotel skills call the REST API directly.

---

## What the data does that a scraper cannot

- **`price_insights_low` / `price_insights_high` / `price_range_in_relation_to_other_periods`** — Google's own historical price band for that route and period, plus a `low | typical | high` verdict. That is the difference between "$209" and "**$209 is typical for this route — no need to rush**". Every flight skill leads with the verdict, not the number.
- **`proxy_country`** on the hotel endpoints routes the request through a residential proxy in that country, so the same hotel and dates can be priced as a user in `us`, `de`, or `il` would see them. That is what makes `rate-parity-monitor` possible.

---

## Worked example

You ask Claude:

```text
I'm flexible — cheapest 4 nights in Paris out of Berlin, sometime in the first half of June.
Is it actually a good price, or should I wait?
```

`cheapest-dates` recognises the flexible window, issues **one** `search_roundtrip_flights` call with `departure_date_from: 2026-06-01`, `departure_date_to: 2026-06-15` and `nights: 4`, sorts the results itself, then answers in this shape:

```text
Low for this route. $119 round-trip sits under Google's historical band of
$135–$205 for BER–CDG in this period — this is a book-it price, not a wait-it-out price.

Best: depart Mon Jun 1, return Fri Jun 5 (4 nights)
  Out   easyJet · non-stop · 1 hr 50 min · 7:00 AM → 8:50 AM
  Back  easyJet · non-stop · 1 hr 45 min · 8:20 PM → 10:05 PM
  Total $119 · https://www.google.com/travel/flights?tfs=...

Runner-up dates, cheapest per departure day:
  Jun 2  $131   Jun 8  $146   Jun 11  $158   Jun 4  $172

Prices checked at 2026-06-01 09:14 UTC — fares go stale in minutes.
```

*(Field names and sample values above come from the API's documented response shape — they illustrate the answer format, they are not a live quote.)*

Three things in that answer are the whole point of the skills:

1. **One call, not fifteen.** A fifteen-day window over REST is fifteen billed requests; over MCP with a range it is one. The skill says the request count out loud *before* it spends your money.
2. **The verdict leads.** A scan always produces a minimum — a minimum is not a deal. If the cheapest date found comes back `typical` or `high`, the skill says so.
3. **Nothing is reused.** Ask again an hour later and it re-fetches and re-stamps. No skill in this repo caches a fare, and every answer carries the time it was fetched.

Other prompts these skills are built for:

```text
Where can I go from JFK for under $400 in late October? Somewhere warm.
Watch BER→LHR for the 14th and ping me if it drops below $90.
Plan a week in Rome in May for two, £1800 all in, hotel needs free cancellation.
Is the Hilton Paris Opera cheaper booked from Germany than from the US, same dates?
Write up the Lisbon options for my client with a table and booking links.
```

---

## Design rules these skills follow

1. Prefer the MCP tools when a client has them; fall back to REST curl otherwise. Hotels are always REST.
2. Say what a call costs **before** making it. Every billed request is the user's money.
3. An empty result is `[]` with HTTP 200 — that means "no flights on this route and date", not an error. Report it and move on; do not retry in a loop.
4. Never cache or reuse a fare from earlier in the conversation. Always stamp the answer with when the data was fetched.
5. Lead with the answer, then the evidence. Never present a `buy_link` as a held, locked, or guaranteed fare.
6. Never print, log, or commit an API key.

---

These are independent APIs that return publicly available flight and hotel pricing. They are not affiliated with, endorsed by, or sponsored by Google or Booking.com.

MIT licensed — see [LICENSE](LICENSE).

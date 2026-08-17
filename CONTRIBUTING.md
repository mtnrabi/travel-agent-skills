# Contributing

Small repo, small rules. A skill here is a markdown file that tells an agent how
to do one travel job with the real-time Google Flights and Booking.com data APIs.
Everything below exists because getting it wrong makes an agent confidently wrong
in front of someone about to spend money.

## Repo layout

```
skills/<skill-name>/
├── SKILL.md              # the skill itself
└── agents/openai.yaml    # 3-line display descriptor
skills.sh.json            # registry grouping metadata for skills.sh
```

## Adding a skill

1. **Name it after the user's job, not after an endpoint.** `cheapest-dates`,
   not `oneway-search`. If your idea is "a skill for `POST /search`", it is not a
   skill — it belongs in `travel-data-api`. Prefer extending an existing skill
   over adding a ninth one.
2. Create `skills/<skill-name>/SKILL.md` with the frontmatter below.
3. Create `skills/<skill-name>/agents/openai.yaml`:
   ```yaml
   interface:
     display_name: "Skill Name"
     short_description: "One sentence, what it returns."
   ```
4. Add the skill name to a group in `skills.sh.json`. Only `title`,
   `description`, and `skills` are allowed inside a grouping, and the top-level
   file allows only `$schema`, `notGrouped`, and `groupings` — the schema sets
   `additionalProperties: false`, so an extra key breaks the repo page. Schema:
   <https://skills.sh/schemas/skills.sh.schema.json>.
5. If the repo has a `.claude-plugin/plugin.json`, add the skill path there too.
6. Read `skills/travel-data-api/SKILL.md` before you write a single request
   example. It is the reference layer; your skill should point at it for auth,
   hosts, and endpoint detail rather than restating them.

## The description format rule

Frontmatter has exactly **two** fields, `name` and `description`. No version, no
license, no `allowed-tools`, no metadata block.

```markdown
---
name: cheapest-dates
description: <clause 1> <clause 2> <clause 3> <clause 4>
---
```

The `description` is the retrieval index — it is the only thing an agent sees
when deciding whether to open your skill. Write it as four clauses, in order:

1. **What it does**, in concrete terms, naming the real parameters and response
   fields involved (`departure_date_from`, `price_insights_low`, `proxy_country`).
2. **`Use when the user asks to...`** — a dense pile of the actual phrasings a
   person would type, including synonyms and short quoted examples. This clause
   is long on purpose. Under-stuffing it is the most common reason a good skill
   never gets picked.
3. **`Returns ...`** — the output shape, using the exact field names the agent
   will see.
4. **A de-escalation clause** naming the sibling skills that own the adjacent
   jobs ("for one fixed date tracked over time use `fare-watch`"). Every skill
   needs this or the skills fight each other over the same query.

Add the non-affiliation line to the description wherever a reader could assume
otherwise — i.e. wherever you name Google or Booking.com.

## SKILL.md body

Keep the section order the other skills use:

```
# Title Case Name
## Overview            2-3 sentences, and where to get auth and endpoint detail
## Endpoints           a table: what you want -> endpoint / MCP tool
## Workflow            numbered, imperative, decision-first
## Examples            runnable curl, key read from an env var
## Common Pitfalls     real failure modes and the correct agent behavior
## Output Standards    how to present the result
```

`Common Pitfalls` and `Output Standards` are the two sections people skip and the
two that make the skill behave. They are not optional here.

## The no-invented-fields rule

**Every endpoint, parameter, filter value, and response key you write must
already exist in `skills/travel-data-api/SKILL.md`.** If it is not there, it does
not go in a skill. A hallucinated parameter is silently dropped by the API and
the agent then reports a filtered result that was never filtered.

Concretely, do not invent:

- request parameters, filter strings, or `sort_type` values;
- response fields, or nested objects the response does not have;
- endpoints, hosts, or MCP tool names;
- **uptime, latency, user counts, revenue, or customer names.** There are no
  benchmark numbers in this repo. Do not add any.

Related rules that have bitten us:

- **Never cache or reuse a fare.** Fares go stale in minutes. A skill that says
  "reuse the results from earlier in the conversation" is a bug. Always re-fetch
  and always stamp the result with when it was fetched.
- **An empty `[]` is a valid answer** — "no flights on this route and date" — and
  arrives as HTTP 200. Do not teach agents to retry it or to treat it as an error.
  Same for a hotel returning `available: false`.
- **Document known defects instead of quietly working around them.** Example:
  `sort_type` is accepted by the schema but is not forwarded on one-way searches,
  so a one-way skill must sort client-side and say so.
- **Never present a fare as booked or guaranteed.** Surface the `buy_link` and
  let the user confirm the live price on the booking page.
- **Never commit a key.** No `x-rapidapi-key` value, token, publisher id, or
  `.env` content in any file. Use `YOUR_RAPIDAPI_KEY` in examples and read from
  an env var in code. `.env` is gitignored; keep it that way.
- **Every call is the user's money.** Say what a search will cost before running
  it, not after.

## Before you open a PR

- [ ] `skills.sh.json` still parses and the new skill name appears in exactly one group.
- [ ] Frontmatter has only `name` and `description`; the description has all four clauses.
- [ ] Every field and parameter used appears in `skills/travel-data-api/SKILL.md`.
- [ ] `Common Pitfalls` and `Output Standards` are present and specific.
- [ ] No key, no cached fare, no invented metric, no claim of affiliation.

These are independent APIs returning publicly available flight and hotel pricing.
They are not affiliated with, endorsed by, or sponsored by Google or Booking.com.

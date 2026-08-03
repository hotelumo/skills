---
name: hotelumo
description: Read and manage Hotelumo hotel data with the `hotelumo` CLI — reservations and their stay dates, the guest directory, rooms and room types, rate plans and per-day prices, live availability and booking requests, webhook subscriptions and API keys, backed by the Hotelumo REST API at api.hotelumo.com. Use when asked to look up, create, update or delete Hotelumo reservations, guests, rooms or rates, to check what is bookable for a stay, to publish prices, or for anything involving the `hotelumo` command or app.hotelumo.com data.
license: Apache-2.0
compatibility: Requires Node.js 18+, network access, and a Hotelumo account — log in through the browser or with an API key (created at https://app.hotelumo.com)
metadata:
  author: Hotelumo
  version: '1.0.0'
  repository: https://github.com/hotelumo/skills
allowed-tools: Bash(hotelumo:*) Bash(npx hotelumo:*)
---

# Hotelumo CLI

`hotelumo` reads and manages the data on
[app.hotelumo.com](https://app.hotelumo.com) through the Hotelumo REST API:
hotels, reservations, guests, rooms and room types, rate plans and per-day
prices, availability, webhook subscriptions and API keys — the same records
the dashboard shows.

Run it as `hotelumo` when installed globally (`npm install -g hotelumo`) or
as `npx hotelumo` otherwise. `hotelumo --help` and `hotelumo <command> --help`
are accurate and self-sufficient; `hotelumo schema` prints the whole command
tree as JSON — prefer those over memory when unsure of a flag.
`hotelumo skills list` names the guides bundled with the installed version,
and `hotelumo skills get hotelumo` prints the copy of this guide matching it.

## Setup

No account yet? Create one without leaving the terminal — the generated
password prints exactly once, and the session is stored so every other
command works immediately. Signing up seeds a demo hotel with room types,
rooms and rate plans, so the read commands return something straight away:

```bash
hotelumo signup --email founder@example.com --json
```

```bash
hotelumo login                # interactive: choose browser or API key login
hotelumo login --browser      # opens app.hotelumo.com/cli; stores a session
hotelumo login --with-key     # masked prompt for an API key secret
hotelumo logout               # clears everything stored in ~/.hotelumo/
```

Two ways in, both stored in `~/.hotelumo/`:

- **Browser** — `hotelumo login --browser` opens
  [app.hotelumo.com](https://app.hotelumo.com) to log in with the Hotelumo
  account and stores a token session (refreshed automatically).
- **API key** — an organization administrator creates a key in the Hotelumo
  dashboard, under the hotel's **API keys** settings, or from an existing
  session with `hotelumo keys add`. The secret is shown once, at creation.
  `hotelumo login --with-key` prompts for it interactively, masked — never
  pass it as a command argument and never echo it, so it stays out of shell
  history and transcripts.

Headless agents can skip `login` entirely: export
`HOTELUMO_API_KEY=<key secret>` and every command authenticates with it at
request time. When several credentials exist the precedence is: stored
browser session, then stored key, then the environment variable — a stored
login always wins over an exported key.

Without a terminal `hotelumo login` never hangs and never opens a browser:
it exits 1 immediately with instructions on stderr. Commands run with no
credentials fail fast the same way — a `401` ending in "Not logged in?"
means run `hotelumo login`, or export `HOTELUMO_API_KEY`, then retry.

### What a key can and cannot do

An API key carries per-resource scopes: `reservations:read`,
`reservations:write`, `guests:read`, `guests:write`, `rooms:read`,
`rooms:write`, `rates:read`, `rates:write`. New keys start with the four
`:read` scopes. `403` on a scope is a permissions gap, not a bug — an
administrator widens the key. `rooms` covers rooms **and** room types;
`rates` covers rate plans, daily rates **and** the availability query.

Three surfaces are deliberately **operator-session only** and answer `401`
to an API key, because they expose credentials: `hotelumo hotels list`,
every `hotelumo webhooks` subcommand, and every `hotelumo keys` subcommand.
Use `hotelumo login --browser` for those. A single hotel is still readable
by id with a key, which is what the other commands need.

A key belongs to one organization and may additionally be pinned to one
hotel. A pinned key supplies `hotelId` implicitly, and cannot be widened to
a sibling hotel by passing `--hotelId`. Requests are rate limited per key;
a `429` means back off, not retry-loop.

## Where to start

| You want                                | Run                                                                 |
| --------------------------------------- | ------------------------------------------------------------------- |
| The hotels and their ids                | `hotelumo hotels list` (browser session)                            |
| Arrivals and departures for a window    | `hotelumo reservations list --stayDateFrom … --stayDateTo …`        |
| Find a guest by email                   | `hotelumo guests list --email <email>`                              |
| The room inventory and its state        | `hotelumo rooms list --hotelId <hotelId>`                           |
| What is sellable, and at what price     | `hotelumo rate-plans list --hotelId <hotelId>`                      |
| What is bookable for a stay             | `hotelumo availability check <hotelId> --checkInDate … --checkOutDate …` |
| Publish prices for a date range         | `hotelumo daily-rates set --ratePlanId … --dateFrom … --price …`    |
| One record, formatted                   | `hotelumo reservations read <id>` (or the guests / rooms / room-types / rate-plans equivalent) |
| The command tree as JSON                | `hotelumo schema` (or `hotelumo schema guests list` for one subtree) |

Ids are Hotelumo record ids (24-character lowercase hex) — get them from the
`list` commands, never guess them. A record owned by another organization
responds `404`, indistinguishable from a missing one; that is tenant
isolation, not an error to work around.

**Money is always in minor currency units.** `basePrice`, `price` and
`totalPrice` are integers: `12000` is `120.00` in a two-decimal currency.
Keep the returned `currency` attached and never sum across currencies.

**Dates are `YYYY-MM-DD` and checkout is exclusive.** A stay from `2026-08-01`
to `2026-08-03` is two nights and frees the room on the 3rd.

## Hotels (read-only)

```bash
hotelumo hotels list              # needs a browser session, not an API key
hotelumo hotels get <hotelId>     # raw JSON; works with a key too
hotelumo hotels read <hotelId>    # formatted: address, currency, check-in hours
```

Hotels are created at signup and edited in the dashboard — there is no
`hotels` scope and no write command here.

## Reservations

```bash
hotelumo reservations list                       # --json, -n/--limit, --skip
hotelumo reservations list --hotelId <hotelId> --guestId <guestId>
hotelumo reservations list --stayDateFrom 2026-08-01 --stayDateTo 2026-08-31
hotelumo reservations list --sortField checkInDate --sortDirection ASC
hotelumo reservations get <reservationId>
hotelumo reservations read <reservationId>
hotelumo reservations add --hotelId <hotelId> --guestId <guestId> \
  --checkInDate 2026-08-10 --checkOutDate 2026-08-12 --status CONFIRMED
hotelumo reservations add --hotelId <hotelId> \
  --body '{"rooms":[{"roomTypeId":"…","ratePlanId":"…"}]}'
hotelumo reservations update <reservationId> --status CHECKED_IN
hotelumo reservations approve <reservationId>
hotelumo reservations delete <reservationId...>  # one or more ids — permanent
```

`--status` accepts `INQUIRY`, `REQUESTED`, `CONFIRMED`, `CHECKED_IN`,
`CHECKED_OUT`, `NO_SHOW` and `CANCELED`. `--source` accepts `DIRECT`,
`BOOKING_COM`, `EXPEDIA`, `AIRBNB`, `WALK_IN`, `PHONE` and `EMAIL`.

Two filters look similar and are not: `--startTime`/`--endTime` bound the
reservation's own timestamps, while `--stayDateFrom`/`--stayDateTo` select
stays that *overlap* a calendar window (checkout after the start, check-in on
or before the end). Use the stay filters for "who is here in August".

Rooms live in the `rooms` array of the body, so a multi-room stay is created
through `--body`. A `CANCELED` reservation cannot be reopened — create a new
one. `approve` only works on a `REQUESTED` reservation, and it is what turns
a booking request from `availability book` into a real stay with assigned
rooms.

## Guests

```bash
hotelumo guests list                             # --json, --fields
hotelumo guests list --hotelId <hotelId>
hotelumo guests list --email jane@example.com    # exact match, not a search
hotelumo guests list --ids <id1>,<id2>
hotelumo guests get <guestId>
hotelumo guests read <guestId>
hotelumo guests add --hotelId <hotelId> --firstName Jane --lastName Fonda \
  --email jane@example.com --phone 5551234567
hotelumo guests update <guestId> --phone 5559876543
hotelumo guests delete <guestId...>              # one or more ids — permanent
```

Guest records are personal data. Read only what the task needs, project with
`--fields`, and never dump the whole directory into a transcript.

## Rooms and room types

```bash
hotelumo rooms list --hotelId <hotelId>          # --roomTypeId, --fields, --json
hotelumo rooms get <roomId>
hotelumo rooms read <roomId>
hotelumo rooms add --hotelId <hotelId> --roomTypeId <roomTypeId> \
  --number 101 --floor 1
hotelumo rooms update <roomId> --housekeepingStatus DIRTY
hotelumo rooms delete <roomId...>                # one or more ids — permanent
```

```bash
hotelumo room-types list --hotelId <hotelId>
hotelumo room-types get <roomTypeId>
hotelumo room-types read <roomTypeId>
hotelumo room-types add --hotelId <hotelId> --name "Double room" --maxGuests 2
hotelumo room-types update <roomTypeId> --maxGuests 3
hotelumo room-types delete <roomTypeId...>       # also deletes its rate plans
```

Room `--status` accepts `AVAILABLE`, `OUT_OF_ORDER` and `OUT_OF_SERVICE`;
`--housekeepingStatus` accepts `CLEAN`, `DIRTY` and `INSPECTED`. Prices are
**not** on the room type — they are on its rate plans. Deleting a room type
deletes its rate plans and their daily rates; the rooms of that type survive
as unassigned.

## Rate plans and daily rates

```bash
hotelumo rate-plans list --hotelId <hotelId>     # --roomTypeId
hotelumo rate-plans get <ratePlanId>
hotelumo rate-plans read <ratePlanId>
hotelumo rate-plans add --hotelId <hotelId> --roomTypeId <roomTypeId> \
  --title "Bed and breakfast" --basePrice 9000 --mealPlan bed_and_breakfast
hotelumo rate-plans update <ratePlanId> --basePrice 9500
hotelumo rate-plans delete <ratePlanId...>       # also deletes its daily rates
```

```bash
hotelumo daily-rates list --hotelId <hotelId> --ratePlanId <ratePlanId> \
  --dateFrom 2026-08-01 --dateTo 2026-08-31       # max 365 days per call
hotelumo daily-rates set --hotelId <hotelId> --ratePlanId <ratePlanId> \
  --dateFrom 2026-08-01 --dateTo 2026-08-31 --price 12000 --availability 4
hotelumo daily-rates set --hotelId <hotelId> --ratePlanId <ratePlanId> \
  --dateFrom 2026-12-24 --dateTo 2026-12-26 --stopSell
hotelumo daily-rates set --hotelId <hotelId> \
  --values '[{"ratePlanId":"…","date":"2026-08-01","price":12000}]'
hotelumo daily-rates delete <dailyRateId...>      # reverts those days to the plan's basePrice
```

`--mealPlan` accepts `room_only`, `bed_and_breakfast`, `half_board`,
`full_board` and `all_inclusive`; `--sellMode` accepts `per_room` and
`per_person`; `--rateMode` accepts `manual` and `derived`.

`daily-rates set` is a batch upsert keyed on `(ratePlanId, date)`: the flag
form expands `--dateFrom`/`--dateTo` into one value per day, and `--values`
takes the raw array when several plans or prices go in one call. **Maximum
500 values per call** — split a longer range rather than retrying. Rewriting
a day overwrites it silently, so read the range first when the user asked for
a relative change ("raise August by 10%"). Publishing rates pushes them to
connected channel managers, so a mistake reaches the OTAs.

`--hotelId` is required on `rate-plans list`, `daily-rates list` and
`daily-rates set` unless the key is pinned to one hotel.

## Availability and booking requests

```bash
hotelumo availability check <hotelId> --checkInDate 2026-08-10 \
  --checkOutDate 2026-08-12 --adults 2 --children 1
hotelumo availability book <hotelId> --roomTypeId <roomTypeId> \
  --ratePlanId <ratePlanId> --checkInDate 2026-08-10 --checkOutDate 2026-08-12 \
  --name "Jane Fonda" --email jane@example.com --phone 5551234567 \
  --quotedTotal 24000 --quotedCurrency EUR
```

`check` returns unassigned offers, not a hold — an offer is not a booking and
proves nothing about a specific guest. Each offer carries the `roomTypeId`,
`ratePlanId`, `totalPrice`, `currency` and how many rooms are left.

`book` revalidates the offer and creates a **`REQUESTED`** reservation with
one unassigned `PENDING` room for staff to approve — it is not a confirmed
stay. `--quotedTotal` and `--quotedCurrency` must match the offer exactly, so
always run `check` immediately before `book` and copy the numbers from it; a
stale quote is refused with `409`. Confirm the stay afterwards with
`hotelumo reservations approve <reservationId>`.

Stays are 1–30 nights and the check-in date must not be in the past.

## Webhooks and API keys (operator session)

```bash
hotelumo webhooks list
hotelumo webhooks get <webhookSubscriptionId>
hotelumo webhooks add --hotelId <hotelId> --url https://example.com/hooks \
  --events reservation.created,reservation.updated
hotelumo webhooks update <webhookSubscriptionId> --active true
hotelumo webhooks delete <webhookSubscriptionId...>
```

```bash
hotelumo keys list
hotelumo keys get <keyId>
hotelumo keys add --name "Channel manager" --hotelId <hotelId> \
  --scopes reservations:read,rates:read,rates:write
hotelumo keys delete <keyId...>                  # revokes the key immediately
```

Events are `reservation.created`, `reservation.updated`,
`reservation.deleted`, `guest.created`, `guest.updated` and `guest.deleted`.
Deliveries are signed with `X-Hotelumo-Signature`; an endpoint that fails
twenty times in a row is disabled automatically and is re-enabled with
`--active true`.

Both `webhooks add` and `keys add` return a **secret exactly once** — store
it at creation. Nothing can read it back afterwards. Print it only where the
user asked for it, never into a shared log.

## Writes — the rules that bite

- **Every write needs a `hotelId`** unless the API key is pinned to one hotel
  (then it is implied). `add` takes `--hotelId`; `update` reuses the stored
  record's hotel when you omit it, which costs one extra read and therefore
  needs the matching `:read` scope too.
- **`update` sends only the fields you pass**; the API merges them into the
  stored record. Arbitrary fields go through `--body '<json>'`; named options
  win over `--body` on conflict.
- **Writes fire webhooks** (`reservation.created`, `guest.updated`, …) to the
  organization's subscribed endpoints, exactly like dashboard edits — a bulk
  import loop triggers one delivery per record.
- **Deletes are permanent** and variadic. Confirm with the user and resolve
  names to ids via `list` before deleting anything.
- `add` is not idempotent — list first and check the record does not already
  exist before creating one.

## Output discipline

Every subcommand accepts `--json`: real, parseable `JSON.stringify` output on
stdout — no colors, no `util.inspect`. Without the flag, `list` and `read`
print terminal summaries. Mutations with `--json` print a small result object
— `{ "ok": true, "id": "…" }` for `add`/`update`,
`{ "ok": true, "deleted": ["…"] }` for `delete`.

Failures always go to stderr with exit code 1; with `--json` the error is a
single JSON line, `{"error":{"message":"…","status":404}}`, so stdout stays
parseable.

When exploring, project instead of dumping — use `--fields` server-side or
pipe `--json` through `jq` and keep only what you need:

```bash
hotelumo reservations list --json -n 100 \
  | jq '.[] | {id: ._id, checkInDate, checkOutDate, status}'
```

## Schema

```bash
hotelumo schema               # the whole command tree as JSON
hotelumo schema guests list   # one subtree: its options and arguments
```

Every command, option and positional argument as parseable JSON — use it to
discover capabilities instead of scraping `--help` text.

## Skills

```bash
hotelumo skills list           # bundled agent guides
hotelumo skills get hotelumo   # this guide, matching the installed version
```

## Configuration

`HOTELUMO_API_URL` overrides the API endpoint (default
`https://api.hotelumo.com`) — only for non-production deployments.
`HOTELUMO_API_KEY` supplies an API key secret from the environment for
headless use — no login step needed, but a stored login outranks it.
Stored credentials live only in `~/.hotelumo/`.

## Errors → remediation

| Symptom                                     | Fix                                                        |
| ------------------------------------------- | ---------------------------------------------------------- |
| `401` / "Not logged in?"                    | `hotelumo login`, or export `HOTELUMO_API_KEY`, then retry |
| `401` on hotels list, webhooks or keys      | Operator-session only — `hotelumo login --browser`         |
| `403: Invalid api key`                      | The stored secret is wrong — `hotelumo login` again        |
| `403` naming a scope                        | Widen the key's scopes in the Hotelumo dashboard           |
| `404` on an id you were given               | Wrong id, or the record is another organization's          |
| `400: missing hotelId`                      | Pass `--hotelId` (see Writes above)                        |
| `409` on `availability book`                | The offer changed — run `availability check` again first   |
| `409` on `reservations approve`             | Only a `REQUESTED` reservation can be approved             |
| `400: Batch too large` on `daily-rates set` | Over 500 values — split the date range                     |
| `429`                                       | Per-key rate limit — wait, do not retry-loop               |

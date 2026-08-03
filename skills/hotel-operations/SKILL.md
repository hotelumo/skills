---
name: hotel-operations
description: Review a Hotelumo property's safe operational configuration and capacity — property metadata, bookable availability, aggregate schedules, room and housekeeping status, rate plans, daily rates, and extras. Use for read-only hotel operations. Never use it for guest records, individual reservations or allocations, payments, access codes, credentials, or booking and pricing changes.
license: Apache-2.0
compatibility: Requires the Hotelumo MCP connector (https://mcp.hotelumo.com/mcp) and a Hotelumo account with at least one hotel
metadata:
  author: Hotelumo
  version: '1.0.0'
  repository: https://github.com/hotelumo/skills
---

# Hotel operations with Hotelumo

This skill drives the hosted Hotelumo MCP connector. The user must connect
Hotelumo and authorize an account that can access the intended organization.
If authorization fails, ask the user to connect or reauthorize; do not guess
identifiers or retry with another tenant.

## Hard privacy and commerce boundary

This connector intentionally does not expose:

- guest identities, contact details, addresses, government-document details,
  notes, documents or private profile data;
- individual reservation records, exact individual stays, room allocations,
  occupant identifiers or access codes;
- payments, cards, processor data, transaction errors, folios or invoices;
- staff records, organization internals, channel/integration credentials,
  provider secrets or arbitrary metadata; or
- tools that book, cancel, check in, check out, assign rooms, change inventory
  or rates, take payments, message guests, or mutate any Hotelumo record.

Never ask the user to paste excluded data into chat, infer a person's stay from
aggregate counts, or imply that a missing write can be approximated with
another tool. Availability is an unassigned offer, not evidence that a person
has or will make a reservation.

## Available tools

| Tool                         | Use it for                                                           |
| ---------------------------- | -------------------------------------------------------------------- |
| `list_hotels`                | Resolve organization-owned hotel IDs and safe metadata               |
| `get_hotel`                  | Read safe metadata for one owned hotel                               |
| `get_hotel_availability`     | Review up to 100 bookable offers for a stay of at most 30 nights     |
| `get_hotel_schedule_summary` | Count arrivals, departures, in-house stays and statuses for 31 days  |
| `list_room_types`            | Page through safe room-type capacity and names                       |
| `get_room_type`              | Read one owned room type without free-text/private fields            |
| `list_rooms`                 | Review safe room and housekeeping status without internal room IDs   |
| `list_rate_plans`            | Page through sell modes, meal modes and bounded catalog prices       |
| `list_daily_rates`           | Review up to 90 days of bounded prices, availability and stay limits |
| `list_extras`                | Page through safe names, prices, tax and sale modes for extras       |
| `show_hotel_overview`        | Render a bounded aggregate hotel-operations overview                 |

Every tool is read-only and requires the Hotelumo read scope. There is no
connector operation for booking, reservation lookup, guest lookup, payment,
rate updates, room assignment, check-in, check-out, messaging or deletion.

## Identifiers, pagination and bounds

Never guess a hotel ID. Start with `list_hotels`, choose the hotel that matches
the user's request, and reuse its returned 24-character lowercase hexadecimal
ID. If several hotels match, ask which one they mean. Resolve room-type IDs
from `list_room_types` before using `get_room_type`.

Listing tools use bounded limits and offsets. Keep the same hotel and filters
when requesting the next page. A result marked with `hasMore`, or a count shown
with a plus sign, is a lower bound from a bounded read; do not report it as a
complete total. Do not interpret identifiers as meaningful business codes.

## Hotel overview workflow

When the user wants a broad operational snapshot:

1. Call `list_hotels`.
2. Resolve the intended hotel without guessing.
3. Call `show_hotel_overview` with that hotel ID.
4. Interpret only safe property metadata, aggregate next-seven-day reservation
   status counts, room/housekeeping status counts and bounded catalog counts.

The overview's reservation count is a bounded overlap sample, not a list of
stays and not necessarily a complete total.

## Schedule workflow

For workload over a period:

1. Resolve the hotel with `list_hotels`.
2. Call `get_hotel_schedule_summary` with ISO calendar dates. Omit both dates
   for the default seven-day range. Never request more than 31 days.
3. Report only aggregate status, arrival, departure and in-house counts.
4. Treat checkout as exclusive for in-house occupancy. A reservation overlaps
   the requested range only when checkout is after the range start and check-in
   is on or before the range end.

The connector fetches a bounded reservation sample. When the result indicates
more rows, every aggregate is a lower bound. Never try to reconstruct an
individual stay or identify a guest from these counts.

## Availability and pricing

Use `get_hotel_availability` only for unassigned offers. Provide valid ISO
check-in/check-out dates for 1–30 nights and occupant counts in the supported
range. The tool returns safe room-type/rate-plan labels, availability, nightly
rates and totals; it does not make or hold a booking.

Prices from availability, rate plans, daily rates and extras are integer minor
currency units. For a two-decimal currency, 12000 means 120.00. Keep the
returned currency attached and do not assume every currency has two decimals.
Do not sum values across different currencies.

## Rooms, rates and extras

Use `list_rooms` for operational room and housekeeping states, not for guest
or allocation lookup. Internal room IDs are intentionally absent.

Use `list_rate_plans`, `list_daily_rates` and `list_extras` for safe catalog
review. Follow pagination only as far as the user's question requires. Catalog
data does not prove that a guest bought an offer or extra.

## Unsupported requests

For guest, individual reservation/allocation, payment, access-code,
credential, staff or private-integration requests, state that the public
connector cannot access that data. For booking, cancellation, room assignment,
check-in/out, rate changes, payment or any other write, state that every
available tool is read-only and direct the user to Hotelumo itself.

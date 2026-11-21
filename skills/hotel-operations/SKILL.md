---
name: hotel-operations
description: Review a Hotelumo property's safe operational configuration and capacity — property metadata, bookable availability, aggregate schedules, room and housekeeping status, rate plans, daily rates, extras and stock — and correct a stock count. Use when asked about room availability and prices for given dates, arrivals, departures and how busy the hotel is, room and housekeeping status, rate plans, daily rates, extras, or how many towels, linen or other supplies are left, or to update a stock count. Never use it for guest records, individual reservations or allocations, payments, access codes, credentials, or booking and pricing changes.
license: Apache-2.0
compatibility: Requires the Hotelumo MCP connector (https://mcp.hotelumo.com/mcp) and a Hotelumo account with at least one hotel
metadata:
  author: Hotelumo
  version: '1.1.0'
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
- tools that book, cancel, check in, check out, assign rooms, change room
  availability or rates, take payments, message guests, or change any Hotelumo
  record other than a stock count.

Never ask the user to paste excluded data into chat, infer a person's stay from
aggregate counts, or imply that a missing write can be approximated with
another tool. Availability is an unassigned offer, not evidence that a person
has or will make a reservation.

## Available tools

| Tool                         | Use it when someone asks…                                                      |
| ---------------------------- | ------------------------------------------------------------------------------ |
| `list_hotels`                | which hotels they manage; also resolves the hotel ID every other tool needs    |
| `get_hotel`                  | about one hotel's default check-in/check-out times, currency or timezone       |
| `get_hotel_availability`     | whether rooms are free for a stay (1–30 nights) and what it would cost         |
| `get_hotel_schedule_summary` | how busy each day is: arrivals, departures, in-house stays (at most 31 days)   |
| `list_room_types`            | what kinds of rooms the hotel has and how many guests each sleeps              |
| `get_room_type`              | about one room type's name and maximum guests                                  |
| `list_rooms`                 | which rooms are clean, dirty, inspected, out of order or out of service        |
| `list_rate_plans`            | which rate plans are sold, with meal plan, sell mode and base price            |
| `list_daily_rates`           | what a rate costs on given nights, and its minimum stay or stop-sell days      |
| `list_extras`                | which extras are sold (breakfast, parking, spa…) and their prices              |
| `list_inventory_items`       | how many towels, sheets, toiletries or other supplies are left, or what is low |
| `show_hotel_overview`        | for a one-card overview of one hotel's operations                              |
| `set_inventory_quantity`     | to correct or update a stock count, after counting or a delivery               |

Every tool but `set_inventory_quantity` is read-only and requires the Hotelumo
read scope; `set_inventory_quantity` also requires the write scope. There is no
connector operation for booking, reservation lookup, guest lookup, payment,
rate updates, room assignment, check-in, check-out, messaging or deletion.

If `set_inventory_quantity` is missing, the connection was authorized to read
only (every connection made before November 2026 was). Ask the user to
disconnect Hotelumo and connect it again, allowing Hotelumo to create and
update data.

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
   Arrivals, departures and in-house counts cover confirmed, checked-in and
   checked-out stays; the status totals include every status, cancellations
   and no-shows too.
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
or allocation lookup. A room status of `AVAILABLE` means in service, not
unoccupied; use `get_hotel_availability` for free rooms on given dates.
Internal room IDs are intentionally absent.

Use `list_rate_plans`, `list_daily_rates` and `list_extras` for safe catalog
review. `list_daily_rates` returns only the dates with a stored daily rate; any
other date sells at the plan's base price from `list_rate_plans`. Follow
pagination only as far as the user's question requires. Catalog data does not
prove that a guest bought an offer or extra.

## Stock

`list_inventory_items` reads one hotel's supplies (towels, linen, toiletries,
minibar and cleaning stock): name, unit, quantity, minimum quantity, a
low-stock flag and each item's `lastEditTime`. It returns up to 100 items, 50
by default, and `hasMore` when there are more; `lowStockOnly` keeps only items
at or below their minimum, and `search` matches item names. It never returns an
item's notes. This is a hotel's own stock of supplies, not the rooms it can
sell: for free rooms on given dates use `get_hotel_availability`.

To correct a count:

1. Read the item with `list_inventory_items` and keep its id and
   `lastEditTime`.
2. Work out the new total. `set_inventory_quantity` replaces the stored
   quantity with the number given; it does not add to it. When the user reports
   a change ("we used 6 towels", "12 arrived"), compute the total from the
   quantity just read.
3. Confirm the item, the hotel and the new total with the user, then call
   `set_inventory_quantity` with `expectedLastEditTime` set to the
   `lastEditTime` read.
4. A `conflict` means the item changed since it was read: read it again, show
   the user the stored quantity, and try again once they confirm. After an
   unknown delivery, read the item before retrying: a saved count shows in its
   new `lastEditTime`.

The write runs as the signed-in member, with that member's Hotelumo role: a
role without stock access is refused. It cannot add, rename or delete a stock
item; direct the user to Inventory in the Hotelumo app for that.

## Unsupported requests

For guest, individual reservation/allocation, payment, access-code,
credential, staff or private-integration requests, state that the public
connector cannot access that data. For booking, cancellation, room assignment,
check-in/out, rate changes, payment or any write other than a stock count,
state that the connector cannot make that change and direct the user to
Hotelumo itself.

# 01 – Product Scope

> **Status:** Draft for approval · **Applies to:** v1
> **Related:** [02 – Architecture](02-architecture.md) · [03 – Data model](03-data-model.md) · [04 – API spec](04-api-spec.md)

This document defines **what** Ipamo v1 does and why. It describes business rules in product terms. Their exact data and API form lives in docs 03 and 04, and if those docs ever disagree with this one, this one wins until it is changed.

---

## 1. Problem

People who rent out equipment on the side (A/V, DJ gear, lighting, staging, event décor) usually track it in their heads, a notes app or a spreadsheet. That breaks down quickly:

- **Double bookings.** The same speaker is promised to two people for the same weekend.
- **Lost or short returns.** Twenty cables went out and eighteen came back, and nobody noticed until the next gig.
- **Forgotten returns.** Nothing says "this was due back three days ago."
- **Condition disputes.** There's no record of what state an item was in when it left.
- **Fee guesswork.** Day rates, deposits and refunds get worked out by hand each time.

## 2. Users

| User | Description | Has an account? |
| --- | --- | --- |
| **Owner** | Owns equipment and rents it out. Manages their own inventory, contacts and rentals. | Yes |
| **Contact (renter)** | Someone who rents from an owner. Exists only as a record in that owner's data. | No |

There is **no admin role** in v1. Registration requires an invite code set in configuration, which keeps the user base to the owner's circle.

**Example owners:**
- *A/V owner:* a DJ who rents out speakers, mixers, lighting and large quantities of cables and stands.
- *Décor owner:* rents backdrops, centrepieces, chair covers and arches, mostly as bulk stock.

**Expected scale:** fewer than 10 owners, each with up to a few hundred equipment records and a few hundred rentals a year. The design should be correct and maintainable at this scale, without premature optimization.

## 3. Key concepts

| Term | Meaning |
| --- | --- |
| **Equipment** | Something an owner rents out. It is either *Individual* or *Bulk*. |
| **Individual equipment** | A single physical unit, usually with a serial number (e.g. a mixer). Quantity is always 1. |
| **Bulk equipment** | Interchangeable stock counted by quantity (e.g. 20 XLR cables). Has a *quantity owned*. |
| **Category** | An owner-defined grouping (Audio, Lighting, Décor…). New owners start with an editable default list. |
| **Contact** | A renter in the owner's records. |
| **Rental** | One booking for one contact over a date range. It contains one or more *rental items*. |
| **Rental item** | One equipment record within a rental, with a quantity (always 1 for Individual equipment). |
| **Day** | Rentals are booked in whole calendar days. There are no times of day. |
| **Today** | The current date in the owner's time zone (default `America/Edmonton`). |

## 4. Rental lifecycle

```text
            check out                 return all items
Reserved ─────────────► CheckedOut ─────────────────────► Returned
   │
   │ cancel
   ▼
Cancelled
```

- **Reserved.** Created with a start date and a due date. Blocks availability for those days. The start date may be today or in the future.
- **CheckedOut.** The renter has the gear. Recorded on or after the start date, together with each item's condition when it went out.
- **Returned.** Every rental item has been returned. Items can come back on different days, and the rental moves to Returned when the last one does.
- **Cancelled.** Only a *Reserved* rental can be cancelled. Cancelling frees its days immediately. Nothing is ever deleted.
- **Editing.** A Reserved rental can be edited freely: dates, contact, items, quantities, deposit and notes. A CheckedOut rental can only have its due date changed (an extension or an earlier return date) and its notes. See BR-25 and BR-26.

**Pickup late.** A Reserved rental whose start date has passed without checkout is shown as *pickup late* in the active list. The owner then either checks it out or cancels it. It keeps blocking its days until then.

## 5. Business rules

Each rule has an ID so that tests and docs can refer to it.

### Availability and booking

- **BR-01 – Whole days, inclusive.** A rental from Dec 1 to Dec 5 occupies Dec 1, 2, 3, 4 and 5. The due date must be on or after the start date.
- **BR-02 – No overlap (Individual).** An Individual item can belong to only one active rental (Reserved or CheckedOut) on any given day. If a rental is due back Dec 5, the item can next be booked from **Dec 6**.
- **BR-03 – No over-commitment (Bulk).** On any given day, the total quantity of a Bulk item across active rentals can't exceed its quantity owned.
- **BR-04 – Overdue items stay blocked.** An item that is checked out and not yet returned stays occupied from its start date until it actually comes back, even after the due date has passed. A late return can therefore collide with someone else's later reservation. That collision is **flagged, never auto-resolved** (see BR-14).
- **BR-05 – Early return frees days.** Once an item is returned, its days after the return date become available again. For a Bulk item this frees only the quantity that actually came back.
- **BR-06 – No past bookings.** A new rental's start date can't be before today.
- **BR-07 – Valid lines.** A rental has at least one item. The same equipment can't appear twice in one rental; you raise the quantity on its existing line instead. Individual items always have quantity 1, and Bulk quantities must be at least 1.
- **BR-08 – Active records only.** Archived equipment and archived contacts can't be used in new rentals.

### Checkout and return

- **BR-09 – Checkout window.** Only a Reserved rental can be checked out, and only on or after its start date and no later than today. Each item's condition when it goes out is recorded at checkout.
- **BR-10 – Return rules.** Only a CheckedOut rental's items can be returned. Each item can be returned **once** (no double returns). The return date can't be before the checkout date or after today, and each returned item's condition when it comes back is recorded.
- **BR-11 – Bulk shortfall.** When a Bulk item is returned, the owner records the quantity that came back, between 0 and the quantity rented. Any shortfall is recorded as missing. Missing stock does **not** automatically reduce the quantity owned; the owner adjusts that deliberately.
- **BR-12 – Rental completes.** A rental becomes Returned when all of its items have been returned.

### Overdue

- **BR-13 – Overdue definition.** A rental item is overdue when its rental is CheckedOut, the item hasn't been returned, and the due date is before today. A rental is overdue if any of its items are.
- **BR-14 – Conflict warning.** The overdue list shows any future reservations that would collide with the overdue items if they're still out on those reservations' start dates.

### Money

- **BR-15 – Day rate snapshot.** Each piece of equipment has a default day rate. When a rental is created, that rate is copied onto the rental item, so later price changes never alter existing rentals.
- **BR-16 – Fee calculation.** A rental item's fee is day rate × number of rental days (inclusive, per BR-01) × quantity. The rental's subtotal is the sum of its item fees.
- **BR-17 – Adjustments, not automatic late fees.** Late days are reported, not charged automatically. The owner can record one fee adjustment on a rental (positive or negative, with a note) to cover late fees, damage or discounts. Total = subtotal + adjustment.
- **BR-18 – Deposit.** A rental can have a deposit amount. When the rental completes, the owner can record how much was refunded, between 0 and the deposit amount.
- **BR-19 – Currency.** All amounts for an owner are in that owner's currency (default CAD) and are never converted.

### Records and ownership

- **BR-20 – No hard deletes.** Equipment, contacts and categories are archived, not deleted. Rentals are cancelled, not deleted. History is never lost.
- **BR-21 – Archive guard.** Equipment or a contact can't be archived while it's part of a Reserved or CheckedOut rental.
- **BR-22 – Quantity guard.** A Bulk item's quantity owned can't be lowered below the quantity already committed to active rentals on any day from today onward.
- **BR-23 – Unique names per owner.** Category names are unique per owner, ignoring case. An equipment serial number, when given, is unique per owner.
- **BR-24 – Strict data isolation.** An owner can see and change only their own records. Any request for another owner's record behaves exactly as if the record doesn't exist.

### Editing rentals

- **BR-25 – Editing a reservation.** While a rental is Reserved, the owner can change its contact, start date, due date, items, quantities, deposit and notes. The edited rental must pass BR-01, BR-02, BR-03, BR-06, BR-07 and BR-08 as if it were new, except that it never conflicts with its own current booking. Items already on the rental keep their snapshotted day rate (BR-15); newly added items snapshot the current day rate. An edit either succeeds completely or changes nothing.
- **BR-26 – Changing the due date while out.** While a rental is CheckedOut, the owner can change its due date (and notes), but nothing else. An extension must not create a conflict under BR-02 or BR-03 for items that are still out. The new due date can't be before the checkout date. If the change makes an unreturned item overdue, it immediately counts as overdue (BR-13). Fees are recalculated from the new due date (BR-16).

## 6. v1 features

1. **Accounts:** register (with invite code), log in, refresh session, log out. View and update profile settings (display name, time zone, currency).
2. **Categories:** list, create, rename and archive. Defaults are seeded on registration.
3. **Equipment:** create, view, list (with search, category filter and paging), update and archive. Supports both Individual and Bulk tracking.
4. **Availability:** for a date range, list equipment with the quantity available.
5. **Contacts:** create, view, list (search and paging), update and archive.
6. **Rentals:** create a reservation, view, list (filtered by status, contact or date), edit a reservation, change the due date while out, check out, return items, cancel, and record a fee adjustment and deposit refund.
7. **Active and overdue lists:** dedicated views of what's out now, what's late (with conflict warnings), and pickups that are late.
8. **API docs:** an OpenAPI document and Swagger UI in Development.
9. **Health check:** a `/health` endpoint for local checks and, later, Azure.

## 7. Out of scope for v1

| Item | Why it's deferred | Planned for |
| --- | --- | --- |
| Payments and invoicing | Adds compliance and integration work beyond the learning goals | Not planned |
| Photo uploads | Needs Blob Storage | Later (Azure Blob) |
| Photo-to-form AI | Depends on photos and an AI provider | Later |
| Email or SMS reminders | Needs email infrastructure and a scheduler | Later (Azure Function) |
| Shared inventory and teams | Changes the ownership model throughout | Not planned |
| Web or mobile front end | v1 is API-first | Later (React + TypeScript) |
| Password reset by email | No email in v1; resets are done manually | With reminders |
| Hourly or partial-day rentals | Rentals are day-based by design | Not planned |
| Multi-currency per owner | One currency per owner is enough | Not planned |
| Azure deployment | Local and CI first | Right after v1 |

**Extension points v1 must leave room for, without building them:**
- Equipment can later gain photos and AI-suggested fields without changing its core columns. The AI's output will map onto the existing create-equipment request, using the owner's own categories.
- Rental data supports a scheduled "due tomorrow / overdue" query that a reminder function can call later.
- The API is shaped for a front end: consistent error format, paging and CORS configuration ready.

## 8. Success criteria

v1 is done when all of the following are true.

**Functional**
- Every v1 feature in §6 works through the API and is documented in Swagger.
- Every business rule BR-01 to BR-26 has at least one automated test named after its ID.

**Quality**
- `dotnet test` passes locally and in GitHub Actions on every push to `main`.
- At least 80% line coverage on business-logic code. The target is the rules, not boilerplate.
- No compiler warnings, nullable reference types on, and analyzer warnings treated as errors.

**Security**
- Every endpoint that reads or changes a record has an integration test proving a second owner gets `404`.
- No secrets in the repository, and passwords are never stored or logged in plain text.

**Practical**
- The owner uses Ipamo for real rentals for at least two weeks without falling back to a spreadsheet.
- A new developer can clone the repo and get it running on a Mac by following the README alone.

**Learning (personal)**
- The owner can explain, without notes: dependency injection and service lifetimes, the middleware pipeline, how EF Core tracks changes and generates migrations, how a JWT is issued and validated, and how the integration tests spin up the API and database.

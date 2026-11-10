# 04 – API Specification

> **Status:** Draft for approval · **Applies to:** v1
> **Related:** [01 – Product scope](01-product-scope.md) (BR-xx) · [03 – Data model](03-data-model.md) (§5 calculations) · [05 – Auth & security](05-auth-and-security.md)

This document defines every v1 endpoint: its request and response shapes, status codes and errors. The running app's OpenAPI document (`/openapi/v1.json`) must match it. If they differ, one of them gets fixed in the same pull request.

---

## 1. Conventions

### 1.1 Basics

| Topic | Rule |
| --- | --- |
| Base path | `/api` |
| Format | JSON (UTF-8). Request bodies require `Content-Type: application/json`. |
| Property names | `camelCase` |
| IDs | UUID strings, e.g. `"0193a4f2-7c1e-7b3a-9d2e-5f8a1c2b3d4e"` |
| Calendar dates | `"YYYY-MM-DD"` (`DateOnly`) |
| Instants | ISO 8601 in UTC, e.g. `"2026-12-01T17:04:22Z"` |
| Money | A JSON number with up to 2 decimal places, e.g. `45.00`. The currency comes from the owner's profile or the rental's `currencyCode`, and is never repeated per amount. |
| Enums | camelCase strings: `"individual"`, `"bulk"`, `"checkedOut"`, `"excellent"`… |
| Nulls | Optional fields that have no value are returned as `null`, not omitted, so response shapes stay stable. |
| Auth | `Authorization: Bearer <accessToken>` on every endpoint except those marked **Public** |
| "Today" | The current date in the owner's time zone (doc 03 §5) |

### 1.2 Paging, search and sorting

List endpoints that can grow large accept:

| Query | Default | Rules |
| --- | --- | --- |
| `page` | `1` | ≥ 1 |
| `pageSize` | `25` | 1–100 |
| `q` | none | Case-insensitive "contains" search on the listed fields, max 100 characters |
| `sort` | per endpoint | One field name, prefixed with `-` for descending (e.g. `-dueDate`). Unknown fields → `400`. |
| `includeArchived` | `false` | Where the resource can be archived |

Paged response envelope:

```json
{
  "items": [ ],
  "page": 1,
  "pageSize": 25,
  "totalCount": 42,
  "totalPages": 2
}
```

### 1.3 Status codes

| Code | Meaning in Ipamo |
| --- | --- |
| `200 OK` | Success with a body |
| `201 Created` | Resource created. The body is the new resource, and a `Location` header points to it. |
| `204 No Content` | Success with no body |
| `400 Bad Request` | **Malformed input**: invalid JSON, a missing required field, wrong type, out-of-range value, too-long text, or an unknown sort field. Checked before any business logic runs. |
| `401 Unauthorized` | Missing, invalid or expired access token, or failed login or refresh |
| `403 Forbidden` | Authenticated or well-formed, but not allowed (only used for an invalid invite code) |
| `404 Not Found` | The ID in the **route** doesn't exist **or belongs to another owner** (BR-24). The response is identical in both cases. |
| `409 Conflict` | The request clashes with **other data**: availability, a duplicate name or serial, a concurrent change, or an active rental blocking an archive |
| `422 Unprocessable Content` | Well-formed, but breaks a **business rule** about this resource's own state or dates (BR-xx), or references an ID in the **body** that doesn't exist for this owner |
| `429 Too Many Requests` | Rate limit hit (auth endpoints). Includes a `Retry-After` header. |
| `500 Internal Server Error` | Unexpected failure. Generic message only; details go to the server logs. |

### 1.4 Error format

Every error is RFC 9457 **ProblemDetails** (`Content-Type: application/problem+json`):

```json
{
  "type": "https://ipamo.dev/errors/rental.unavailable",
  "title": "Some items are not available for those dates.",
  "status": 409,
  "detail": "1 item does not have enough quantity available between 2026-12-06 and 2026-12-08.",
  "instance": "/api/rentals",
  "code": "rental.unavailable",
  "ruleId": "BR-03",
  "traceId": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "unavailable": [
    { "equipmentId": "0193…", "requested": 20, "available": 12 }
  ]
}
```

- `code` is **stable** and meant for clients to act on. `title` and `detail` are for humans and may change.
- `ruleId` is included when a business rule from doc 01 was broken.
- `traceId` matches the server log entry.
- Validation errors (`400`) use `code: "validation.failed"` and an `errors` object of field → messages:

```json
{
  "status": 400,
  "title": "One or more validation errors occurred.",
  "code": "validation.failed",
  "errors": {
    "items[0].quantity": ["Quantity must be at least 1."],
    "dueDate": ["The dueDate field is required."]
  }
}
```

A full list of error codes is in §9.

### 1.5 Concurrency

- Booking, editing, checkout and return operations lock the equipment rows they touch, so simultaneous bookings and returns can't both succeed. See doc 02 §4. A request that loses the race gets `409` with `code: "concurrency.conflict"` and can safely be retried.
- Simple field edits (e.g. two browser tabs editing the same contact) are **last write wins** in v1. ETag/`If-Match` support can be added when the front end arrives, without breaking clients.

---

## 2. Auth (`/api/auth`)

Token lifetimes, hashing, rotation and rate limits are defined in [05 – Auth & security](05-auth-and-security.md).

**Token response** (shared by register, login and refresh):

```json
{
  "accessToken": "eyJhbGciOi…",
  "accessTokenExpiresAt": "2026-12-01T17:19:22Z",
  "refreshToken": "q3Nw…(opaque, 64 random bytes, base64url)",
  "refreshTokenExpiresAt": "2026-12-15T17:04:22Z",
  "owner": { "id": "0193…", "email": "stephen@example.com", "displayName": "Stephen" }
}
```

### POST `/api/auth/register` · Public · rate-limited

Creates a user and an owner profile, seeds the default categories, and logs the new user in.

| Field | Type | Required | Rules |
| --- | --- | --- | --- |
| `email` | string | yes | Valid email, ≤ 254 characters |
| `password` | string | yes | 12–128 characters (doc 05) |
| `displayName` | string | yes | 1–80 characters |
| `inviteCode` | string | yes | Must match configuration |
| `timeZone` | string | no | IANA ID. Default `America/Edmonton`. |
| `currencyCode` | string | no | 3 uppercase letters. Default `CAD`. |

**201** Token response, with `Location: /api/profile`.

Errors: `400 validation.failed`, `400 auth.weak_password`, `403 auth.invalid_invite`, `409 auth.email_taken`, `429`.

### POST `/api/auth/login` · Public · rate-limited

Body: `{ "email": "…", "password": "…" }`

**200** Token response.

Errors: `400 validation.failed`, `401 auth.invalid_credentials`, `429`. The same `401` is returned for an unknown email, a wrong password or a locked-out account, so attackers can't use it to discover accounts.

### POST `/api/auth/refresh` · Public · rate-limited

Body: `{ "refreshToken": "…" }`

**200** A new token response. The old refresh token stops working immediately (rotation).

Errors: `400 validation.failed`, `401 auth.invalid_refresh_token`. If a refresh token that was already used is presented again, the whole session family is revoked and this same error is returned (doc 05).

### POST `/api/auth/logout` · Public

Body: `{ "refreshToken": "…" }`

**204** always, even if the token is unknown, so the response doesn't reveal anything. Revokes the token's family. The current access token stays valid until it expires (at most 15 minutes).

---

## 3. Profile (`/api/profile`)

**Profile response:**

```json
{ "id": "0193…", "email": "stephen@example.com", "displayName": "Stephen",
  "timeZone": "America/Edmonton", "currencyCode": "CAD" }
```

### GET `/api/profile`
**200** Profile response.

### PUT `/api/profile`
Body: `{ "displayName": "…", "timeZone": "…", "currencyCode": "…" }`. All fields are required; this is a full replace.

**200** Profile response.

Errors: `400 validation.failed`, `422 profile.invalid_time_zone`.

Changing `currencyCode` only affects **new** rentals (doc 03, `rentals.currency_code`).

### POST `/api/profile/password`
Body: `{ "currentPassword": "…", "newPassword": "…" }`

**204.** Revokes all of this owner's refresh tokens, which logs out other sessions.

Errors: `400 validation.failed`, `400 auth.weak_password`, `422 auth.wrong_password`.

---

## 4. Categories (`/api/categories`)

**Category response:** `{ "id": "…", "name": "Audio", "archivedAt": null }`

### GET `/api/categories?includeArchived=false`
**200** An array of categories sorted by name. Not paged, because the lists are short.

### POST `/api/categories`
Body: `{ "name": "Draping" }` (1–60 characters, trimmed)

**201** Category.

Errors: `400`, `409 category.name_taken` (BR-23, case-insensitive, archived categories included).

### PUT `/api/categories/{id}`
Body: `{ "name": "…" }`

**200** Category.

Errors: `400`, `404`, `409 category.name_taken`.

### POST `/api/categories/{id}/archive` · POST `/api/categories/{id}/restore`
**200** Category. Both are idempotent: archiving an archived category is a no-op that returns `200`. Archiving is allowed even while equipment uses the category; that equipment keeps it.

Errors: `404`.

---

## 5. Equipment (`/api/equipment`)

**Equipment summary** (lists):

```json
{ "id": "…", "name": "QSC K12.2 Speaker", "categoryId": "…", "categoryName": "Audio",
  "trackingType": "individual", "serialNumber": "GDB123456", "quantityOwned": 1,
  "condition": "excellent", "dayRate": 45.00, "archivedAt": null }
```

**Equipment detail** = the summary plus `replacementValue`, `notes`, `createdAt` and `updatedAt`.

### GET `/api/equipment`
Query: `q` (searches name and serial number), `categoryId`, `trackingType`, `includeArchived`, `page`, `pageSize`, `sort` (`name` default, `-name`, `createdAt`, `-createdAt`, `dayRate`, `-dayRate`).

**200** Paged equipment summaries.

### GET `/api/equipment/{id}`
**200** Equipment detail.

Errors: `404`.

### GET `/api/equipment/available?from=&to=`
Lists **non-archived** equipment with the quantity available across the whole range (doc 03 §5.2).

| Query | Required | Rules |
| --- | --- | --- |
| `from`, `to` | yes | Dates. `from ≤ to`, and the range can be at most 366 days long. |
| `categoryId`, `q` | no | Filters |
| `onlyAvailable` | no | Default `true`, which hides items with 0 available |
| `page`, `pageSize` | no | Sorted by name |

**200** Paged:

```json
{ "items": [ { "equipmentId": "…", "name": "XLR Cable 25ft", "categoryName": "Cables & Accessories",
               "trackingType": "bulk", "quantityOwned": 20, "quantityAvailable": 12 } ],
  "page": 1, "pageSize": 25, "totalCount": 1, "totalPages": 1 }
```

Errors: `400 validation.failed` (missing or invalid dates, `from > to`, range too long).

> **Routing note:** `/available` is declared before `/{id:guid}`, and the `:guid` route constraint guarantees "available" is never parsed as an ID.

### POST `/api/equipment`
`CreateEquipmentRequest`. This is also the shape the future photo-to-form AI will pre-fill (doc 02 §8).

| Field | Type | Required | Rules |
| --- | --- | --- | --- |
| `name` | string | yes | 1–120 characters |
| `categoryId` | uuid | yes | An active category of this owner |
| `trackingType` | enum | yes | `individual` or `bulk`. Can't be changed later. |
| `serialNumber` | string | no | ≤ 100 characters. Individual items only. |
| `quantityOwned` | int | no | Individual: omitted or `1`. Bulk: required, 1–100000. |
| `condition` | enum | no | Default `good` |
| `replacementValue` | number | no | 0–99,999,999.99, per unit |
| `dayRate` | number | yes | 0–99,999,999.99 |
| `notes` | string | no | ≤ 2000 characters |

**201** Equipment detail.

Errors: `400 validation.failed`, `422 equipment.category_not_found`, `422 equipment.category_archived`, `422 equipment.serial_not_allowed` (serial number on a Bulk item), `422 equipment.invalid_quantity` (Individual ≠ 1, or Bulk without a quantity), `409 equipment.serial_taken` (BR-23).

### PUT `/api/equipment/{id}`
Same fields as create **except** `trackingType`. This is a full replace.

**200** Equipment detail.

Errors: as for create, plus `404`, `422 equipment.archived` (restore it first), and `409 equipment.quantity_below_committed` (BR-22). That last error includes `{ "minimumQuantity": 14 }`.

### POST `/api/equipment/{id}/archive`
**200** Equipment detail. Idempotent.

Errors: `404`, `409 equipment.in_active_rental` (BR-21). That error includes `{ "rentalIds": ["…"] }`.

### POST `/api/equipment/{id}/restore`
**200** Equipment detail. Idempotent.

Errors: `404`.

---

## 6. Contacts (`/api/contacts`)

**Contact summary:** `{ "id", "fullName", "email", "phone", "organization", "archivedAt" }`
**Contact detail** = the summary plus `notes`, `createdAt` and `updatedAt`.

### GET `/api/contacts`
Query: `q` (searches name, email, phone and organization), `includeArchived`, `page`, `pageSize`, `sort` (`fullName` default, `-fullName`, `createdAt`, `-createdAt`).

**200** Paged contact summaries.

### GET `/api/contacts/{id}`
**200** Contact detail.

Errors: `404`.

### POST `/api/contacts` · PUT `/api/contacts/{id}`

| Field | Type | Required | Rules |
| --- | --- | --- | --- |
| `fullName` | string | yes | 1–120 characters |
| `email` | string | no | Valid email, ≤ 254 characters |
| `phone` | string | no | ≤ 30 characters |
| `organization` | string | no | ≤ 120 characters |
| `notes` | string | no | ≤ 2000 characters |

**201** / **200** Contact detail.

Errors: `400`, `404` (PUT only), `422 contact.archived` (PUT on an archived contact).

### POST `/api/contacts/{id}/archive` · POST `/api/contacts/{id}/restore`
**200** Contact detail. Idempotent.

Errors: `404`, `409 contact.in_active_rental` (archive only, BR-21).

A contact's rental history is listed with `GET /api/rentals?contactId={id}`.

---

## 7. Rentals (`/api/rentals`)

### 7.1 Shapes

**Rental summary** (lists):

```json
{ "id": "…", "contactId": "…", "contactName": "Ade Bello", "status": "checkedOut",
  "startDate": "2026-12-01", "dueDate": "2026-12-05", "itemCount": 3,
  "total": 310.00, "currencyCode": "CAD", "isOverdue": false, "isPickupLate": false }
```

**Rental detail:**

```json
{
  "id": "…",
  "status": "checkedOut",
  "contact": { "id": "…", "fullName": "Ade Bello" },
  "startDate": "2026-12-01",
  "dueDate": "2026-12-05",
  "rentalDays": 5,
  "checkedOutOn": "2026-12-01",
  "completedOn": null,
  "cancelledAt": null,
  "currencyCode": "CAD",
  "items": [
    {
      "id": "…", "equipmentId": "…", "equipmentName": "XLR Cable 25ft", "trackingType": "bulk",
      "quantity": 20, "dayRate": 1.00, "lineFee": 100.00,
      "conditionOut": "good", "conditionOutNotes": null,
      "returnedOn": null, "quantityReturned": null, "quantityMissing": null,
      "conditionIn": null, "conditionInNotes": null,
      "daysLate": 0, "isOverdue": false
    }
  ],
  "subtotal": 310.00,
  "feeAdjustment": 0.00,
  "feeAdjustmentNote": null,
  "total": 310.00,
  "depositAmount": 100.00,
  "depositRefunded": null,
  "notes": null,
  "isOverdue": false,
  "isPickupLate": false,
  "createdAt": "…",
  "updatedAt": "…"
}
```

Calculated fields (`rentalDays`, `lineFee`, `subtotal`, `total`, `daysLate`, `quantityMissing`, `isOverdue`, `isPickupLate`) follow doc 03 §5 exactly.

Every rental operation below returns **200 with the full rental detail**, so the client always gets the new state back.

### 7.2 Reading

**GET `/api/rentals`**
Query: `status` (repeatable, e.g. `status=reserved&status=checkedOut`), `contactId`, `from` and `to` (rentals whose `startDate…dueDate` overlaps the window), `page`, `pageSize`, `sort` (`startDate`, `-startDate` default, `dueDate`, `-dueDate`).

**200** Paged rental summaries.

**GET `/api/rentals/active`**
Everything currently out, plus reservations whose pickup is late. That means every `checkedOut` rental, and every `reserved` rental with `startDate < today`. Sorted by `dueDate`. Not paged.

**200** An array of rental summaries, with `isOverdue` and `isPickupLate` set.

**GET `/api/rentals/overdue`**
Every overdue rental (BR-13), with conflict warnings (BR-14). Sorted by oldest due date first. Not paged.

**200:**

```json
[
  {
    "rental": { "…rental summary…": "" },
    "overdueItems": [
      { "rentalItemId": "…", "equipmentId": "…", "equipmentName": "QSC K12.2 Speaker",
        "quantityOutstanding": 1, "dueDate": "2026-12-05", "daysLate": 2 }
    ],
    "conflicts": [
      { "rentalId": "…", "contactName": "Tolu Ade", "startDate": "2026-12-08",
        "equipmentId": "…", "equipmentName": "QSC K12.2 Speaker" }
    ]
  }
]
```

**GET `/api/rentals/{id}`**
**200** Rental detail.

Errors: `404`.

### 7.3 Creating and editing

**POST `/api/rentals`** creates a reservation (status `reserved`).

```json
{
  "contactId": "…",
  "startDate": "2026-12-06",
  "dueDate": "2026-12-08",
  "items": [ { "equipmentId": "…", "quantity": 1 }, { "equipmentId": "…", "quantity": 12 } ],
  "depositAmount": 100.00,
  "notes": "Wedding at Heritage Park"
}
```

| Field | Required | Rules |
| --- | --- | --- |
| `contactId` | yes | An active contact of this owner (BR-08) |
| `startDate` | yes | ≥ today (BR-06) |
| `dueDate` | yes | ≥ `startDate` (BR-01) |
| `items` | yes | 1–100 lines. The same `equipmentId` can't appear twice (BR-07). |
| `items[].quantity` | yes | ≥ 1. Must be `1` for Individual items (BR-07). |
| `depositAmount` | no | Default `0`. Range 0–99,999,999.99. |
| `notes` | no | ≤ 2000 characters |

Day rates are copied from each piece of equipment at this moment (BR-15). The currency is copied from the profile.

**201** Rental detail, with `Location: /api/rentals/{id}`.

Errors: `400 validation.failed`, `422 rental.contact_not_found`, `422 rental.contact_archived`, `422 rental.equipment_not_found`, `422 rental.equipment_archived`, `422 rental.start_in_past` (BR-06), `422 rental.invalid_dates` (BR-01), `422 rental.duplicate_item` (BR-07), `422 rental.invalid_quantity` (BR-07), `409 rental.unavailable` (BR-02 or BR-03, with the `unavailable` array shown in §1.4), `409 concurrency.conflict`.

**PUT `/api/rentals/{id}`** edits a reservation (BR-25). The body is the same shape as create, and this is a full replace of `contactId`, `startDate`, `dueDate`, `items`, `depositAmount` and `notes`.

- Only allowed while the rental is `reserved`.
- Lines are matched by `equipmentId`. Existing lines keep their stored `dayRate`, new lines copy the current rate, and lines that are left out are removed.
- Availability is re-checked excluding this rental's own current lines.
- The edit is all-or-nothing.

**200** Rental detail.

Errors: as for create, plus `404` and `422 rental.not_editable` (not `reserved`).

**POST `/api/rentals/{id}/change-due-date`** changes the due date while the gear is out (BR-26).
Body: `{ "dueDate": "2026-12-09" }`

- Only allowed while the rental is `checkedOut`.
- The new date must be ≥ `checkedOutOn`. An extension is checked for availability on the items that are still out.

**200** Rental detail, repriced.

Errors: `400`, `404`, `422 rental.not_checked_out`, `422 rental.invalid_dates`, `409 rental.unavailable`, `409 concurrency.conflict`.

**PUT `/api/rentals/{id}/notes`**
Body: `{ "notes": "…" }` (≤ 2000 characters, or `null` to clear). Allowed in any status except `cancelled`.

**200** Rental detail.

Errors: `400`, `404`, `422 rental.cancelled`.

**PUT `/api/rentals/{id}/settlement`** records the fee adjustment and deposit refund (BR-17, BR-18).
Body: `{ "feeAdjustment": -20.00, "feeAdjustmentNote": "Loyal client discount", "depositRefunded": 100.00 }`

| Field | Rules |
| --- | --- |
| `feeAdjustment` | Required. −99,999,999.99 to 99,999,999.99. Allowed in any status except `cancelled`. |
| `feeAdjustmentNote` | Required when `feeAdjustment ≠ 0`. ≤ 500 characters. |
| `depositRefunded` | Optional, or `null`. 0 to `depositAmount`. **Only allowed once the rental is `returned`.** |

**200** Rental detail.

Errors: `400`, `404`, `422 rental.cancelled`, `422 rental.note_required`, `422 rental.refund_not_allowed` (not returned yet), `422 rental.invalid_refund` (more than the deposit).

### 7.4 Lifecycle

**POST `/api/rentals/{id}/checkout`** (BR-09)

```json
{
  "checkedOutOn": "2026-12-06",
  "items": [ { "rentalItemId": "…", "conditionOut": "excellent", "conditionOutNotes": null } ]
}
```

- `checkedOutOn` is optional and defaults to today. It must be between `startDate` and today.
- `items` must list **every** line on the rental exactly once.

**200** Rental detail (`checkedOut`).

Errors: `400`, `404`, `422 rental.not_reserved`, `422 rental.checkout_too_early` (today < `startDate`), `422 rental.invalid_checkout_date`, `422 rental.checkout_incomplete` (lines missing, unknown or duplicated), `409 concurrency.conflict`.

**POST `/api/rentals/{id}/returns`** (BR-10, BR-11, BR-12)

```json
{
  "returnedOn": "2026-12-08",
  "items": [
    { "rentalItemId": "…", "quantityReturned": 1,  "conditionIn": "good", "conditionInNotes": null },
    { "rentalItemId": "…", "quantityReturned": 10, "conditionIn": "fair", "conditionInNotes": "2 cables missing" }
  ]
}
```

- `returnedOn` is optional and defaults to today. It must be between `checkedOutOn` and today.
- `items` lists **one or more** lines that haven't been returned yet. A partial return is fine.
- `quantityReturned` is between 0 and the line's quantity. It defaults to the full quantity if omitted. For an Individual item, `0` means it was lost.
- When an Individual line is returned, the equipment's current condition is set to `conditionIn` (doc 03 §5.4).
- When the last line is returned, the rental becomes `returned` and `completedOn` is set to `returnedOn` (BR-12).

**200** Rental detail.

Errors: `400`, `404`, `422 rental.not_checked_out`, `422 rental.item_already_returned` (BR-10), `422 rental.invalid_return_date`, `422 rental.invalid_quantity_returned`, `422 rental.unknown_item`, `409 concurrency.conflict`.

**POST `/api/rentals/{id}/cancel`**
No body. Only allowed while the rental is `reserved`.

**200** Rental detail (`cancelled`).

Errors: `404`, `422 rental.not_cancellable`.

---

## 8. Operational

| Endpoint | Auth | Response |
| --- | --- | --- |
| `GET /health` | Public | `200 Healthy` or `503 Unhealthy` (includes a database check). Returns no details. |
| `GET /openapi/v1.json` | Public, **Development only** | The OpenAPI document |
| `GET /swagger` | Public, **Development only** | Swagger UI, with an "Authorize" button for the bearer token |

---

## 9. Error code catalogue

| Code | Status | Rule | Meaning |
| --- | --- | --- | --- |
| `validation.failed` | 400 | | Input shape is invalid; see `errors` |
| `auth.weak_password` | 400 | | Password doesn't meet the policy (doc 05) |
| `auth.invalid_credentials` | 401 | | Login failed (always generic) |
| `auth.invalid_refresh_token` | 401 | | Refresh token unknown, expired, revoked or reused |
| `auth.invalid_invite` | 403 | | Invite code doesn't match |
| `auth.email_taken` | 409 | | An account already exists for this email |
| `auth.wrong_password` | 422 | | Current password incorrect (password change) |
| `profile.invalid_time_zone` | 422 | | Not a recognised IANA time zone |
| `not_found` | 404 | BR-24 | Route resource doesn't exist for this owner |
| `concurrency.conflict` | 409 | | Lost a race with a simultaneous change; retry |
| `category.name_taken` | 409 | BR-23 | |
| `equipment.category_not_found` | 422 | BR-24 | |
| `equipment.category_archived` | 422 | | |
| `equipment.serial_not_allowed` | 422 | | Serial number on a Bulk item |
| `equipment.invalid_quantity` | 422 | | Quantity doesn't match the tracking type |
| `equipment.serial_taken` | 409 | BR-23 | |
| `equipment.archived` | 422 | BR-20 | Edit attempted on archived equipment |
| `equipment.quantity_below_committed` | 409 | BR-22 | Includes `minimumQuantity` |
| `equipment.in_active_rental` | 409 | BR-21 | Includes `rentalIds` |
| `contact.archived` | 422 | BR-20 | |
| `contact.in_active_rental` | 409 | BR-21 | Includes `rentalIds` |
| `rental.contact_not_found` | 422 | BR-24 | |
| `rental.contact_archived` | 422 | BR-08 | |
| `rental.equipment_not_found` | 422 | BR-24 | Includes `equipmentIds` |
| `rental.equipment_archived` | 422 | BR-08 | Includes `equipmentIds` |
| `rental.start_in_past` | 422 | BR-06 | |
| `rental.invalid_dates` | 422 | BR-01, BR-26 | |
| `rental.duplicate_item` | 422 | BR-07 | |
| `rental.invalid_quantity` | 422 | BR-07 | |
| `rental.unavailable` | 409 | BR-02, BR-03 | Includes `unavailable[]` |
| `rental.not_editable` | 422 | BR-25 | Not reserved |
| `rental.not_reserved` | 422 | BR-09 | |
| `rental.checkout_too_early` | 422 | BR-09 | |
| `rental.invalid_checkout_date` | 422 | BR-09 | |
| `rental.checkout_incomplete` | 422 | BR-09 | |
| `rental.not_checked_out` | 422 | BR-10, BR-26 | |
| `rental.item_already_returned` | 422 | BR-10 | |
| `rental.invalid_return_date` | 422 | BR-10 | |
| `rental.invalid_quantity_returned` | 422 | BR-11 | |
| `rental.unknown_item` | 422 | | `rentalItemId` isn't on this rental |
| `rental.not_cancellable` | 422 | | Not reserved |
| `rental.cancelled` | 422 | | Change attempted on a cancelled rental |
| `rental.note_required` | 422 | BR-17 | |
| `rental.refund_not_allowed` | 422 | BR-18 | Not returned yet |
| `rental.invalid_refund` | 422 | BR-18 | More than the deposit |
| `rate_limited` | 429 | | |
| `server.error` | 500 | | Unexpected; see `traceId` |

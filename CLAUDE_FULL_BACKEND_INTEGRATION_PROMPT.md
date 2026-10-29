# Claude Implementation Brief — Webtizo QR Ordering Dashboard Backend

## Mission

You are the senior full-stack engineer responsible for connecting the existing Webtizo QR Ordering Dashboard frontend to a secure, maintainable, production-ready backend.

**Frontend source of truth:** `webtizo-dashboard-production.html` (the current corrected dashboard supplied with this prompt).

Read the entire HTML file before coding. Do not rebuild the UI from scratch. Preserve its current design, layout, colors, typography, animation, navigation, responsive behavior, labels, and user workflows. The existing frontend is a single-page HTML/CSS/JavaScript application. Its UI uses vanilla JavaScript, Tailwind CDN utilities, and Chart.js. Do not change the visual design except for small loading, error, empty, and success states needed for reliable data loading.

Implement the actual backend, database, authentication, validation, authorization, and API integration. Do not fake successful API calls, return hard-coded business records, reintroduce demo data, or leave CRUD operations backed only by in-memory arrays/localStorage.

## Required deliverables

Create a complete, runnable project with:
1. Backend source code and a clear folder structure.
2. Database schema/migrations and seed process that creates **no fake business records**. A development-only admin account may be created only through a documented secure setup command and must require a password supplied through environment variables or an interactive prompt.
3. API routes/controllers/services/repositories, validation, authentication and authorization.
4. Updated `webtizo-dashboard-production.html` (or a cleanly separated frontend JS file if that is more maintainable) wired to the API while preserving the existing UI.
5. `.env.example` containing placeholders only. Never commit real credentials.
6. API documentation with methods, paths, request/response examples, status codes, and permissions.
7. Setup instructions for local development and deployment on Hostinger (if the chosen stack is compatible), including database setup, environment configuration, migrations, document/file storage, HTTPS, and production build steps.
8. Automated tests for authentication, permissions, validation, CRUD, onboarding details, and key dashboard calculations.
9. A final integration report listing files created/changed, routes implemented, how to run it, and any configuration still required.

## Stack and deployment assumptions

First inspect the existing repository/project files. Reuse an existing stack if it is already established and working. If no backend exists, choose a straightforward stack compatible with the intended Hostinger deployment:
- PHP 8.2+ with PDO and MySQL/MariaDB is a practical default for standard Hostinger PHP hosting.
- Use a maintainable separation of routes/controllers/services/repositories/validation.
- Use prepared statements for all SQL.
- Use database migrations or a versioned SQL schema.
- Do not use a browser-side database connection.
- Do not put database credentials or secrets in frontend code.
- If you choose Node.js instead, first verify the actual target hosting supports the required runtime and deployment model; explain any hosting limitations. Do not silently assume.
- Keep the frontend and API same-origin where possible, for example `/api/v1/...`, to avoid unnecessary CORS complexity.

Do not begin by asking the user to design the schema. Derive the schema from the fields and interactions in the supplied frontend, document any unavoidable assumptions, and implement a coherent first version.

---

# 1. Frontend architecture and identifiers to inspect

The current HTML uses an IIFE with strict mode and a central state object. Inspect the exact current source before editing; line numbers can change.

## Global DOM/helpers and state

- `$` and `$$`: DOM query helpers.
- `esc(...)`: HTML escaping helper. Continue escaping all server-provided values before inserting them into HTML.
- `uid()`: current client-side temporary ID helper. Server-created records must use server-generated IDs; do not trust client-generated IDs as authoritative.
- `S`: central in-memory UI/data state.
- `S.ui`: transient UI state such as active view, filters, selected client, calendar month/day, chart mode, and search query. This state should remain frontend-only and must not be persisted to the database.
- `load()`: initializes state and reads initial route/hash. Remove all business-data seeding and make it fetch real data from the API after authentication.
- `save()`: currently intentionally disabled/no-op in the corrected frontend. Replace the old assumption that save means browser persistence with explicit asynchronous API calls for each create/update/delete action. Do not turn `save()` into localStorage persistence.
- `render(animate=false, opts={})`: re-render current view and rebuild chart when needed. Avoid calling it on every keystroke if that destroys focus/caret; preserve input state and debounce server searches.
- `go(view)`: navigation function. Keep view names and hash routing stable.
- `VIEWS`: maps view names to render functions.
- `A`: delegated event-action registry for `data-act` controls.
- `modal(...)`: shared modal helper. Form submissions currently call local callbacks; wire those callbacks to API operations, show validation/server errors, disable submit during requests, and refresh the relevant view after success.
- `toast(...)`: existing notification helper; use it for success and non-blocking messages. Use visible inline errors for API/load failures.
- `buildCharts(...)`: Chart.js charts. Chart values must come from API-backed records/summary endpoints, not fabricated data.
- `exportCSV()`: currently exports the local finance array. Fetch authorized finance records from the backend (or use a backend export endpoint) and preserve CSV escaping. Protect against spreadsheet formula injection for values starting with `=`, `+`, `-`, or `@`.

## Navigation view identifiers

The frontend uses these exact view keys and hashes:
- `overview`
- `pipeline`
- `clients`
- `onboarding`
- `finance`
- `tasks`
- `schedule`

Keep these identifiers stable; they are used in `data-view`, `go(...)`, `VIEWS`, and `location.hash`.

## Delegated `data-act` identifiers

Preserve these existing actions and connect them to real API operations as appropriate:

- `nav`
- `closeModal`
- `openSearch`
- `openBell`
- `openTargets`
- `addClient`
- `addDeal`
- `addTx`
- `addTask`
- `addEvent`
- `export`
- `goto`
- `openClient`
- `selClient`
- `planFilter`
- `toggleStar`
- `delClient`
- `setStatus`
- `contacted`
- `delNote`
- `cur`
- `chartMode`
- `dealMove`
- `dealWon`
- `dealLost`
- `dealReopen`
- `toggleClosed`
- `taskToggle`
- `taskMove`
- `delTask`
- `delEvent`
- `monthNav`
- `txType`
- `delTx`
- `receive`
- `calNav`
- `calToday`
- `calDay`
- `onbDetails`
- `onbFilter`

Inspect the current `A` object and all markup before implementing. If an action exists in markup but not in the registry, add it deliberately. Never let an action silently mutate only the frontend when it represents a persistent record.

## Search and input identifiers

Current UI uses:
- `data-bind="clientQ"` — client search
- `data-bind="dealQ"` — pipeline search
- `data-bind="txQ"` — finance search
- `data-bind="onbQ"` — restaurant application search
- `id="onbSearch"` — onboarding search input
- `data-act="onbFilter"` — onboarding status filter
- `id="pq"` and `id="pr"` — global search input/results
- `data-field` with `data-cid` — inline client fields
- `data-form="quickTask"` — quick task form
- `data-form="note"` with `data-cid` — add client note form
- modal form controls named after the field keys passed to `fld(...)`.

Keep IDs and data attributes where practical because delegated handlers depend on them. Fix any broken/missing action binding discovered during the audit.

---

# 2. Current frontend data models and required backend entities

Use normalized database tables with server-generated primary keys. Use UUIDs or opaque public IDs where suitable. Do not use display names as foreign keys.

## A. Users and sessions

Create `users` (or equivalent):
- `id`
- `name`
- `email` (unique, normalized)
- `password_hash`
- `role` (at minimum `admin`; optionally `staff` with restricted permissions)
- `is_active`
- `last_login_at`
- `created_at`, `updated_at`

Use secure server-side sessions for PHP or a similarly secure auth mechanism. Prefer HttpOnly, Secure-in-production, SameSite cookies. Implement CSRF protection for cookie-authenticated state-changing requests. Rate-limit login attempts. Use password hashing (`password_hash`/`password_verify` for PHP). Never store plaintext passwords or tokens in localStorage.

Provide:
- `POST /api/v1/auth/login`
- `POST /api/v1/auth/logout`
- `GET /api/v1/auth/me`
- Optional `POST /api/v1/auth/change-password`

Unauthenticated dashboard requests must not return private records. All API routes except login and health check require authentication.

## B. Restaurant clients

The frontend client object currently references:
`id`, `name`, `biz`, `phone`, `email`, `city`, `plan`, `tables`, `status`, `fee`, `star`, `last`, and `notes`.

Use a normalized `clients` table:
- `id`
- `owner_name` (maps to frontend `name`)
- `business_name` (maps to frontend `biz`)
- `phone`
- `email`
- `city`
- `plan` (validate allowed plan values; frontend currently offers Basic, Standard, Premium)
- `table_count` (maps to `tables`)
- `status` (frontend values: `pending`, `setup`, `live`)
- `monthly_fee` in INR paise or a clearly documented decimal currency column; never use floating-point arithmetic for money
- `is_favorite` (maps to `star`)
- `last_contacted_at` (maps to `last`)
- `created_at`, `updated_at`, optional `deleted_at` if using soft deletion

Return a frontend adapter/DTO using the existing property names if changing every UI template would create unnecessary risk. Document the mapping explicitly:
- `owner_name` ↔ `name`
- `business_name` ↔ `biz`
- `table_count` ↔ `tables`
- `monthly_fee` ↔ `fee`
- `is_favorite` ↔ `star`
- `last_contacted_at` ↔ `last`

Do not use cascading deletes to silently erase financial history. Deleting a client should either be blocked when linked records exist or use a documented soft-delete/archive policy.

Client routes:
- `GET /api/v1/clients` (search/filter/pagination)
- `POST /api/v1/clients`
- `GET /api/v1/clients/{id}`
- `PATCH /api/v1/clients/{id}`
- `DELETE /api/v1/clients/{id}` (soft delete/archive recommended)
- `POST /api/v1/clients/{id}/contacted`
- `POST /api/v1/clients/{id}/favorite` or `PATCH` favorite field
- `PATCH /api/v1/clients/{id}/status`
- `GET /api/v1/clients/{id}/notes`
- `POST /api/v1/clients/{id}/notes`
- `DELETE /api/v1/clients/{id}/notes/{noteId}`
- `GET /api/v1/clients/{id}/history`

Notes table:
- `id`, `client_id`, `text`, `tone` if still needed by UI, `created_by`, `created_at`, `updated_at`.
- Map `text` and `date` to the frontend note shape.
- The history view currently mixes notes and financial entries; build it from related notes and client-linked transactions, ordered newest first.

## C. Sales pipeline deals

Frontend deal fields:
- `id`
- `title`
- `client` (currently a display name/free text, not a reliable foreign key)
- `value`
- `stage`
- `closes`
- `closed` (optional close date)

Stages used by UI:
- open stages: `lead`, `demo`, `nego`
- closed stages: `won`, `lost`

Labels are held in `STAGES`:
- `lead` → Qualification
- `demo` → Demo and proposal
- `nego` → Negotiation
- `won` and `lost` are closed stages.

Use `deals`:
- `id`, `title`, `client_id` nullable, `lead_business_name` nullable for prospects not yet clients, `value_minor_units`, `stage`, `expected_close_date`, `closed_at`, `created_at`, `updated_at`.

Do not store only a business name if it can be linked to an existing client. For leads without a client, store the lead business name. Avoid double-counting revenue: moving a deal to `won` should not silently create a finance income transaction unless the product requirement explicitly defines that behavior. If creating a transaction on win, use a database transaction and prevent duplicates.

Routes:
- `GET /api/v1/deals` (stage/search/closed filters)
- `POST /api/v1/deals`
- `GET /api/v1/deals/{id}`
- `PATCH /api/v1/deals/{id}` (title, value, stage, expected close)
- `DELETE /api/v1/deals/{id}`
- `POST /api/v1/deals/{id}/stage` or `PATCH` stage

Moving a deal via arrows or drag-and-drop must persist through the API and only update the UI after server success (or rollback optimistic updates on failure).

## D. Finance transactions

Frontend transaction fields:
- `id`
- `type`: `income`, `expense`, `invest`, `pending`
- `title`
- `amount`
- `date`
- `category`
- `clientId`

Frontend categories in `CATS`:
- `income`: `Monthly plan`, `QR menu setup`, `Custom work`
- `pending`: `Monthly plan`, `QR menu setup`, `Custom work`
- `expense`: `Hosting and domain`, `Marketing`, `Printing and stands`, `Freelancer`, `Tools and software`, `Other`
- `invest`: `Ads budget`, `Equipment`, `Learning`, `Other`

Use `transactions`:
- `id`
- `type` enum
- `title`
- `amount_minor_units` (INR paise) or an equivalent exact-decimal representation
- `currency` (initially INR)
- `transaction_date`
- `category`
- `client_id` nullable
- `status` if needed; do not overload `type` to represent paid/unpaid if it breaks existing behavior
- `created_by`
- `created_at`, `updated_at`

Routes:
- `GET /api/v1/transactions` (month, type, search, client, pagination filters)
- `POST /api/v1/transactions`
- `GET /api/v1/transactions/{id}`
- `PATCH /api/v1/transactions/{id}`
- `DELETE /api/v1/transactions/{id}`
- `POST /api/v1/transactions/{id}/mark-received` for pending collection
- `GET /api/v1/finance/summary?month=YYYY-MM`
- `GET /api/v1/finance/timeseries?months=7`
- `GET /api/v1/finance/export.csv` (optional)

The existing `receive` action changes a transaction from `pending` to `income` and updates the date to today. Implement this atomically and auditably. Consider recording `received_at`; preserve the original due date if useful.

All finance totals must be calculated server-side from authorized records. Use exact arithmetic for money. Currency conversions currently use sample static rates in `CUR`; do not describe those as live exchange rates. Keep the UI default in INR. Either remove/disable conversions until a real rate provider is configured or clearly label conversion rates as configured reference rates. Do not fabricate live rates.

## E. Tasks

Frontend task fields:
- `id`
- `title`
- `status`: `todo`, `doing`, `done`
- `prio`: `high`, `med`, `low`
- `due`
- `client` (currently client ID in task data, although rendered as client label)

Use `tasks`:
- `id`, `title`, `status`, `priority`, `due_date`, `client_id` nullable, `created_by`, `created_at`, `updated_at`, optional `completed_at`.

Routes:
- `GET /api/v1/tasks` (status, due date, client, search filters)
- `POST /api/v1/tasks`
- `GET /api/v1/tasks/{id}`
- `PATCH /api/v1/tasks/{id}` (title, status, priority, due date, client)
- `DELETE /api/v1/tasks/{id}`

Drag-and-drop, status arrows, checkbox completion, and delete must persist. Dashboard overdue/today counts must use the correct local date/time zone and not include completed tasks as overdue.

## F. Schedule events

Frontend event fields:
- `id`
- `title`
- `date`
- `start`
- `end`
- `client` (client ID or empty string)

Use `events`:
- `id`, `title`, `event_date`, `start_time`, `end_time`, `client_id` nullable, `created_by`, `created_at`, `updated_at`.

Routes:
- `GET /api/v1/events?from=YYYY-MM-DD&to=YYYY-MM-DD`
- `POST /api/v1/events`
- `GET /api/v1/events/{id}`
- `PATCH /api/v1/events/{id}`
- `DELETE /api/v1/events/{id}`

Validate end time is after start time, validate dates, and return records in a predictable order. Schedule calendar events and task reminders should come from the backend, not seed data.

## G. Settings / dashboard targets

Frontend settings currently include:
- `S.settings.name`
- `S.settings.goal` (monthly earning goal)
- `S.settings.spendLimit` (monthly spending limit)

Use a per-user or workspace `settings` table:
- `user_id` or `workspace_id`
- `display_name`
- `monthly_income_goal_minor_units`
- `monthly_spending_limit_minor_units`
- `currency` (INR default)
- `created_at`, `updated_at`

Routes:
- `GET /api/v1/settings`
- `PATCH /api/v1/settings`

Do not use a frontend-supplied user ID to choose whose settings to update; use the authenticated session identity.

## H. Restaurant QR onboarding applications — priority feature

This section is a core part of the dashboard. It lists submitted restaurant applications and opens the full application record in a View Details modal. The application data likely originates from the separate public QR Ordering System application/checkout form. Inspect the actual form/repository if available and reuse its existing field names and payment flow. Do not invent field values.

The frontend expects `ONB_APPLICATIONS`, and `vOnboarding()` and `showOnboardingDetails(id)` consume these fields. Replace the in-memory constant with API-loaded application records. Use the same `id` in:
- table row: `data-act="onbDetails" data-id="${r.id}"`
- action registry: `onbDetails: el => showOnboardingDetails(el.dataset.id)`
- detail lookup: application record ID matches the `id` passed to `showOnboardingDetails(id)`.

Never use `reference_id` as the primary key unless the existing submission system already guarantees this. Use internal `id` for database relations and `reference_id` as a public reference code.

### Fields directly referenced by the onboarding list/details UI

Required base properties:
- `id`
- `reference_id`
- `created_at`
- `status`
- `payment_status`
- `restaurant_name`
- `restaurant_type`
- `owner_name`
- `email`
- `phone`
- `whatsapp`
- `address`
- `city`
- `state`
- `pincode`
- `gst_number`
- `fssai`
- `table_count`
- `staff_count`
- `open_time`
- `close_time`
- `current_ordering`
- `order_types`
- `gst_percent`
- `payment_mode`
- `special_notes`
- `plan`
- `billing_cycle`
- `amount`
- `logo_url`
- `menu_file_urls`
- `primary_color`
- `secondary_color`
- `menu_style`
- `menu_layout`
- `instagram`
- `maps_link`
- `design_notes`

The detail modal displays sections in this order:
1. Selected plan / billing cycle / application amount.
2. Application status: `reference_id`, `created_at`, `status`, `payment_status`.
3. Restaurant information: `restaurant_name`, `restaurant_type`, `owner_name`, `email`, `phone`, `whatsapp`, `address`, `city`, `state`, `pincode`, `gst_number`, `fssai`.
4. Tables & operations: `table_count`, `staff_count`, `open_time`, `close_time`, `current_ordering`, `order_types`, `gst_percent`, `payment_mode`, `special_notes`.
5. Brand & menu: `logo_url`, `menu_file_urls`, `primary_color`, `secondary_color`, `menu_style`, `menu_layout`, `instagram`, `maps_link`, `design_notes`.

`value(...)` in the UI renders missing fields as “Not provided”; preserve that behavior for optional fields. Escape every string. For arrays such as `order_types` or `menu_file_urls`, return JSON arrays and ensure the frontend renders them safely (or normalize to a documented display string). Do not return raw filesystem paths or private storage credentials.

### Application status values used by UI

- `pending`
- `processing`
- `approved`
- `rejected`

### Payment status values used by UI

- `pending`
- `paid`
- `failed`

The current list maps plans for display:
- `starter` → Starter
- `smart_ordering` → Smart Ordering
- `enterprise` → Enterprise

Keep stored plan keys stable if they are already used by the public order form. If existing plan IDs differ, implement a documented mapping adapter instead of silently changing plan IDs.

### Recommended onboarding tables

`restaurant_applications`:
- `id` (internal primary key)
- `reference_id` (unique public reference)
- `restaurant_name`
- `restaurant_type`
- `owner_name`
- `email`
- `phone`
- `whatsapp`
- `address`
- `city`
- `state`
- `pincode`
- `gst_number` nullable
- `fssai` nullable
- `table_count`
- `staff_count` nullable
- `open_time` nullable
- `close_time` nullable
- `current_ordering` nullable
- `order_types` JSON nullable
- `gst_percent` decimal nullable
- `payment_mode` nullable
- `special_notes` text nullable
- `plan`
- `billing_cycle`
- `amount_minor_units` exact integer paise or decimal equivalent
- `status`
- `payment_status`
- `logo_url` nullable
- `menu_file_urls` JSON nullable
- `primary_color` nullable
- `secondary_color` nullable
- `menu_style` nullable
- `menu_layout` nullable
- `instagram` nullable
- `maps_link` nullable
- `design_notes` text nullable
- `created_at`, `updated_at`
- `reviewed_at` nullable
- `reviewed_by` nullable

If the existing application form has additional fields, preserve them and include them in the detail response. Avoid discarding any field submitted by the client. Store uploaded files in controlled storage and save safe URLs/keys in the database; validate MIME type, extension, size, and permissions.

`payment_records` (or integrate with the existing payment provider records):
- `id`
- `application_id`
- provider/payment order ID
- provider payment ID (nullable until paid)
- amount
- currency
- status
- verified_at
- created_at, updated_at

**Security-critical payment rule:** Never trust a browser-submitted `payment_status=paid`. Confirm payment through a verified provider webhook or server-side provider API. Verify webhook signatures and make webhook processing idempotent. Do not mark an application paid merely because a redirect page says success.

### Onboarding endpoints

- `GET /api/v1/onboarding/applications` — paginated list; supports `q`, `status`, `payment_status`, `plan`, `page`, `per_page`, and safe sorting.
- `GET /api/v1/onboarding/applications/{id}` — return the complete application detail object using the internal ID.
- `PATCH /api/v1/onboarding/applications/{id}` — authorized admin update for application status and permitted operational fields.
- Optional `POST /api/v1/onboarding/applications/{id}/approve`
- Optional `POST /api/v1/onboarding/applications/{id}/reject`
- Optional payment endpoints/webhooks based on the payment provider already used by the public application flow.

For the table, return the fields needed for the list. For View Details, fetch the complete record from the server when clicked (or ensure list payload is intentionally complete and documented). Prefer fetching details by ID to avoid stale or incomplete list data. Show loading/error states in the modal. If not found, show a visible message rather than silently doing nothing.

### Onboarding summary values

The current view computes:
- total applications = count of all applications
- pending review = status `pending`
- in progress = status `processing`
- approved = status `approved`
- payments received = payment status `paid`

Use SQL aggregate queries or an efficient summary endpoint. The summary and list must use the same authorized dataset and filters should not accidentally change the global summary unless explicitly intended.

---

# 3. Required API contract

Use a consistent JSON envelope and HTTP status codes. Example:

Success:
```json
{
  "data": { "id": "server-id" },
  "meta": {}
}
```

List:
```json
{
  "data": [],
  "meta": {
    "page": 1,
    "per_page": 20,
    "total": 0
  }
}
```

Error:
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Please check the submitted fields.",
    "fields": {
      "email": "Enter a valid email address."
    },
    "request_id": "..."
  }
}
```

Use appropriate HTTP codes:
- `200` read/update success
- `201` created
- `204` successful delete if no body is returned
- `400` malformed request
- `401` unauthenticated
- `403` authenticated but not authorized
- `404` missing record
- `409` conflict/duplicate
- `422` field validation failure
- `429` rate limit
- `500` unexpected server error (do not expose stack traces or SQL details)

Use `/api/v1` prefix. Use UTC timestamps in storage and ISO 8601 strings in API responses; document the dashboard display timezone (likely `Asia/Kolkata`). Dates like task due dates and event dates should be handled consistently as date-only values where appropriate.

---

# 4. Dashboard data loading and calculation requirements

Create a dashboard summary endpoint, for example:
- `GET /api/v1/dashboard/summary?month=YYYY-MM`

It should provide enough data to populate Overview KPIs, finance chart, goal progress, open deals, client list preview, open tasks, and upcoming events. Alternatively use multiple well-defined endpoints if it improves maintainability.

Do not invent records when tables are empty. Return zero totals and empty arrays. Display a polished empty state instead of blank content or fake figures.

Expected calculations:
- Client counts and status counts from real clients.
- Revenue, expenses, investments, and pending collections from real transactions.
- Net profit calculation must be explicitly defined and consistent. Current frontend often uses earnings minus spend; determine whether investment is included, document the rule, and apply it consistently.
- Monthly goal progress from `settings.monthly_income_goal`.
- Spending limit progress from `settings.monthly_spending_limit`.
- Pipeline totals from open deals; won/lost deals separately.
- Alerts from real events today, incomplete overdue/due-today tasks, and pending collections.
- Charts from actual monthly aggregate data.
- Use the same aggregation rules on Overview and Finance so totals agree.

All data fetching must handle loading, empty, and error states. A network failure must not be rendered as “no records” without explaining the failure.

---

# 5. Frontend API integration rules

Implement one centralized API client:
- Base URL configurable through one constant or environment/build config, e.g. `API_BASE_URL = '/api/v1'`.
- Use `fetch` with `credentials: 'same-origin'` for cookie sessions.
- Set `Accept: application/json`; set `Content-Type: application/json` for JSON request bodies.
- Include CSRF token for state-changing requests if using cookie sessions.
- Parse API errors centrally; handle `401` by showing login/session-expired state.
- Use `AbortController` for obsolete searches/page changes.
- Debounce search input.
- Prevent duplicate submissions and show pending state.
- Never render a successful toast before the server confirms success.
- On failure, show a clear error and preserve the user's entered form data.
- Update/reload only the affected data after mutations.
- Use optimistic updates only if there is a rollback path.
- Do not persist private business data to localStorage/sessionStorage.
- UI preferences such as selected tab may remain in the URL hash; transient filters may remain in memory.
- Keep all IDs in API calls encoded safely with `encodeURIComponent`.
- Keep `esc(...)` or a safe DOM text insertion method for server data.
- Avoid direct `innerHTML` with untrusted strings. If templates continue using `innerHTML`, escape all server-provided values and test XSS payloads.
- Add a shared `setLoading`, `setError`, and empty-state mechanism where appropriate.

## Specific current frontend mutations to connect

Wire every one of these to an API request:
- Add client → `POST /clients`
- Edit client fields via `data-field` → `PATCH /clients/{id}`
- Favorite toggle → `PATCH /clients/{id}` or favorite endpoint
- Change QR status → `PATCH /clients/{id}/status`
- Mark contacted → `POST /clients/{id}/contacted`
- Delete client → `DELETE /clients/{id}` with confirmation
- Add/delete client note → `POST`/`DELETE /clients/{id}/notes`
- Add/edit/move/delete deal → deal routes
- Add/delete finance entry → transaction routes
- Mark pending entry received → mark-received route
- Add/move/complete/delete task → task routes
- Add/delete schedule event → event routes
- Edit targets/settings → `PATCH /settings`
- Onboarding list/filter/search → `GET /onboarding/applications`
- View onboarding details → `GET /onboarding/applications/{id}`
- Update onboarding status → authorized onboarding `PATCH`/approve/reject endpoint
- Overview metrics/charts → dashboard summary and finance aggregate endpoints
- CSV export → authorized backend export or API-loaded finance list

Any current local mutation function such as `moveDeal(...)`, `moveTask(...)`, or modal `onSubmit(...)` must call the backend and handle errors. Do not leave a second hidden local-only data path.

---

# 6. Database design, integrity, and business rules

- Use foreign keys for client relationships where appropriate.
- Add indexes for common filters: application `status`, `payment_status`, `created_at`, unique `reference_id`; client business name/email/phone; transaction date/type/client; task status/due date; event date; deal stage/expected close.
- Enforce unique email for users. Consider duplicate checks for application references and provider payment IDs.
- Use transactions for multi-step operations such as payment confirmation and conversion of pending finance entries to received income.
- Validate all inputs on the server, even if the browser validates them.
- Validate enum values, dates, time ranges, positive/nonnegative money, lengths, phone/email formats, URLs, and JSON arrays.
- Use parameterized SQL; never concatenate user input into SQL.
- Use a least-privilege database user.
- Add audit fields for created/updated records; audit high-impact status/payment changes.
- Never expose internal exception messages, SQL, password hashes, session IDs, secret paths, or provider credentials.
- Implement CSRF protection, secure cookies, login rate limiting, output escaping, security headers, and request size limits.
- Validate file uploads and prevent executable file uploads.
- Apply authentication and role checks to all administrative operations, especially onboarding applications and payment status.
- Provide pagination with a reasonable maximum page size; do not return unbounded records.
- Use database migrations and document backup/restore.
- Avoid destructive migrations; provide a backup and rollback plan.
- Include request IDs in logs and API errors, but never log passwords, session tokens, payment secrets, or unnecessary personal data.

---

# 7. Authentication and first-admin setup

Do not create a default login such as `admin/admin`, do not hard-code credentials, and do not publish secrets in the frontend.

Provide a CLI command or secure one-time setup flow to create the first admin. It must:
- Prompt securely for the email and password, or read them from environment variables that are not committed.
- Hash the password.
- Refuse to create another initial admin unless an authorized procedure is used.
- Be disabled or inaccessible after initial setup if a web setup route is used.
- Document how to change/rotate credentials.

All dashboard pages should require authentication. If the existing frontend has no login UI, add a minimal, visually consistent login/session-expired state without redesigning the dashboard. Protect API endpoints regardless of frontend visibility.

---

# 8. Testing and acceptance criteria

Do not declare complete until these checks pass:

## App startup
- Clean database migration succeeds.
- Application starts from documented commands.
- No demo business data appears after first login.
- No console exceptions or failed API calls on a fresh database.
- No JavaScript ReferenceError (including the previously observed `r is not defined` error).
- Favicon request does not produce a 404.
- No production dependency on Tailwind CDN if a compiled production CSS build is delivered; if CDN remains for local preview, document it and create a production build path.

## Authentication/security
- Unauthenticated API requests return `401`.
- Non-admin users cannot perform admin-only actions.
- Invalid CSRF requests are rejected for cookie-authenticated mutations.
- Invalid login attempts are rate-limited.
- XSS strings render as text.
- SQL injection inputs do not alter queries.
- Payment status cannot be changed to paid by an untrusted client request.
- Secret configuration is not committed or included in JS.

## CRUD
- Create/read/update/delete clients.
- Update client status, favorite, contacted date, notes.
- Create/read/update/move/close deals.
- Create/read/update/delete finance entries and mark pending entry received.
- Create/read/update/move/complete/delete tasks.
- Create/read/update/delete schedule events.
- Read and update settings.
- Search, filter, and paginate lists.
- Refreshing the browser preserves database-backed changes.

## Onboarding
- Empty database shows a proper empty state and zero summary cards.
- Submitting a real application through the existing public form creates a database record.
- It appears in the dashboard list after reload.
- Search by restaurant, owner, city, and reference works.
- Filter statuses `pending`, `processing`, `approved`, `rejected` works.
- The payment count includes only verified `paid` records.
- View Details uses the row's internal `id` and displays every listed field, including all optional fields and arrays.
- Missing optional fields display “Not provided.”
- Unknown ID shows a visible not-found state.
- Admin status updates persist and are authorized.
- Payment confirmation is verified server-side and webhook handling is idempotent.

## Responsive and UI
- Test 320px, 375px, 430px, 768px, 1024px, 1366px, and wide desktop widths.
- Keep the original theme and visual hierarchy.
- Tables can scroll horizontally on small screens without breaking the whole page.
- Modals fit the viewport and are keyboard accessible.
- Loading, empty, error, and success states are visually consistent.
- Chart.js handles empty datasets gracefully.
- Re-rendering does not lose the user's search input focus/caret.

## Documentation
- Document each API endpoint, fields, permissions, and errors.
- Document all frontend-to-database field mappings, especially the onboarding detail fields and client property aliases.
- Include `.env.example`, migration steps, setup, tests, deployment, backups, and rollback notes.
- State clearly which external integrations (payment provider, email, file storage) still require credentials or configuration.

---

# 9. Execution order

Follow this order and work through to completion without stopping after only writing a plan:

1. Inspect the full existing frontend and any existing public restaurant application form/backend/payment integration. Search the project for `ONB_APPLICATIONS`, `reference_id`, `restaurant_name`, `payment_status`, `save-form.php`, payment provider callbacks/webhooks, and any existing database connection/config files.
2. Report briefly what already exists and what must be reused. Do not overwrite working public form or payment code without understanding it.
3. Design and implement schema/migrations.
4. Implement authentication and secure API foundation.
5. Implement onboarding API first, including the complete detail endpoint and safe status/payment handling.
6. Connect the onboarding list, filters, summary, and View Details UI.
7. Implement clients and notes.
8. Implement deals/pipeline.
9. Implement finance and dashboard summary calculations.
10. Implement tasks, schedule events, settings, alerts, search, and CSV export.
11. Replace all local-only mutation paths with API requests.
12. Add loading/error/empty states and handle expired sessions.
13. Run syntax checks, automated tests, and manual UI checks at all target widths.
14. Fix every discovered issue; do not merely list bugs.
15. Produce the deliverables and exact run/deployment instructions.

## Final non-negotiable instructions

- Preserve the existing dashboard design and existing navigation IDs.
- Do not re-add demo data or sample restaurant records.
- Do not fake backend connectivity.
- Do not expose database credentials in the frontend.
- Do not trust client-side IDs, roles, amounts, payment statuses, or validation.
- Do not claim production-ready until tests and setup steps have actually been run.
- If the existing public QR ordering application form has fields not listed here, preserve those fields and document them. The actual form/database are the source of truth for submission compatibility.
- If something is ambiguous, inspect the repository and make the safest documented implementation choice instead of silently omitting functionality.

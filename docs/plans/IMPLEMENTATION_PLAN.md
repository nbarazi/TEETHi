# TEETHi Backend — Implementation Plan

This document is the single source of truth for building out the TEETHi domain (Patient App, Doctor App, Admin Dashboard) on top of the clean Laravel 13 base already bootstrapped in the `TEETHi-Backend` repo (Sanctum auth, `role`/`status` on `User`, standardized `ApiResponse` envelope, URI-versioned routes via `grazulex/laravel-apiroute`).

**How to use this file**: work phase by phase, top to bottom — each phase depends on the ones before it. Check items off (`- [x]`) as they land. This file lives in the root docs repo, not in `TEETHi-Backend`, so commit the plan change in the root repo alongside the backend commit that completes the item (mention that commit's hash). That keeps `git log -p docs/plans/IMPLEMENTATION_PLAN.md` usable as a progress history. Re-open a phase's items if a later phase reveals a gap.

---

## 1. Context

TEETHi is a dental services marketplace: patients discover and book dentists for at-home or emergency visits, sign consent forms, and pay; doctors manage their profile/availability/expertise and work incoming requests through to clinical documentation; admins run the treatment catalog and reconcile the business. The current repo has only generic scaffolding (auth, response envelope, role/status enums) — none of the dental domain exists yet. This plan defines every model, table, endpoint, and HIPAA control needed to get from "clean base" to "all three apps have a working backend," in dependency order, with a built-in way to track progress.

**Compliance framing**: patient clinical data, medical/dental history, medications & allergies, insurance numbers, and consent/legal documents are all PHI. Every phase below treats encryption, audit logging, and access-scoping as part of the feature, not an add-on at the end.

---

## 2. Decisions locked in for this build

These were open questions; each has a chosen answer that the plan below is written against. If a decision changes, re-check the phases it touches (noted in parentheses).

| # | Decision | Chosen approach | Phases affected |
|---|---|---|---|
| 1 | Email verification UX | **Switch to a numeric code** (matches the proposal), replacing the currently-built signed-URL flow. One polymorphic `otp_verifications` table serves both email and phone codes. | 0, 1 |
| 2 | Patient↔doctor messaging transport | **Polling-based REST for v1** (`GET messages?since=`). No WebSocket/broadcasting infra now; can be swapped in later without a data model change. | 8 |
| 3 | Family/dependent profiles | **In scope for v1.** An account holder (`User`) can manage more than one `PatientProfile` (self + dependents). All clinical/booking data still keys off `patient_profile_id`, so this is additive, not a redesign. | 1, 4, 5, 6 |
| 4 | Payment processing depth | **Invoicing + "pay later" fully implemented; card/Apple Pay processing stubbed** behind a `PaymentGatewayService` interface with a fake implementation. Swapping in a real gateway (Stripe, etc.) is an isolated follow-up phase. | 7, 11 (deferred) |
| 5 | Dashboard RBAC | **`User.role` stays a coarse platform gate** (patient/doctor/admin — "which app"). A separate, dashboard-only many-to-many RBAC system (`roles`, `permissions`, `role_user`, `permission_role`) is layered on top for "which dashboard actions this specific staff member can perform." Permissions are developer-defined/seeded (not creatable from the UI); role/permission *assignment* does get a minimal admin CRUD; a `roles.is_super_admin` flag bypasses all permission checks. Patients and doctors never participate in this system — only `role=admin` accounts do. | 2, 3, 5, 7, 9 |
| 6 | List endpoint pagination | **Every `GET` list endpoint returns a Laravel paginator**, never a bare `->get()`, via two small shared helpers: `ApiController::perPage($request)` (reads `?per_page=`, default 15, clamped to a max of 100) and `ApiResponse::paginated($resourceCollection)` (unwraps the paginator's `data`/`meta`/`links` into the standard `{success, message, data}` envelope, adding `meta`/`links` alongside it). Retrofitted onto every list endpoint built in Phases 0–3 on 2026-09-07 after a review found none of them paginated; every list endpoint from Phase 4 onward must use these helpers from the start. Phase 9's Spatie query-builder integration adds filtering/sorting *on top of* this base pagination, not instead of it. | 0, 1, 2, 3 (retrofit), 4–11 (going forward) |
| 7 | Timezones for availability & appointment times | **One IANA timezone per city, and exact instants on the wire.** `cities.timezone` holds an IANA name (e.g. `America/Los_Angeles`), with one config default for anything without a city. A doctor's availability slots and exceptions are wall-clock times in **their city's** zone. Every appointment has a **visit timezone**, resolved in this order: the doctor's city, then the patient profile's city, then the default. The server stores and compares instants in UTC, and converts to the relevant zone for every slot match and every "today" check. Availability responses give each slot's exact `starts_at` (ISO-8601 with the zone's offset on that date, so daylight saving is handled). Clients send that value back unchanged as `scheduled_at`, and never build a booking time from the device's own clock. Appointment responses carry `timezone` and `scheduled_at_local`, so the patient, dentist and admin all see the visit's local time, not their device's. Decided 2026-10-17 after the cross-project review found bookings from non-UTC devices failing the slot check (`docs/CONSISTENCY_REPORT.md` C1). Rejected: one platform-wide zone (doesn't scale past one region), a zone per doctor (tied to where the doctor's phone is, not where visits happen), and clock time plus zone on the wire (every client would need its own timezone library). | 4, 13 |
| 8 | Who confirms a rescheduled visit | **The party whose calendar is affected confirms.** A doctor's reschedule (picked from their own open slots) settles straight back to `accepted`, and the patient gets one notification. Both steps are kept in the status history. A patient's (or admin's) reschedule stays `rescheduled` and appears in the doctor's **Pending** list next to new bookings. Accept moves it to `accepted`; Decline moves it to `cancelled` with the doctor's reason (`declined` stays reserved for a fresh `pending` request). Only an `accepted` visit can be rescheduled, and starting a visit requires `accepted`. Decided 2026-10-18 after the live run found rescheduled visits could never be started (`docs/CONSISTENCY_REPORT.md` E2). Rejected: auto-accepting every reschedule, because it would let a patient move a visit without the doctor, while a new booking needs the doctor's acceptance. | 4 |

Other assumptions made without a dedicated question (revisit if wrong — see §7 Open Questions): one shared treatment catalog for both apps (not two audience-specific catalogs); `appointments.insurance_id` is a point-in-time snapshot reference, not a live one; admin authenticates through the same `/login` endpoint gated by `role:admin`, with dashboard RBAC (decision #5) providing the finer-grained access control within that gate rather than a separate auth surface.

---

## 3. Architecture overview

### 3.1 Route organization

```
routes/api/v1.php
  ├── (existing) public + authenticated auth routes
  ├── patient/   → auth:sanctum, active, role:patient, throttle:authenticated
  ├── doctor/    → auth:sanctum, active, role:doctor, throttle:authenticated
  ├── admin/     → auth:sanctum, active, role:admin, throttle:authenticated
  └── shared resource routes (appointments/{id}, conversations, treatments, notifications)
        → auth:sanctum, active, throttle:authenticated (Policy authorizes per object)
```

Role-prefixed groups own screens that are genuinely different per app (registration wizards, home dashboards, profile completion). Shared routes own the handful of entities more than one role touches (the appointment record itself, a message thread, the read-only catalog) — one controller, one Policy, role-varying API Resources — instead of duplicating CRUD three times.

### 3.2 Authorization

Three distinct layers, deliberately not collapsed into one:

- **`EnsureRole` middleware** (alias `role:patient|doctor|admin`) — route-level "which app" gate, based on the coarse `User.role` platform enum. Unchanged from Phase 0.
- **Dashboard RBAC** (§4.13, new in Phase 2) — *only* for `role:admin` accounts: a `roles`/`permissions` many-to-many system answering "which dashboard actions can this specific staff member perform." A dashboard user can hold multiple roles (e.g. Billing + Support). Enforced via a new `EnsurePermission` middleware (alias `permission:{slug}`) applied per-route inside `/admin/*`, alongside — not instead of — `role:admin`. This intentionally *does* reintroduce the RBAC-table pattern that Phase 0 stripped from the donor project, but scoped narrowly to dashboard staff, not patients/doctors (see §2 decision #5 for why).
- **Laravel Policies** per PHI-bearing/ownership-sensitive model for object-level "which row" gate — e.g. a doctor may only view a patient's clinical data if an `appointments` row links them. This layer is unrelated to dashboard RBAC and applies across all three apps.
- **Minimum necessary** is enforced in API Resources, not Policies: Policies stay binary (can/can't), Resources vary which fields are exposed to which viewer (`AppointmentResource`, `PatientProfileResource` branching on the requesting role).

Rule of thumb: `role:` answers "which app," `permission:` answers "which dashboard action," Policies answer "which specific row."

### 3.3 HIPAA controls baked into the base

- **Encryption at rest** via Laravel's `encrypted` / `encrypted:array` Eloquent casts on every PHI column (flagged per-table in §4). Where an encrypted column must stay searchable (e.g. a license number), add a companion HMAC `_hash` column and query that instead.
- **Append-only audit log** (`phi_access_logs`) covering every read/write/export of PHI, via a `LogsPhiAccess` trait applied to controllers, plus Eloquent model-event hooks for writes. The table itself is insert-only (an observer blocks `updating`/`deleting`).
- **Private file storage only** for this domain — the existing `local` disk (root outside the webroot, `serve => true` for signed temporary URLs) is the PHI-safe disk; every domain upload must target it explicitly, never the `public` disk. (Adding a second, identically-configured "private" disk was tried and reverted — Laravel refuses two `serve => true` local disks resolving to the same URL — so `local` *is* the private disk. This also resolves the leftover donor-project files sitting in `storage/app/public/*` — see Phase 0.)
- **No hard deletes for clinical/legal records** — `Appointment` uses status transitions; `LegalDocument` rows are never overwritten, only superseded; corrections to clinical notes are additive.
- **PCI scope reduction** — card data is never received by this backend; only gateway tokens are ever stored (`payment_methods.token`).

---

## 4. Data model

Conventions: snake_case tables, `id` PK, `created_at`/`updated_at` unless noted, FK columns `{singular}_id`. New enums follow the existing pattern in `app/Enums/UserRole.php` (backed string enum + `label()` i18n helper).

### 4.1 Identity, profiles & dependents

**`users`** (existing — migration-only additions)
- `+phone_number` string nullable unique, `+phone_verified_at` timestamp nullable

**`patient_profiles`** — one row per *person receiving care*, not one row per login. An account holder gets a profile with `is_account_holder = true`; dependents are additional rows managed by the same account.
- `user_id` FK **nullable** (set only when `is_account_holder = true`; null for dependents, who don't log in themselves)
- `managing_user_id` FK `users` **not null** (who books/manages this profile — the account holder, always)
- `is_account_holder` boolean
- `relationship_to_account_holder` enum(`spouse`,`child`,`parent`,`other`) nullable — null when `is_account_holder = true`
- `title`, `first_name`, `middle_name` nullable, `last_name`, `suffix` nullable
- `date_of_birth` date — **encrypt**
- `profile_note` text nullable — **encrypt**
- `languages` json nullable
- `city_id` FK → `cities`
- `profile_completed_at` timestamp nullable
- Unique partial index: one `is_account_holder = true` row per `user_id`.
- Model `PatientProfile`: belongsTo `User` (nullable, `user_id`), belongsTo `User` as `manager` (`managing_user_id`), hasOne `Address` (type=home), hasMany `InsuranceDetail`, hasOne `MedicalHistory`.

**`doctor_profiles`** (1:1 with `users`, doctors always log in themselves — no dependent concept here)
- `user_id` FK unique
- `title`, `first_name`, `middle_name` nullable, `last_name`, `suffix` nullable
- `date_of_birth` date — encrypt (lower sensitivity than patient DOB, still PII)
- `mobile_no` — **encrypt**
- `health_authority_registration_no` — **encrypt**, + `health_authority_registration_no_hash` for unique lookup
- `bio` text nullable, `education` json nullable, `experience` text nullable, `skills` json nullable, `honors_awards` text nullable, `social_link` string nullable, `languages` json nullable
- `city_id` FK nullable
- `years_experience` unsigned smallint nullable
- `average_rating` decimal(3,2) default 0, `patients_treated_count` unsigned int default 0, `recommended_percentage` unsigned tinyint default 0 — **denormalized, written only by a `ReviewObserver`/queued recompute job, never by direct API input**
- `profile_completed_at` timestamp nullable

**`specializations`** (admin lookup) — `id`, `name`, `is_active`
**`doctor_specialization`** pivot — `doctor_profile_id`, `specialization_id`

**`cities`** — `id`, `name`, `state`, `is_available`
**`city_requests`** — `user_id`, `city_name`, `status` enum(`pending`,`notified`), `notified_at` nullable — "my city not available → submit + get notified"

**`otp_verifications`** (unified email + phone code verification)
- `user_id` FK, `channel` enum(`email`,`phone`), `code` string (hashed, never stored plain), `expires_at`, `consumed_at` nullable, `attempts` unsigned tinyint default 0

### 4.2 Addresses (polymorphic, shared)

**`addresses`**
- `addressable_type`/`addressable_id` (morphs to `PatientProfile`, `DoctorProfile`, or `Appointment` for one-off visit addresses)
- `type` enum(`home`,`practice`,`visit`)
- `street`, `apartment` nullable, `note` nullable — **encrypt unconditionally, on every row regardless of owner** (revised in Phase 1 — see its "Design decisions" note: proximity search only ever needs the plain `latitude`/`longitude` below, and a conditional cast keyed off `addressable_type` proved fragile under Eloquent's attribute-assignment order)
- `city`, `state`, `zip`, `latitude`/`longitude` decimal nullable (required for proximity-sorted open requests)

### 4.3 Insurance

**`insurance_details`**
- `patient_profile_id` FK
- `provider_name`, `policy_number` — **encrypt**, `group_number` — **encrypt**, `subscriber_name` — **encrypt**, `subscriber_dob` — **encrypt**, `relationship_to_subscriber` enum(`self`,`spouse`,`child`,`other`)
- `card_front_path`, `card_back_path` nullable (private disk)
- `is_primary` boolean default false, `verified_at` nullable
- SoftDeletes (patients may replace insurance; keep history for billing disputes)
- `appointments.insurance_id` is a **point-in-time snapshot reference** (decision in §2) — captures which insurance was on file at booking time.

### 4.4 Consent / legal documents (shared abstraction, two tables)

**`document_templates`** (admin-managed source text)
- `type` enum(`consent_form`,`business_agreement`,`nondiscrimination_notice`)
- `audience` enum(`patient`,`doctor`,`both`)
- `title`, `body` text, `version` unsigned int, `is_active` boolean, `effective_at` timestamp
- Unique (`type`,`version`); only one `is_active` per `type` (service-layer enforced)

**`legal_documents`** (one row per signature or per uploaded credential — append-only, never overwritten)
- `user_id` FK (who signed/uploaded — always the account holder)
- `on_behalf_of_patient_profile_id` FK `patient_profiles` **nullable** — set when a consent form covers a *dependent's* care rather than the signer themselves
- `type` enum(`consent_form`,`business_agreement`,`nondiscrimination_notice`,`dental_license`,`liability_insurance`)
- `document_template_id` nullable FK (populated for the three templated/signed types)
- `file_path` nullable (private disk — license/insurance uploads)
- `metadata` json nullable — **encrypt** (`{license_number, issuing_authority, expiration_date}` or `{insurer_name, policy_number, coverage_amount, expiration_date}`)
- `status` enum(`pending`,`signed`,`submitted`,`verified`,`rejected`,`expired`,`superseded`)
- `signature_attestation` json nullable — **encrypt** (`{typed_name, ip_address, user_agent, signed_at, template_version}`)
- `submitted_at`, `verified_at` nullable, `verified_by` nullable FK `users`, `rejection_reason` text nullable, `expires_at` nullable
- `version` unsigned int default 1, `supersedes_id` nullable self-FK (re-signing/re-uploading creates a new row, never edits in place)

### 4.5 Treatment catalog

**`treatment_categories`** — `id`, `name`, `group` enum(`preventative`,`cosmetic`), `sort_order`, `is_active`
**`treatments`** — `treatment_category_id` FK, `name`, `description`, `standard_amount` decimal(10,2), `is_quick_select` boolean, `requires_tooth_chart` boolean, `is_active`, `created_by` FK `users`
**`doctor_treatment_expertise`** — `doctor_profile_id` FK, `treatment_id` nullable FK, `is_custom` boolean, `custom_title`/`custom_description` nullable, `expertise_level` tinyint 1–5 nullable, `is_active`

### 4.6 Availability

**`doctor_availability_slots`** — `doctor_profile_id`, `visit_type` enum(`at_home`,`emergency`), `day_of_week` 0–6, `start_time`, `end_time`, `capacity` default 1, `is_active`
**`doctor_availability_exceptions`** — `doctor_profile_id`, `date`, `is_available`, `start_time`/`end_time` nullable, `reason` nullable

### 4.7 Appointments

**`appointments`**
- `patient_profile_id` FK (self **or dependent** — both are `patient_profiles` rows, no special-casing needed)
- `booked_by_user_id` FK `users` (who actually made the booking — matters when booking for a dependent)
- `doctor_profile_id` nullable FK (null = unclaimed, in the open pool)
- `treatment_id` FK, `visit_type` enum(`at_home`,`emergency`), `request_mode` enum(`direct`,`open_pool`)
- `status` enum(`open`,`pending`,`accepted`,`in_progress`,`completed`,`rescheduled`,`cancelled`,`declined`)
- `requested_at`, `scheduled_at` nullable, `visit_address_id` FK → `addresses`
- `visit_instructions` text nullable — **encrypt**
- `insurance_id` nullable FK, `consent_confirmed_at` timestamp nullable
- `cancellation_reason`/`rescheduled_reason` text nullable — **encrypt**
- `completed_at` nullable
- No SoftDeletes — cancellation is a status, not a deletion.

**`appointment_status_histories`** — `appointment_id`, `from_status` nullable, `to_status`, `changed_by` FK `users`, `reason` nullable, `created_at` only (immutable audit trail)
**`appointment_treatments`** pivot — `appointment_id`, `treatment_id`, `notes` nullable

### 4.8 Clinical

**`medical_histories`** — `patient_profile_id` FK, `conditions`/`allergies`/`current_medications`/`notes` — all **encrypt**, `updated_by` FK nullable
**`clinical_notes`** — `appointment_id`, `doctor_profile_id`, `chief_complaint`/`clinical_finding`/`recommendation`/`quick_sketch_transcript`/`additional_notes` — all **encrypt**
**`doctor_patient_notes`** — `doctor_profile_id`, `patient_profile_id`, `note` — **encrypt**, `visibility` fixed `doctor_admin` (never visible to the patient)
**`treatment_plans`** — `appointment_id`, `doctor_profile_id`, `category` enum(`restorative`,`cosmetic`), `status` enum(`draft`,`finalized`), `notes` nullable — **encrypt**
**`treatment_plan_items`** — `treatment_plan_id`, `treatment_id` nullable, `custom_title`/`custom_description` nullable, `restorative_type` nullable enum(`ceramic_restoration`,`direct_restoration`,`implant_crown`,`bridge_3unit`,`bridge_4unit`), `tooth_chart_entry_id` nullable FK, `status` enum(`planned`,`in_progress`,`completed`,`pending`), `notes` nullable — **encrypt**
**`tooth_charts`** — `treatment_plan_id` FK unique
**`tooth_chart_entries`** — `tooth_chart_id`, `tooth_number` 1–32, `surfaces` json, `condition_code` string, `notes` nullable — **encrypt**; unique (`tooth_chart_id`,`tooth_number`)
**`prescription_templates`** (admin) — `name`, `body`, `is_active`, `created_by`
**`prescriptions`** — `patient_profile_id`, `doctor_profile_id`, `appointment_id` nullable, `prescription_template_id` nullable, `medication_name`/`dosage`/`instructions` — **encrypt**, `prescribed_at`
**`visit_outcomes`** — `appointment_id` unique, `services_completed`/`services_pending`/`medications_added` json — **encrypt**, `summary` nullable — **encrypt**
**`post_visit_recommendations`** — `appointment_id`, `treatment_id` nullable, `custom_title` nullable, `rank` 1–3 (max 3 enforced in service layer), `reason` nullable — **encrypt**

### 4.9 Messaging & notifications (polling-based, per §2 decision #2)

**`conversations`** — `patient_profile_id`, `doctor_profile_id`, `appointment_id` nullable, `last_message_at` nullable; unique (`patient_profile_id`,`doctor_profile_id`)
**`messages`** — `conversation_id`, `sender_id` FK `users`, `body` — **encrypt**, `attachment_path` nullable (private disk), `read_at` nullable, SoftDeletes
**Notifications**: use Laravel's built-in `notifications` table (`php artisan notifications:table`) — polymorphic `notifiable`, `type`, `data` json, `read_at`. Keep clinical detail out of notification payloads (generic "Appointment reminder", not "Reminder: your root canal at 3pm").

### 4.10 Payments (invoicing complete, gateway stubbed per §2 decision #4)

**`invoices`** — `appointment_id`, `patient_profile_id`, `doctor_profile_id`, `treatment_description`, `standard_amount`, `advance_paid` default 0, `remaining_amount`, `status` enum(`pending`,`partially_paid`,`paid`,`refunded`,`cancelled`), `due_at` nullable
**`payments`** — `invoice_id`, `amount`, `method` enum(`apple_pay`,`credit_card`,`pay_later`), `provider_reference` nullable, `status` enum(`initiated`,`succeeded`,`failed`,`refunded`), `paid_at` nullable
**`payment_methods`** — `patient_profile_id`, `provider`, `token` — **encrypt**, `brand`, `last_four`, `is_default`. Never store raw card numbers/CVV anywhere.

`PaymentGatewayService` is an interface (`charge()`, `tokenize()`, `refund()`) bound to a `FakePaymentGateway` in this phase (always succeeds, generates a fake `provider_reference`) so the invoice/payment data model and "pay later" flow are fully real and testable; a real Stripe/Braintree implementation is a later, isolated swap (Phase 11).

### 4.11 Reviews

**`reviews`** — `appointment_id` unique, `patient_profile_id`, `doctor_profile_id`, `doctor_rating` 1–5, `app_rating` nullable 1–5, `comment` nullable, `would_recommend` nullable boolean. On create: queued job recomputes `doctor_profiles.average_rating`/`recommended_percentage`/`patients_treated_count`.

### 4.12 HIPAA audit log (cross-cutting)

**`phi_access_logs`** — insert-only (observer blocks update/delete): `actor_id` FK `users`, `subject_user_id` nullable FK `users`, `resource_type`, `resource_id` nullable, `action` enum(`viewed`,`created`,`updated`,`deleted`,`exported`,`downloaded`), `ip_address`, `user_agent`, `context` json nullable (field names only, never PHI values), `created_at` only.

Applied via a `LogsPhiAccess` trait on every controller touching: patient profiles/dependents, medical history, clinical notes, treatment plans/tooth charts, prescriptions, insurance, legal documents, messages, admin patient/doctor directory views.

### 4.13 Dashboard RBAC (roles & permissions — dashboard-only, §2 decision #5)

No PHI here. This system is never consulted for patient/doctor authorization — only for `role:admin` accounts, and only to decide which dashboard screens/actions a specific staff member can use.

**`roles`** — `id`, `name`, `slug` unique, `description` nullable, `is_super_admin` boolean default false (bypasses every permission check)
**`permissions`** — `id`, `name` (slug, `{action}_{module}` convention, e.g. `view_invoices`, `manage_treatment_catalog`), `description` nullable. Developer-defined and seeded incrementally — each phase that introduces new `/admin/*` endpoints adds its module's permissions to `PermissionSeeder`; there is no "create a permission" UI.
**`permission_role`** pivot — `role_id`, `permission_id`
**`role_user`** pivot — `role_id`, `user_id` — many-to-many (a dashboard user may hold several roles); only ever populated for `users.role = UserRole::Admin`, enforced in the assignment endpoint/service, not a DB constraint (there's nothing structurally stopping a patient row from getting one, so the staff-assignment endpoint must check `role:admin` before allowing it)

`User` model gains: `roles(): BelongsToMany`, `hasRole(string $slug): bool`, `hasPermissionTo(string $permission): bool` (checks `is_super_admin` across the user's roles first, then falls through to a specific permission match), `getAllPermissions(): Collection<int, string>`. This mirrors the exact helper set the donor project had — reintroduced deliberately, scoped to dashboard accounts only.

---

## 5. API surface

All new controllers extend the existing `App\Http\Controllers\Api\ApiController` (already wires in `ApiResponse`), validate via Form Requests, shape output via API Resources, and authorize via Policies — matching `AuthController`'s existing style exactly. Every endpoint below that returns a list (an `index`/`GET`-collection action) paginates via `$this->perPage($request)` + `$this->paginated(...)` (decision #6) — never a bare `->get()`.

### 5.1 Patient app — `/api/v1/patient/...`

| Endpoint | Purpose |
|---|---|
| `POST/PATCH /profile` , `GET /profile` | Complete/update/view own patient profile (`missing_fields` for home-screen reminder) |
| `GET/POST/PATCH/DELETE /dependents` | Manage family member profiles (§2 decision #3) |
| `PUT /address` | Home address |
| `GET/POST/PATCH/DELETE /insurance` | Insurance details (per-profile, incl. dependents via `patient_profile_id` param) |
| `GET /consent-forms`, `POST /consent-forms/{template}/sign` | Consent forms (supports signing on behalf of a dependent) |
| `GET /providers`, `GET /providers/{doctor}` | Browse/select provider |
| `GET /treatments` | Catalog (shared, see §5.4) |
| `GET /availability` | Open slots for a doctor/visit_type/date |
| `POST /appointments`, `GET /appointments`, `GET /appointments/{id}` | Booking flow + list/view (choose `patient_profile_id` = self or dependent) |
| `PATCH /appointments/{id}/reschedule`, `PATCH /appointments/{id}/cancel` | |
| `POST /appointments/{id}/payment` | Card/Apple Pay (stubbed) or pay-later |
| `GET /appointments/{id}/recommendations`, `POST /appointments/{id}/review` | Post-visit |
| `GET/PUT /medical-history` (per profile) | |
| `GET /notifications` | |
| `GET /conversations`, `GET /conversations/{id}/messages`, `POST /conversations/{id}/messages` | Shared, see §5.4 |

**Implementation note (Phase 1):** email/phone OTP verification (`/email/verify`, `/email/resend`, `/phone`, `/phone/verify`, `/phone/resend`) live at the top-level auth group, not nested under `/patient/`, since both patient and doctor accounts use the exact same `AuthController` endpoints — nesting them per-role would have meant duplicating identical logic. `GET /cities` and `GET /specializations` are similarly top-level, shared, authenticated-any-role catalog reads (needed for the city-selection and specialization-browsing steps but never listed as their own endpoints in the original sketch above).

### 5.2 Doctor app — `/api/v1/doctor/...`

| Endpoint | Purpose |
|---|---|
| `POST/PATCH /profile`, `POST /specializations` | Profile + specialization selection |
| `PUT /address` | Practice address (symmetric addition — not in the original sketch, but Address is the same shared/polymorphic table as the patient's home address) |
| `POST /city-requests` | "My city not available" flow (addition — not in the original sketch) |
| `GET /profile/rewards`, `GET /profile/payments`, `GET /profile/completed-services` | Services Profile tab |
| `GET/POST /agreements`, `/licenses`, `/liability-insurance`, `/nondiscrimination` | All backed by `legal_documents` filtered by `type` |
| `GET/POST/PATCH/DELETE /services` | Treatment expertise (catalog + unlisted custom) |
| `GET/POST/PATCH/DELETE /availability`, `POST /availability/exceptions` | |
| `GET /requests/open?scope=nearby\|all`, `POST /requests/{appointment}/accept` | Open pool, proximity-sorted |
| `GET /requests/accepted` | Home-screen accepted requests |
| `PATCH /requests/{appointment}/reschedule` (reason required), `PATCH /requests/{appointment}/cancel` (reason required) | |
| `GET /patients/{patient}` (requires an appointment relationship), `POST /patients/{patient}/notes` | |
| `POST /appointments/{appointment}/clinical-notes` | |
| `POST /appointments/{appointment}/treatment-plan`, `POST /treatment-plans/{plan}/tooth-chart`, `PUT /tooth-charts/{chart}/teeth/{toothNumber}` | Per-tooth upsert + bulk save |
| `POST /appointments/{appointment}/prescriptions` | |
| `POST /appointments/{appointment}/visit-outcome`, `POST /appointments/{appointment}/recommendations` | |
| `GET /conversations`, messaging endpoints | Shared, see §5.4 |
| `GET /notifications` | |

### 5.3 Admin dashboard — `/api/v1/admin/...`

Every route below sits behind `role:admin` (platform gate) **and** the listed `permission:` (dashboard RBAC, §4.13/Phase 2) — a dashboard user can authenticate but still get a 403 from a specific screen if their assigned role(s) don't grant it.

| Endpoint | Purpose | Permission |
|---|---|---|
| Auth via shared `/login` gated by `role:admin` | | — (platform gate only) |
| `GET/POST /roles`, `GET/PATCH/DELETE /roles/{role}`, `POST /roles/{role}/permissions` (sync) | Manage dashboard roles (Phase 2) | `manage_roles` |
| `GET /permissions` | Read-only permission catalog, for building the role-editing UI | `view_roles` |
| `GET/POST /staff`, `GET /staff/{user}`, `POST /staff/{user}/roles` (sync) | Create/manage dashboard staff accounts + their role assignments (Phase 2) | `manage_staff` |
| `GET /appointments` (filter by date range/status) | | `view_appointments` |
| `GET /patients`, `GET /patients/{id}` (tabs: personal info/appointments/history/prescriptions/medical reports/invoices) | | `view_patients` |
| `GET /doctors`, `GET /doctors/{id}` | | `view_doctors` |
| `PATCH /doctors/{id}/status`, `PATCH /patients/{id}/status` | | `manage_doctors` / `manage_patients` |
| `PATCH /legal-documents/{id}/verify`, `PATCH /legal-documents/{id}/reject` | License/insurance verification | `verify_legal_documents` |
| `GET /invoices` (search/filter), `POST /invoices/{id}/reconcile` | | `view_invoices` / `manage_invoices` |
| `GET/POST/PATCH/DELETE /treatments`, `/treatment-categories` | "Add new treatment" catalog config | `manage_treatment_catalog` |
| `GET/POST/PATCH/DELETE /prescription-templates` | | `manage_prescription_templates` |
| `GET/POST /document-templates` | Consent/agreement text + versioning | `manage_document_templates` |
| `GET /phi-access-logs` | Compliance review | `view_phi_access_logs` |
| `GET/PATCH /cities`, `GET /city-requests` | | `manage_cities` |

The `is_super_admin` role bypasses every `permission:` check above, so the seeded Super Admin account never gets locked out as new permissions are added phase by phase.

### 5.4 Shared resource endpoints

- `GET/PATCH /appointments/{id}` — one controller, `AppointmentPolicy` differentiates patient-owner/managing-user, assigned doctor, admin; `AppointmentResource` varies fields by viewer.
- `GET /conversations`, `GET /conversations/{id}/messages`, `POST /conversations/{id}/messages` — polling (`?since=`), one controller for both apps, `ConversationPolicy` restricts to the two participants (+ admin, read-only).
- `GET /treatments` — public-ish catalog read.
- `GET /notifications`, `PATCH /notifications/{id}/read`.

### 5.5 Rate limits

`throttle:auth` (5/min) — registration/login-adjacent. New `throttle:otp` (6/min) — email/phone code send+verify. `throttle:authenticated` (120/min) — normal CRUD (existing). New `throttle:admin-export` (10/min) — `phi-access-logs` and any bulk export.

---

## 6. Phased roadmap (progress tracker)

### Phase 0 — Groundwork ✅ complete (2026-09-06)
- [x] Clear leftover donor-project files from `storage/app/public/{attachments,documents,logos,payment-files,profile-images}`
- [x] Standardize on the existing `local` disk (`config/filesystems.php`) as the PHI-safe private disk; all new uploads must use it exclusively, never `public`
- [x] Add `EnsureRole` middleware + `role` alias in `bootstrap/app.php`
- [x] Add `phone_number`, `phone_verified_at` to `users` (migration)
- [x] Replace signed-URL email verification with code-based flow (§2 #1): new `otp_verifications` table (`OtpVerification` model + `OtpService`), updated `AuthController::register`/`verifyEmail`/`resendVerificationEmail`, removed the `email/verify/{id}/{hash}` signed route in favor of `POST email/verify` with a `code` body param
- [x] Add `.env.example`/`.env` entries: SMS provider (OTP), push notification keys, geocoding/maps provider key (proximity sorting) — payment gateway keys deferred to Phase 11
- [x] Add `throttle:otp` and `throttle:admin-export` rate limiters
- [x] Scaffold `phi_access_logs` table + `LogsPhiAccess` trait (wired into the shared `ApiController` base, alongside `ApiResponse`) + immutability guard (`PhiAccessLog` blocks `updating`/`deleting`)

Verified: `vendor/bin/pint --test` clean, `phpstan analyse` (Larastan level max) clean, full Pest suite green (41 tests), migrations applied cleanly against the real `teethi` MySQL database in addition to the in-memory test DB.

**Note on decision §2 #1**: only the *email* side moved to code-based OTP in this phase (`otp_verifications.channel` supports `phone` too, but the phone send/verify endpoints and SMS delivery are Phase 1 work, once a provider is configured).

### Phase 1 — Identity, profiles & dependents ✅ complete (2026-09-07)
- [x] Migrations/models: `cities`, `city_requests`, `specializations`, `patient_profiles` (with `managing_user_id`/`is_account_holder`/`relationship_to_account_holder`), `doctor_profiles`, `doctor_specialization`, `addresses`
- [x] `PatientProfilePolicy` (self/managing-user or admin — `view`/`update`/`delete`, the last blocking the account holder's own row), `DoctorProfilePolicy` (anyone may `view`, owner or admin may `update`)
- [x] `EnsureRole` applied to `patient/` and `doctor/` route groups
- [x] Profile-completion `missing_fields` computed attribute (`App\Models\Concerns\TracksProfileCompletion`, shared by both profile models) + `syncCompletionStatus()` called on every profile update
- [x] OTP send/verify service extended to phone (`PUT /phone`, `POST /phone/verify`, `POST /phone/resend`) — email side was already done in Phase 0. Phone delivery goes through a placeholder `SmsLogChannel` (logs instead of texting) until a real SMS provider is configured; the rest of the flow (issue/verify/expiry/attempts) is fully real
- [x] Dependents CRUD (`GET/POST/PATCH/DELETE /patient/dependents`) — dependents are additional `patient_profiles` rows (`is_account_holder = false`), never reachable as the account holder's own row through this endpoint
- [x] Seeders: 8 UAE cities, 8 dental specializations
- [x] Additions beyond the original sketch: `PUT /doctor/address` (practice address, symmetric with the patient's home address), `POST /doctor/city-requests` ("my city not available"), shared `GET /cities` and `GET /specializations` catalog reads, `AddressService` (shared upsert-by-owner-relation logic for both home and practice addresses)
- [x] Tests: profile completion (patient + doctor), phone OTP flow, dependent CRUD + ownership/ownership-boundary checks, role-gated route access via real HTTP routes, address upsert, doctor registration-number uniqueness, specialization sync, city requests, catalog reads, and an `EncryptionTest` spot-checking every new encrypted column is never stored in plaintext

Verified: `vendor/bin/pint --test` clean, `phpstan analyse` (Larastan level max) clean, full Pest suite green (83 tests), migrations + fresh seed applied successfully against the real `teethi` MySQL database.

**Design decisions made during implementation (update §4.2 accordingly):**
- **Address encryption is unconditional**, not conditional on owner type as originally sketched. `street`/`apartment`/`note` are encrypted on every `addresses` row regardless of whether the owner is a `PatientProfile` or `DoctorProfile` — proximity search only ever needs the plain `latitude`/`longitude` columns, so there was no real reason to leave a doctor's practice street address unencrypted, and a conditional per-row cast keyed off a sibling attribute (`addressable_type`) would have been fragile under Eloquent's attribute-assignment order (e.g. `MorphOne::create()` sets the morph columns *after* the given attributes, so a cast inspecting `addressable_type` at set-time can't reliably see it yet).
- **Profile auto-creation lives on `User::booted()` (a `created` model event), not in `AuthController::register()`.** The first draft only auto-created a `PatientProfile`/`DoctorProfile` when a user registered through the API, which meant any other way of creating a `User` (factories, seeders, a future admin-provisioning endpoint) silently skipped it — a real bug caught by tests (`ModelNotFoundException` on `$user->patientProfile()->firstOrFail()`). Moving it to a model event makes "every patient/doctor account has a profile row" a genuine invariant of the `User` model itself, not a side effect of one specific code path. This is also why `DatabaseSeeder`'s `WithoutModelEvents` trait was removed — it would have silently suppressed this invariant during seeding.
- **`PatientProfile`/`DoctorProfile` factories must create their nested test `User` with `User::withoutEvents(...)`**, precisely because of the point above — otherwise the auto-created profile from the `User` event and the factory's own explicit profile insert collide on the `user_id` unique constraint.
- **`phone_verified_at` updates use `forceFill()->save()`, not `update()`** — like `email_verified_at`, it's deliberately excluded from `User::$fillable` so arbitrary input can never set it directly, which means the trusted, controller-originated writes in `verifyPhone()`/`updatePhoneNumber()` must explicitly bypass mass-assignment guarding.
- **A DB column default doesn't populate the in-memory model after `create()`** — `CityRequestController::store()` originally relied on `city_requests.status`'s DB-level default (`'pending'`) instead of passing it explicitly, which left the freshly-created instance's `status` attribute null (correct in the database, wrong in the API response) until fixed.
- A generic `App\Contracts\HasAddress` interface was tried (to let `AddressService` accept "any profile with an address") and abandoned: `MorphOne`'s declaring-model template isn't covariant, so no fixed interface contract could satisfy both `PatientProfile`'s and `DoctorProfile`'s own more-specific, natively-inferred `MorphOne<Address, $this>` return types under Larastan's level-max generics checking. `AddressService::upsert()` instead takes the `MorphOne` relation directly (e.g. `$profile->address()`) with its own method-level `@template`, which sidesteps the conflict entirely.

### Phase 2 — Dashboard RBAC (roles & permissions) ✅ complete (2026-09-08)
- [x] Migrations/models: `roles`, `permissions`, `permission_role`, `role_user`
- [x] `User::roles()`, `hasRole()`, `hasPermissionTo()` (super-admin bypass first), `getAllPermissions()`, plus a public `isSuperAdmin()` helper the Policies also rely on
- [x] `EnsurePermission` middleware (alias `permission:{slug}`), applied alongside `role:admin` — never on `patient/`/`doctor/` routes
- [x] `PermissionSeeder` seeding this phase's own permissions (`view_roles`, `manage_roles`, `manage_staff`); later phases append their module's permissions to the same seeder as they add `/admin/*` endpoints
- [x] Seed a "Super Admin" role (`is_super_admin = true`) and assign it to the existing `admin@teethi.test` account, via a new `RoleSeeder` run after `UserSeeder`
- [x] `RolePolicy`/`UserPolicy` (see naming note below) — only super-admins or `manage_roles`/`manage_staff` holders may write
- [x] Endpoints: `GET/POST /admin/roles`, `GET/PATCH/DELETE /admin/roles/{role}`, `POST /admin/roles/{role}/permissions`, `GET /admin/permissions`, `GET/POST /admin/staff`, `GET /admin/staff/{staff}`, `POST /admin/staff/{staff}/roles`
- [x] **Retroactive addition (2026-09-09, prompted by the admin dashboard frontend plan — see `docs/plans/ADMIN_DASHBOARD_FRONTEND_PLAN.md`):** `GET /admin/me` (`StaffController::me()` + `StaffMeResource`) — the acting staff account's own `is_super_admin` flag, flat `permissions` slug list, and `roles`. Deliberately the only `/admin/*` endpoint with **no** `permission:` middleware gate: every other endpoint requires a specific permission to even ask "what can I do," which made it structurally impossible for a staff account holding zero permissions to discover that fact (`GET /admin/staff/{id}` — the only endpoint that already returned roles+permissions — is itself gated by `manage_staff`). `permissions` is the concrete list attached to the account's roles; it is commonly **empty for a super-admin**, since the seeded "Super Admin" role carries no individual permissions of its own and bypasses every check purely via `is_super_admin` (confirmed via `tinker` against the real seeded account). Any frontend consuming this endpoint must check `is_super_admin` first, unconditionally, and never treat `permissions` as a complete list on its own.
- [x] **Retroactive addition (2026-09-09, same prompt):** `PATCH /admin/staff/{staff}` (`UpdateStaffRequest`, `UserPolicy::update`) — lets a `manage_staff` holder edit another staff account's `name`/`email`/`status`. Deliberately excludes `password` (an admin resetting someone else's password goes through the existing `POST /forgot-password` flow instead, addressed to that person, not a second "set anyone's credentials" surface) and guards against self-deactivation in `UpdateStaffRequest` (a lone super-admin setting `status: inactive` on their own account would lock everyone, including themselves, out of every `permission:`-gated screen at once — deactivating *other* accounts, including other super-admins, is unaffected).
- [x] **Retroactive addition (2026-09-09, same prompt):** top-level `GET /enums` (`EnumController`, no role/permission gate beyond authentication — shared by all three apps like `/cities`/`/specializations`/`/treatments`) — the full `value => localized label` map for every enum in the app, keyed by the same group names as `lang/{locale}/enums.php`, honoring `Accept-Language` like everything else. Implemented as a bare `trans('enums')` call rather than hand-listing every enum class: Laravel's translator returns a whole lang file's array unchanged when called with no specific nested key and no replacements, so this can never drift out of sync with `lang/{locale}/enums.php` (verified via `tinker` against the real app: 16 groups, correct per-locale labels). Not paginated — decision #6's pagination rule is about growable per-row resource lists, and this is a small, fixed reference catalog, not one.
- [x] Tests: super-admin bypass, permission grant/deny per role, role/permission CRUD, staff creation + role assignment, a non-super-admin without `manage_roles`/`manage_staff` is forbidden, a `role:patient`/`role:doctor` account can never be assigned a dashboard role, plus an `EnsurePermission` middleware unit test mirroring `EnsureRoleMiddlewareTest`; retroactively added: `GET /admin/me` (super-admin vs. a role with concrete permissions vs. no permissions at all), `PATCH /admin/staff/{staff}` (update, duplicate-email rejection, self-deactivation block, cross-account deactivation still works, non-admin-target rejection, missing-permission rejection), `GET /enums` (shape + locale)

Verified: `vendor/bin/pint --test` clean, `phpstan analyse` (Larastan level max) clean, full Pest suite green (107 tests), migrations + fresh seed applied successfully against the real `teethi` MySQL database (confirmed via `tinker`: `admin@teethi.test` is a super-admin and passes `hasPermissionTo()` for an undefined permission slug, while `patient@teethi.test` passes neither `hasPermissionTo()` nor `isSuperAdmin()`), `route:list --path=admin` shows all 11 routes. The three retroactive additions above were verified the same way at the time they landed (2026-09-09): Pint/PHPStan clean, full suite green (244 tests by that point), `route:list` shows `GET /admin/me`, `PATCH /admin/staff/{staff}`, and `GET /enums`, and each was spot-checked against the real MySQL-backed app via `tinker` (the `GET /admin/me` shape against the actual seeded `admin@teethi.test` account; `trans('enums')` returning all 16 groups correctly per-locale).

**Design decisions made during implementation (update §3.2/§4.13 accordingly):**
- **The staff-management Policy is named `UserPolicy`, not `StaffPolicy` as sketched.** Laravel resolves a model's Policy by convention — Model name + "Policy" — and this project has no manual policy registrations anywhere (`AppServiceProvider` is untouched), so a Policy meant to authorize `$this->authorize('view', $someUser)` must be named `UserPolicy` to be discovered at all. It only ever governs `role = UserRole::Admin` targets in practice (every ability checks the target's role), so it has no bearing on patient/doctor accounts elsewhere in the app.
- **Privilege-escalation guards live in FormRequests, not Policies**, using the same per-field `Closure` rule style as `UpdateDoctorProfileRequest`'s registration-number uniqueness check (Phase 1): `StoreRoleRequest`/`UpdateRoleRequest` reject setting `is_super_admin = true` unless the acting user already `isSuperAdmin()`, and `SyncStaffRolesRequest` rejects assigning any role that is *already* `is_super_admin` unless the actor is one. This closes the loop with `RolePolicy::update`, which separately blocks *any* edit to a role that is already super-admin unless the actor is one — together they stop both "promote an ordinary role to super-admin" and "touch the existing Super Admin role" from a `manage_roles`-only account.
- **The Super Admin role can never be deleted, by anyone, even another super-admin** (`RolePolicy::delete`) — a structural fixture rather than an ordinary row, to avoid a system that could accidentally lock every future `permission:` gate out.
- **`StaffController::store()` uses `User::forceCreate()`, not `create()`** — same reasoning as Phase 1's `phone_verified_at` fix: `email_verified_at` is deliberately excluded from `User::$fillable`, so a plain `create()` would silently leave a freshly provisioned staff account's email unverified in memory even though the intent (skip the OTP flow for dashboard-provisioned accounts) requires it set.
- Route model binding uses `{staff}` rather than `{user}` for the staff endpoints (`User $staff` in `StaffController`), purely for readability — it has no bearing on the currently-authenticated user resolved via `$request->user()` elsewhere in the same controller.

### Phase 3 — Treatment catalog & doctor expertise ✅ complete (2026-09-09)
- [x] Migrations/models: `treatment_categories`, `treatments`, `doctor_treatment_expertise`
- [x] Admin catalog CRUD (`FormRequest`s, gated by `permission:manage_treatment_catalog` — added to `PermissionSeeder`)
- [x] Seeders: full preventative/cosmetic catalog from the proposal, merged into one shared list (see design decisions below)
- [x] `GET /treatments` shared endpoint (quick-select filter, plus a `treatment_category_id` filter not in the original sketch)
- [x] Doctor expertise CRUD incl. custom/unlisted entries (`GET/POST/PATCH/DELETE /doctor/services`)
- [x] Tests: catalog CRUD (categories + treatments, permission gating), expertise assignment (catalog pick, custom entry, duplicate-claim rejection, cross-field validation), quick-select and category filtering

Verified: `vendor/bin/pint --test` clean, `phpstan analyse` (Larastan level max) clean, full Pest suite green (132 tests), migrations + fresh seed applied successfully against the real `teethi` MySQL database (confirmed via `tinker`: 2 categories, 20 treatments, 4 quick-select, 5 requiring a tooth chart, all attributed to `admin@teethi.test`), `route:list` confirms all 16 new routes.

**Design decisions made during implementation (update §4.5/§7.1 accordingly):**
- **Resolved open question #1 (treatment catalog divergence) by merging the proposal's two service lists into one shared catalog**, per the existing "one shared catalog" assumption in §2. The patient-facing (§2.5) and doctor-facing (§3.4) lists disagree on both naming and grouping — most notably "root canal" sits under Cosmetic for patients but "endodontics (root canal)" sits under Preventative for doctors — so `TreatmentSeeder` reconciles them under the doctor app's more clinically precise naming. `standard_amount` values are illustrative AED placeholders; an admin with `manage_treatment_catalog` can edit them via the CRUD this phase builds. Revisit if the business wants two audience-specific catalogs instead.
- **`Treatment.created_by` and `DoctorTreatmentExpertise.doctor_profile_id` are deliberately excluded from `$fillable`**, same defensive pattern as `PatientProfile.managing_user_id` (Phase 1) — both are authorization-relevant identifiers that must always come from the authenticated actor/controller, never client input, so both controllers use `forceCreate()`.
- **`DoctorTreatmentExpertise` needed an explicit `protected $table = 'doctor_treatment_expertise'`.** Eloquent's inflector doesn't know "expertise" is uncountable and guesses the table name `doctor_treatment_expertises`, which doesn't exist — a real bug that surfaced as a PHPStan error (`Access to an undefined property ...::$created_at`) before it could have become a runtime "table not found" error.
- **Mutual-exclusivity validation (catalog pick XOR custom entry) lives in `withValidator()`**, not `required_if`/`prohibited_if` — those rules do a loose string comparison against a literal value, which doesn't reliably match a real JSON boolean like `is_custom`. `StoreTreatmentExpertiseRequest` also proactively checks for an already-claimed treatment (a friendly 422) rather than letting the `doctor_treatment_expertise` unique constraint surface as a raw 500.
- **A treatment category/expertise entry has no dedicated Policy** — `TreatmentCategory`/`Treatment` are gated entirely by the `permission:manage_treatment_catalog` route middleware (no per-row ownership concept applies to a shared catalog), while `DoctorTreatmentExpertisePolicy` only guards the doctor-owned `doctor_treatment_expertise` rows, matching the Phase 1 precedent of Policies for per-row, not per-catalog, authorization.

### Cross-cutting fix — list endpoint pagination retrofit ✅ complete (2026-09-07)

A review after Phase 3 found that every `GET` list endpoint built so far (Phases 0–3) returned a bare `->get()` — unbounded, unpaginated collections. Fixed once, as a shared, reusable pattern (decision #6 in §2), then applied retroactively everywhere:

- [x] `ApiController::perPage(Request $request, int $default = 15, int $max = 100): int` — reads `?per_page=`, clamps to `[1, max]`
- [x] `ApiResponse::paginated(ResourceCollection $resource, string $message = 'responses.success'): JsonResponse` — calls `->paginate($this->perPage($request))` upstream, then unwraps the resource collection's own paginated response (`data`/`meta`/`links`) into the standard `{success, message, data}` envelope, with `meta`/`links` added alongside
- [x] Retrofitted onto all 10 existing list endpoints: `GET /cities`, `GET /specializations`, `GET /treatments` (shared), `GET /admin/roles`, `GET /admin/permissions`, `GET /admin/staff`, `GET /admin/treatment-categories`, `GET /admin/treatments`, `GET /patient/dependents`, `GET /doctor/services`
- [x] Tests: `?per_page=` is honored, out-of-range values clamp to the max, page 2 excludes page-1 items, and every retrofitted endpoint's response carries `meta`

Verified: `vendor/bin/pint --test` clean, `phpstan analyse` (Larastan level max) clean, full Pest suite green (135 tests), spot-checked live against the real `teethi` MySQL database (`GET /treatments?per_page=5` against 20 seeded rows correctly returned 5 items with `meta.last_page = 4`).

**Design decisions:**
- `ApiResponse::paginated()` keeps `data` as the resource collection *object* itself, not an array extracted from it — the first version extracted `data`/`meta`/`links` via `$resource->response()->getData(true)`, which broke `dedoc/scramble`'s OpenAPI generation for every paginated endpoint (it lost the static trace back to the underlying `JsonResource`, so `data` documented as an opaque `string` instead of e.g. `CityResource[]`, caught while regenerating `api.json` on 2026-09-07). `json_encode()` still resolves the embedded resource collection to the plain array of items at runtime (pagination-wrapping only happens through `toResponse()`, not through nesting inside another array), so the actual response body is unchanged — only `meta`/`links` still come from `getData(true)`, since there's no resource type for Scramble to trace there anyway.
- Every list endpoint from Phase 4 onward must use `perPage()`/`paginated()` from the start rather than `->get()`/`->success()`.

### Phase 4 — Availability & booking/appointments ✅ complete (2026-09-10)
- [x] Migrations/models: `doctor_availability_slots`, `doctor_availability_exceptions`, `insurance_details`, `appointments` (incl. `booked_by_user_id`), `appointment_status_histories`, `appointment_treatments`
- [x] `AvailabilityService` (template + exceptions + existing bookings → open slots, fixed 60-minute increments — see design decisions)
- [x] Proximity search for `GET /doctor/requests/open?scope=nearby|all` — Haversine computed in PHP, not raw SQL (see design decisions)
- [x] `AppointmentPolicy`; `BookingService` enforcing the missing-info gate (phone verified, date of birth present, consent confirmed) before creating the appointment
- [x] `AppointmentStatusService` — the sole writer of `appointments.status`, validating every transition against a state table and recording it to `appointment_status_histories`; `claimOpenAppointment()` uses a single atomic conditional `UPDATE` for race-safe open-pool claiming
- [x] Endpoints: `GET/POST /patient/insurance`, `GET/PATCH/DELETE /patient/insurance/{insurance}`, `GET /patient/availability`, `GET/POST /patient/appointments`, `GET/POST/PATCH/DELETE /doctor/availability`, `POST /doctor/availability/exceptions`, `GET /doctor/requests/open`, `GET /doctor/requests/accepted`, `POST /doctor/requests/{appointment}/accept`, `POST /doctor/requests/{appointment}/decline` (addition — symmetric with `accept`, not in the original sketch), shared `GET/PATCH /appointments/{appointment}/{reschedule,cancel}`
- [x] Tests: booking happy path (self and dependent, direct and open-pool), missing-info blocking (phone/DOB/consent, each independently), open-pool concurrent-claim atomicity, reschedule/cancel reason enforcement, availability CRUD + computation, proximity sorting, insurance CRUD incl. private-disk card uploads

Verified: `vendor/bin/pint --test` clean, `phpstan analyse` (Larastan level max) clean, full Pest suite green (193 tests), migrations + fresh seed applied successfully against the real `teethi` MySQL database, a `tinker` walkthrough of the full booking→accept flow (direct-mode booking → `pending`, `AppointmentStatusService::transition()` → `accepted`, 2 audit rows, treatment attached, visit address linked, `AvailabilityService` correctly showing 7 of 8 daily slots remaining after the booking), `route:list` confirms all 20 new routes, `scramble:analyze`/`scramble:export` regenerate `api.json` cleanly.

**Design decisions made during implementation:**
- **The missing-info gate's "consent signed" check is a self-attested `consent_confirmed` boolean on the booking request itself, not a real document lookup.** `legal_documents` doesn't exist until Phase 5, so `BookingService::missingRequirements()` only checks phone-verified and date-of-birth-present for now; `StoreAppointmentRequest` separately requires `consent_confirmed: true` (an `accepted` validation rule) and the booking sets `consent_confirmed_at = now()` unconditionally on creation. Phase 5 replaces this with a real "has the patient signed the required consent_form template" lookup inside `missingRequirements()` — no caller needs to change, since the method's return contract (a list of unmet-requirement translation keys) is unchanged.
- **No per-treatment duration exists in the catalog** (§4.5 `treatments` has no `duration_minutes`), so `AvailabilityService` slices every availability window into fixed 60-minute slots regardless of the treatment being booked — a deliberate MVP simplification. Revisit if the business needs per-treatment durations (e.g. a cleaning vs. a root canal taking different amounts of a doctor's day).
- **Proximity search computes Haversine distance in PHP, not a raw SQL trig expression.** The first version used a raw `acos()`/`radians()` SQL expression (MySQL-only), which the initial implementation validated only via `phpstan`/manual MySQL testing — it broke immediately against the test suite's SQLite connection (`no such function: least`), since SQLite has no built-in trig functions. The open pool is never large enough for an in-PHP sort to matter perf-wise, so `RequestController::open()` now fetches candidates plainly and sorts a Collection, manually building a `LengthAwarePaginator` over the sorted result. **Lesson: any future raw-SQL feature must be validated against the test suite's actual SQLite connection before being considered done, not just against the real MySQL one.**
- **A Laravel/Sanctum testing pitfall surfaced while writing multi-actor tests**: `Auth::guard('sanctum')` is a `RequestGuard`, which caches its resolved user for the lifetime of the guard instance (`if (! is_null($this->user)) return $this->user;`, with no reset on `setRequest()`). Since one Pest test method's `$this->app` (and therefore the guard) persists across multiple sequential `postJson()` calls, a **second request in the same test method with a different Bearer token silently re-authenticates as the first request's user** — it does not error, it just silently produces the wrong actor, which can make an assertion pass or fail for the wrong reason. Any test asserting behavior that depends on *which* authenticated user makes a request must use a separate `it()` block per actor, never multiple `withHeader(...)->postJson(...)` calls with different users' tokens inside one test. Two Phase 4 tests written this way were rewritten after being caught by this: one exercising three different viewers of an appointment, one exercising a doctor accepting an already-claimed request (the latter's intent — proving the claim is exclusive — is instead covered by a direct, non-HTTP unit test against `AppointmentStatusService::claimOpenAppointment()`, which sidesteps the whole issue).
- **`DoctorAvailabilityException.date` needed an explicit `date:Y-m-d` cast, not the bare `date` cast.** The bare cast round-trips through the connection's full datetime format on save (e.g. `"2026-09-14 00:00:00"`), but `AvailabilityController::storeException()` looks up the existing row via `updateOrCreate(['date' => $request->date], ...)`, comparing against the *raw* request string (`"2026-09-14"`) — a query-builder `where()` never goes through the model's cast. The mismatch meant `updateOrCreate` could never find the row it just created, and the second call collided with the `(doctor_profile_id, date)` unique constraint instead of updating in place. Caught by a test, fixed by making the cast's stored format match exactly what's searched for.
- **One new index's auto-generated name exceeded MySQL's 64-character identifier limit** (`doctor_availability_slots_doctor_profile_id_day_of_week_visit_type_index`) — invisible against the test suite's SQLite connection (no such limit there), only surfacing during the real-MySQL `migrate:fresh` verification step. Fixed with an explicit, shorter index name. Reinforces the same lesson as the raw-SQL portability point above: SQLite passing is necessary but not sufficient — the real MySQL connection is what actually catches connection-specific limits.
- **Reschedule/cancel are one shared top-level surface** (`PATCH /appointments/{appointment}/reschedule`, `/cancel`), not duplicated under both `/patient/appointments/{id}/...` and `/doctor/requests/{id}/...` as the original per-app sketch implied — `AppointmentPolicy` already differentiates patient-owner vs. assigned-doctor, so one controller/route pair serves both apps, consistent with §5.4's own "one shared surface, Policy differentiates" framing for `GET/PATCH /appointments/{id}`.

### Phase 5 — Consent/legal documents ✅ complete (2026-09-11)
- [x] Migrations/models: `document_templates`, `legal_documents` (incl. `on_behalf_of_patient_profile_id`)
- [x] `LegalDocumentPolicy`; signature capture + append-only versioning (`supersedes_id`)
- [x] Credential upload endpoints (license, liability insurance) → private disk; admin verify/reject workflow gated by `permission:verify_legal_documents`
- [x] Wire booking's "consent signed" gate (Phase 4) to this table
- [x] Admin `document_templates` CRUD (`permission:manage_document_templates`) + versioning (one active version per type)
- [x] Tests: sign flow (self + on behalf of dependent), re-sign on version bump, expiry query, admin verify/reject
- [x] Additions beyond the original sketch: `GET /admin/legal-documents` (an admin review queue needs some way to find submitted documents — the original sketch only listed verify/reject), a shared top-level `GET /legal-documents/{id}` show endpoint (same "one controller, Policy differentiates" pattern as `GET /appointments/{id}`), and `?status=`/`?type=` filters on both admin list endpoints

Verified: `vendor/bin/pint --test` clean, `phpstan analyse` (Larastan level max) clean, full Pest suite green (230 tests), migrations applied cleanly against the real `teethi` MySQL database, a one-off `artisan` command walkthrough of the full flow (publish v1 → sign → version bump deactivates v1 and the old signature stops satisfying the gate → re-sign against v2 supersedes the v1 row → doctor credential upload → admin verify → verifying an already-verified document correctly no-ops → `BookingService::missingRequirements()` returns empty once phone/DOB/consent are all satisfied), `route:list` confirms all 16 new routes, `scramble:analyze`/`scramble:export` regenerate `api.json` cleanly.

**Design decisions made during implementation:**
- **Two enums instead of one for document types**: `DocumentTemplateType` (the 3 admin-authored, signable kinds: `consent_form`, `business_agreement`, `nondiscrimination_notice`) and `LegalDocumentType` (those same 3, plus `dental_license`/`liability_insurance`, which have no template). Credential uploads never go through `document_templates` at all — a single shared enum would have let a template be created with a type that can never have one, which the type system should rule out rather than a runtime check.
- **Templates are never edited or activated separately — publishing *is* the only write.** `DocumentTemplateService::publish()` is `Admin\DocumentTemplateController::store()`'s only action (§5.3 only ever listed `GET`/`POST` for this endpoint, never `PATCH`): it auto-assigns the next `version` for the type (never client-supplied, avoiding a raw client value colliding with the `(type, version)` unique constraint) and, when the new version is marked active (the default), deactivates whichever version of that type was previously live — so "one active version per type" holds by construction, with no separate activate endpoint and no way to end up with two active versions of the same type.
- **Re-signing/re-uploading is how "editing" works for `legal_documents`, too** — `LegalDocumentService::sign()`/`uploadCredential()` are the only writers, and both follow the same shape: look up the most recent non-terminal row for that (user, on-behalf-of, type) combination, mark it `superseded`, then insert a new row with `version + 1` and `supersedes_id` pointing back. Nothing ever calls `update()` on a `legal_documents` row's identity/content columns — only `verify()`/`reject()` mutate an existing row in place, and only its workflow columns (`status`, `verified_at`, `verified_by`, `rejection_reason`), which is a state transition, not a rewrite of what was signed/submitted.
- **"Re-sign on version bump" needed no extra bookkeeping beyond the above.** `LegalDocumentService::hasSignedCurrentConsentForm()` checks for a `signed` row whose `document_template_id` matches the *currently active* consent form template's id specifically (not just "any signed row of type consent_form"). When admin publishes a new version, the patient's old signature still exists and is still `status = signed`, but its `document_template_id` now points to a deactivated template — so the gate correctly reports it unmet without anything needing to reach back and touch the old row. The old row only actually flips to `superseded` at the moment the patient re-signs, not at the moment the template version bumps.
- **`BookingService::missingRequirements()`/`book()` needed no interface change to wire in the real check** (per Phase 4's note that this was the intended seam) — `missingRequirements()` gained one more `if`, and `book()`'s `consent_confirmed_at` now comes from the signed document's `submitted_at` (when the patient actually signed) rather than `now()` (when this particular booking happened to occur). `StoreAppointmentRequest`'s interim self-attested `consent_confirmed: accepted` field was removed entirely, since the real gate supersedes it — every Phase 4 booking test's `completeBookingPrerequisites()` Pest helper (`tests/Pest.php`) had to be extended to also sign a consent form, split into a standalone `signConsentFormForBooking()` helper so the phone/date-of-birth gate tests could still isolate just the one requirement they're testing.
- **A `LegalDocumentResource` field can be a `null` PHP value stored through an `encrypted:array` cast just fine** — no surprises there; the one real wrinkle was the opposite direction: `DocumentTemplateResource` needed to render two different underlying shapes — a bare `DocumentTemplate` (admin's plain catalog listing, where "signed by whom" is meaningless) and a `['template' => ..., 'signed_document' => ...]` pair (the patient/doctor sign-flow listings, annotating each template with whether the *current* viewer already signed it). Resolved by branching on `is_array($this->resource)` inside `toArray()` rather than adding a second, near-duplicate Resource class.
- **PHP's `??` operator already suppresses "property access on null" the same way `isset()` does** — `$possiblyNull->prop ?? $default` never throws even when `$possiblyNull` is null, making `$possiblyNull?->prop ?? $default` (nullsafe *and* coalesce together) redundant; PHPStan's `nullsafe.neverNull` rule catches this. Every `($previous?->version ?? 0)`/`$consentDocument?->submitted_at ?? now()` construct in the new services was written with the nullsafe operator out of habit and had to be simplified to plain `->` once PHPStan flagged it — worth remembering for any future "nullable object, non-nullable-typed property, immediate `??` fallback" code, which is a common enough shape in this codebase's service layer.
- **`DocumentTemplate::factory()`/`LegalDocument::factory()` write through `forceCreate`-equivalent paths in tests, same as the controllers do** — both models exclude workflow-controlled columns (`version`, `is_active` on `DocumentTemplate`; almost everything except `file_path`/`metadata`/`signature_attestation` on `LegalDocument`) from `$fillable`, so a test helper that reaches for `firstOrCreate()` (which mass-assigns) rather than `forceCreate()` silently drops those columns and trips the `NOT NULL` constraint on `version` — caught while extending `tests/Pest.php`'s booking helper, fixed by querying first and falling back to an explicit `forceCreate()`.

### Phase 6 — Clinical documentation & tooth chart ✅ complete (2026-09-12)
- [x] Migrations/models: `medical_histories`, `clinical_notes`, `doctor_patient_notes`, `treatment_plans`, `treatment_plan_items`, `tooth_charts`, `tooth_chart_entries`, `prescription_templates`, `prescriptions`, `visit_outcomes`, `post_visit_recommendations`
- [x] Policies: `MedicalHistoryPolicy`, `ClinicalNotePolicy`, `TreatmentPlanPolicy`, `PrescriptionPolicy`, `DoctorPatientNotePolicy`, `VisitOutcomePolicy` — all enforcing "doctor must have an `appointments` link to this patient" (see design decisions for exactly how each resolves that check); `post_visit_recommendations` has no dedicated policy — it keys directly off `appointment_id` with nothing extra to authorize, so it's a new `AppointmentPolicy::recommend()` ability instead
- [x] Per-tooth upsert endpoint (`PUT /doctor/tooth-charts/{toothChart}/teeth/{toothNumber}`) + bulk chart save endpoint (`POST /doctor/treatment-plans/{treatmentPlan}/tooth-chart`), both routed through the same `ToothChartService::upsertTooth()` so a single tooth is always written through one path regardless of which endpoint triggered it
- [x] Post-visit recommendation cap-of-3 enforcement in `PostVisitRecommendationService`
- [x] `LogsPhiAccess` applied to every controller in this phase touching patient-specific clinical data — `Admin\PrescriptionTemplateController` is the one deliberate exception (reusable template *text*, no patient attached, same footing as `Treatment`/`TreatmentCategory` catalog CRUD from Phase 3)
- [x] Tests: cross-doctor access denial (`GET /doctor/patients/{patient}`, every clinical write endpoint), tooth chart CRUD (bulk save, per-tooth upsert-in-place, duplicate-tooth-number rejection, out-of-range tooth number), recommendation cap (a direct service-level test — see design decisions for why not HTTP), prescription-from-template
- [x] Additions beyond the original sketch: `PATCH /doctor/treatment-plans/{treatmentPlan}` (the only way a plan ever leaves `draft`), `GET /doctor/patients/{patient}` returns a full composite bundle (profile, medical history, shared appointments, treatment plans, prescriptions, this doctor's own notes) rather than a single resource, `GET /patient/appointments/{appointment}/recommendations` (the patient-side read of this phase's doctor-written recommendations, per §5.1's "Post-visit" row)

Verified: `vendor/bin/pint --test` clean, `phpstan analyse` (Larastan level max) clean, full Pest suite green (291 tests), all 11 migrations applied cleanly against the real `teethi` MySQL database, a full `artisan` command walkthrough of the entire phase (appointment → `hasTreatedPatient()` true for the assigned doctor / false for an unrelated one → treatment plan + catalog and custom items → bulk tooth chart save → per-tooth upsert updating the same row in place → prescription-from-template correctly filling `medication_name`/`instructions` from the template → 3 recommendations created, a 4th correctly rejected by the cap → plan finalized), `route:list` confirms all 18 new routes, `scramble:analyze`/`scramble:export` regenerate `api.json` cleanly.

**Design decisions made during implementation:**
- **`DoctorProfile::hasTreatedPatient(int $patientProfileId): bool`** is the one relationship check every policy in this phase ultimately reduces to, added directly on the model rather than duplicated per-policy: an `open`/`pending` request (never claimed/accepted) or a `declined`/`cancelled` one doesn't establish a real clinical relationship — only `accepted`/`in_progress`/`rescheduled`/`completed` do. `ClinicalNotePolicy`/`TreatmentPlanPolicy`/`PrescriptionPolicy`/`VisitOutcomePolicy` all check the narrower "is this the *specific* appointment's assigned doctor" (`$appointment->doctorProfile?->user_id === $user->id`) instead, since those four are always scoped to one appointment already; `DoctorPatientNotePolicy` and `MedicalHistoryPolicy`'s doctor-view branch use the broader `hasTreatedPatient()` check directly, since neither `doctor_patient_notes` nor "viewing a patient's history" is scoped to a single appointment.
- **`MedicalHistoryPolicy` authorizes against the owning `PatientProfile`, not a `MedicalHistory` instance** — a patient may not have written one yet, and "can this account see/edit patient X's history" doesn't depend on the row existing. Called via Laravel's class-hint array form, `$this->authorize('view', [MedicalHistory::class, $patientProfile])`, which resolves the Policy from the class string without requiring an instance of it — the same pattern used for `ClinicalNotePolicy`/`TreatmentPlanPolicy::create()`/`PrescriptionPolicy`/`VisitOutcomePolicy` (all authorize against the `Appointment` the not-yet-existing row would belong to) and `DoctorPatientNotePolicy` (against the `PatientProfile`).
- **`GET /patient/medical-history` is patient-only** (`role:patient` route middleware) even though `MedicalHistoryPolicy::view()` has a doctor branch — a doctor can never reach that route at all, so the doctor branch is only ever exercised through `GET /doctor/patients/{patient}`, which calls the same `$this->authorize('view', [MedicalHistory::class, $patient])` explicitly (redundant with that endpoint's own `hasTreatedPatient()` gate, kept anyway so the required Policy is the actual authority over that one field, not just an inline check standing in for it). A first draft of this phase's tests tried exercising the doctor branch through the patient route and failed for the right reason — the route's `role:patient` gate, not the Policy — which is what surfaced this.
- **Treatment plan items are created once, atomically with the plan** (`TreatmentPlanService::createWithItems()`, one `items: [...]` array in the same `POST`) — there's no endpoint to add or edit an individual item afterward. `PATCH /doctor/treatment-plans/{treatmentPlan}` (an addition — the sketch only had a create endpoint, which would otherwise leave every plan permanently `draft`) only ever touches `status`/`notes` on the plan itself. Deliberately scoped this way to keep an already-large phase's surface to what §6 actually asks for; per-item CRUD is a natural, separate follow-up.
- **The per-tooth upsert and the bulk chart save are the same code path.** `ToothChartService::upsertTooth()` is the sole writer of `tooth_chart_entries`; `createOrBulkSave()` (backing the bulk endpoint) just loops over its `teeth` array calling `upsertTooth()` once per tooth. A tooth's data is never written two different ways depending on which endpoint touched it.
- **Prescription-from-template resolves `medication_name`/`instructions` from the template's `name`/`body` only when the request omits them** (`PrescriptionService::create()`) — `dosage` always comes from the request, since it varies per patient even for the same medication/template. The request can still override either field explicitly alongside a `prescription_template_id`.
- **The cap-of-3 recommendation limit can't be exercised as a distinct 409 through the public API once all 3 ranks (1–3) are taken** — `rank` is validated to `between:1,3`, and `(appointment_id, rank)` is unique at the DB level, so the 4th recommendation for any appointment always collides with an already-taken rank (a `422`) before the cap check (a `409`) would ever get a chance to fire on its own through HTTP. Same shape as Phase 4's `claimOpenAppointment()` precedent: the cap itself is verified with a direct, non-HTTP test against `PostVisitRecommendationService::create()`, proving it returns `null` (and, critically, never even attempts the colliding insert) once 3 rows already exist — the HTTP-level tests instead cover the two things that *are* independently reachable that way: rejecting a duplicate rank on its own, and rejecting a catalog-XOR-custom violation.
- **`Admin\PrescriptionTemplateController` doesn't use `LogsPhiAccess`**, unlike every other controller in this phase — a prescription template is reusable boilerplate *text* with no patient attached (`created_by` is the only identifying column), the same footing as `Treatment`/`TreatmentCategory` catalog CRUD from Phase 3, which was likewise exempted.
- **`GET /doctor/patients/{patient}`'s nested arrays (appointments, treatment plans, prescriptions, notes) aren't paginated**, unlike every top-level `GET` list endpoint (decision #6) — they're bounded by one specific doctor-patient relationship, which in practice is always small, the same judgment already applied to `AppointmentResource.treatments`.

### Phase 7 — Payments/invoices (pay-later complete, gateway stubbed) ✅ complete (2026-09-13)
- [x] Migrations/models: `invoices`, `payments`, `payment_methods`
- [x] `PaymentGatewayService` interface (`App\Contracts`) + `FakePaymentGateway` binding, registered in `AppServiceProvider::register()`
- [x] "Pay later" invoice generation path (fully functional) — `InvoiceService::findOrCreateForAppointment()` lazily creates the invoice (derived from the appointment's attached treatments) the first time `POST /patient/appointments/{appointment}/payment` is called with any method, including `pay_later`
- [x] `InvoicePolicy`; admin invoice list/filter/reconcile endpoints gated by `permission:view_invoices`/`permission:manage_invoices`
- [x] Tests: pay-later flow end-to-end, invoice remaining-amount math (full payment, partial payment, admin-collected pay-later), fake-gateway charge/refund (including the refund path through `POST /admin/invoices/{id}/reconcile`)
- [x] Additions beyond the original sketch: `GET/POST/DELETE /patient/payment-methods` (saved cards — the schema's `payment_methods` table and `PaymentGatewayService::tokenize()` both needed *some* way to be exercised, and none was sketched), `AppointmentResource.invoice` (embeds the invoice — with its payments — directly into the existing shared appointment view, masked from doctor viewers same as `insurance`), a new `AppointmentPolicy::pay()` ability (only the patient-owner, never the assigned doctor or admin)

Verified: `vendor/bin/pint --test` clean, `phpstan analyse` (Larastan level max) clean, full Pest suite green (312 tests), all 3 migrations applied cleanly against the real `teethi` MySQL database, a full `artisan` command walkthrough (invoice lazily created and idempotent on a second call → partial card payment → final card payment moves the invoice to `paid` → admin refund reconciliation calls the fake gateway's real `refund()` and correctly reduces the balance by only the refunded payment, not the whole invoice → a separate pay-later appointment stays `pending` until an admin collects the deferred amount), `route:list` confirms all 6 new routes, `scramble:analyze`/`scramble:export` regenerate `api.json` cleanly.

**Design decisions made during implementation:**
- **The enum is `PaymentMethodType`, not `PaymentMethod`** — that name was already taken by the `PaymentMethod` *model* (a saved card/wallet), which is itself described by this enum's sibling column (`payment_methods.provider` is a separate free-text field; `payments.method` is the enum).
- **`Invoice`/`Payment` both have an empty `$fillable`** — there is no direct "create an invoice/payment" endpoint at all; `InvoiceService` is the sole writer of both, always via `forceCreate()`/`forceFill()`, the same pattern already used for `ToothChart` (Phase 6).
- **`pay_later` never touches the gateway or the invoice's balance** — it only records the patient's intent to defer via an `initiated` `Payment` row. An admin later collecting that deferred amount goes through `reconcile()`, which creates its own, separate `succeeded` row, so the audit trail shows both "chose to defer" and "actually paid" as distinct events rather than one row silently mutating into the other.
- **`InvoiceService::reconcile()` genuinely calls `PaymentGatewayService::refund()`** when moved to `status: refunded`, refunding the invoice's most recent successful gateway payment (card/Apple Pay) and creating a `refunded` `Payment` row from the result — without this, `refund()` would exist on the interface and the fake gateway but never actually run anywhere, which the checklist's "fake-gateway charge/refund" test target implied should be avoidable. It refunds only that one payment's amount, not the invoice's entire collected balance — real partial-refund semantics, not an all-or-nothing reversal. Every other `reconcile()` status change is applied directly with no transition graph, since this is an admin correction tool, not a patient-facing state machine.
- **A saved payment method is always attached to the *paying* account's own profile**, never the (possibly-dependent) profile the appointment being paid for belongs to. One saved card is meant to be reusable across every profile the account manages — `PayAppointmentRequest`'s ownership check already allows using any `payment_method_id` whose owning profile's `managing_user_id` matches the acting account, regardless of which specific profile that payment method happens to sit under — so there was no reason to duplicate a card record per dependent, and `POST /patient/payment-methods` deliberately has no `patient_profile_id` parameter.
- **Invoices/payments are not `LogsPhiAccess`-instrumented**, unlike almost every controller in Phases 4–6 — §4.12's explicit list of PHI categories requiring that trait (patient profiles, medical history, clinical notes, treatment plans/tooth charts, prescriptions, insurance, legal documents, messages) does not include billing records, and this phase followed that list literally rather than assuming "money" implies "PHI" here.

### Phase 8 — Messaging/notifications (polling) ✅ complete (2026-09-14)
- [x] Migrations/models: `conversations`, `messages`, Laravel `notifications` table (hand-written to the exact schema `notifications:table` generates)
- [x] `ConversationPolicy`/`MessagePolicy`; `GET messages?since=` polling endpoint
- [x] Notification classes: `AppointmentReminderNotification`, `AppointmentStatusChangedNotification`, `NewMessageNotification`, `DocumentExpiringNotification`, `NewNearbyOpenRequestNotification` — all `database`-channel only (no mail/SMS/push infra for these), all generic payloads with no clinical detail (§4.9)
- [x] Tests: conversation scoping (patient/doctor/admin), notification generation triggers (status change, new message, new nearby request, both scheduled reminder commands including their de-dup behavior)
- [x] Additions beyond the original sketch: `ConversationService::findOrCreateForAppointment()` (called from `BookingService::book()` for a direct booking and `AppointmentStatusService::claimOpenAppointment()` for an open-pool claim — the sketch's endpoints only ever operate on an *existing* conversation, so something has to create one first), `NotificationService` (the event-triggered half of the notification system — status changes and new messages), two scheduled console commands (`appointments:send-reminders`, `legal-documents:send-expiry-reminders`, registered in `routes/console.php`) for the two notifications that are batch/time-triggered rather than event-triggered, and `Address::distanceKmTo()` (the Haversine distance helper extracted out of `Doctor\RequestController` so `NotificationService`'s nearby-doctor lookup doesn't duplicate that math)

Verified: `vendor/bin/pint --test` clean, `phpstan analyse` (Larastan level max) clean, full Pest suite green (336 tests), all 3 migrations applied cleanly against the real `teethi` MySQL database, a full `artisan` command walkthrough (conversation lazily created and idempotent → a message notifies the other participant, not the sender → a doctor-initiated status change notifies the patient, not the doctor → claiming an open-pool request both auto-creates a conversation and fires the same status-change notification → a new open-pool booking notifies only the doctor within the nearby radius, not the far one), `route:list` confirms all 5 new routes, `scramble:analyze`/`scramble:export` regenerate `api.json` cleanly.

**Design decisions made during implementation:**
- **A conversation is per (patient, doctor) pair, never per appointment** — `conversations.appointment_id` only records which appointment first connected them; the `(patient_profile_id, doctor_profile_id)` unique constraint means a second appointment between the same two people reuses the existing thread. `Conversation`'s `$fillable` is deliberately empty (same pattern as `Invoice`/`ToothChart`) — `ConversationService` is the sole writer.
- **Viewing a conversation marks the other participant's messages read as a side effect** — there's no separate "mark message read" endpoint in the sketch, and this matches ordinary messaging-app UX. Guarded so only an actual participant's view does this; an admin's read-only glance at a conversation (§5.4) never touches the participants' own read receipts.
- **`JsonResource::whenCounted($relation)` looks for `{$relation}_count`, not `$relation`** — a real bug caught by a failing test: `ConversationController`'s `withCount(['messages as unread' => ...])` produced an `unread` attribute, but `ConversationResource` called `whenCounted('unread')`, which checks for `unread_count`. Fixed by aliasing the count itself as `unread_count`.
- **A message needs a body, an attachment, or both — never neither** — `messages.body` is nullable at the DB level (an attachment-only message, e.g. a photo with no caption, is a legitimate case) with the "at least one" rule enforced in `StoreMessageRequest::withValidator()`, the same shape as every other cross-field rule in this codebase.
- **The two scheduled reminder commands de-duplicate by querying the real `notifications` table for an existing row of that type, filtered to the relevant users, then comparing the JSON `data` payload in PHP** — not a raw `WHERE data->appointment_id = ?` JSON-path query, which would risk the exact MySQL/SQLite portability gap Phase 4's proximity search already ran into once. This also surfaced a test-writing lesson: a duplicate-prevention test must not wrap the run in `Notification::fake()`, since faking intercepts delivery before anything reaches the real table the de-dup check itself queries — asserting "sent once, not twice" needs the genuine `notifications` table, not the fake's in-memory record.
- **`Address::distanceKmTo()` replaces `Doctor\RequestController`'s private `distanceKm()` method** — `NotificationService`'s new-open-request nearby-doctor lookup needed the exact same Haversine calculation, so it was extracted onto the model both call sites already share, rather than duplicated a second time.

### Phase 9 — Admin dashboard aggregation & reporting ✅ complete (2026-09-15)
- [x] Spatie query-builder filters/sorts on every *new* admin list endpoint per proposal's table columns — all already permission-gated per §5.3/Phase 2; added each new list's permission (`view_appointments`, `view_patients`, `manage_patients`, `view_doctors`, `manage_doctors`, `manage_cities`, `view_phi_access_logs`) to `PermissionSeeder`
- [x] Patient/doctor detail composite endpoints (one endpoint per tab, not one giant payload)
- [x] `phi_access_logs` review endpoint (`permission:view_phi_access_logs`) + audited export
- [x] Tests: filter combinations, permission-based access enforcement, "viewing the audit log is itself logged"

Verified: `vendor/bin/pint --test` clean, `phpstan analyse` (Larastan level max) clean, full Pest suite green (366 tests, +30), no new migrations were needed (Phase 9 only adds read/status-toggle endpoints over existing tables) so the real `teethi` MySQL database's existing schema was confirmed via `migrate:status` (all prior migrations still `Ran`) rather than a fresh migrate; a full `artisan` command walkthrough against that live database (account-holder status toggle flips `users.status` while a dependent's missing `user_id` correctly identifies it as un-toggleable; the medical-reports tab's `ClinicalNote` query returns only notes from the target patient's own appointments, not a different patient's; a direct `PhiAccessLog` write round-trips through the real table; a doctor's status toggle and a city's availability toggle both persist; a live Spatie `QueryBuilder::allowedFilters()` query runs against real MySQL without error; and a real HTTP request through the kernel to `GET /admin/phi-access-logs` writes its own `PhiAccessLog::class`-typed audit row, proving the self-referential logging is a genuine side effect of that route and not just wired up in isolation), `route:list` confirms all 17 new routes, `scramble:analyze`/`scramble:export` regenerate `api.json` cleanly.

**Design decisions made during implementation:**
- **`Admin\PatientController` has one endpoint per tab** (`show`, `appointments`, `medicalHistory`, `prescriptions`, `medicalReports`, `invoices`, plus `updateStatus`) rather than one aggregate payload — the same "a dashboard screen only ever fetches the tab currently open" judgment already applied elsewhere. `Admin\DoctorController` has no tabs at all (just `index`/`show`/`updateStatus`) since the original sketch never described a tabbed doctor detail view the way it did for patients.
- **"Medical reports" is interpreted as the doctor-authored `ClinicalNote` entries across every appointment the patient has had**, kept on a separate tab from the patient-reported `MedicalHistory` — the two halves of a patient's clinical record the proposal's own tab list already keeps apart.
- **`account_status` was added to both `PatientProfileResource` and `DoctorProfileResource`**, `whenLoaded()`-guarded and additive/backward-compatible. For a patient it reads through the `manager` relation (a dependent has no `user_id` of its own, so the *managing* account's status is what's shown); for a doctor it reads through `user` (always present, 1:1). Neither resource populates the field unless the admin controller explicitly eager-loads the relevant relation, so every existing (non-admin) caller of these resources is unaffected.
- **Both status-toggle endpoints reuse the existing `UserStatus` enum and `users.status` column** — no new schema — via one shared `UpdateAccountStatusRequest`. A patient profile can only be toggled when it's an account holder (`user_id` not null); a dependent has no individual login of its own, so `Admin\PatientController::updateStatus()` aborts with a `422` (`__('admin.no_account_to_toggle')`) rather than silently affecting the whole family account. A doctor profile always has exactly one `User`, so `Admin\DoctorController::updateStatus()` needs no such guard.
- **`Spatie\QueryBuilder\QueryBuilder::allowedFilters()`/`::allowedSorts()` are variadic, not array-accepting** (`AllowedFilter|string ...$filters`) — confirmed by reading `vendor/spatie/laravel-query-builder/src/Concerns/FiltersQuery.php` directly, which does no array-normalization. An initial draft passed a single array argument to each across all four call sites (`Admin\AppointmentController`, `Admin\PatientController`, `Admin\DoctorController`, and the Phase 7 `Admin\InvoiceController` retrofit); PHPStan flagged the type mismatch before any test was written, catching what would otherwise have been a real runtime bug (the whole array treated as one filter) rather than a style nag. Fixed by passing comma-separated arguments with no wrapping brackets everywhere Spatie's builder is used.
- **Retrofitting `Admin\InvoiceController::index()` onto Spatie's query builder is a breaking change to its query-string shape** — Phase 7's ad-hoc `?status=`/`?patient_profile_id=` params became `?filter[status]=`/`?filter[patient_profile_id]=`, requiring the existing Phase 7 filter test to be updated to the new convention.
- **Spatie's query builder was deliberately applied only to Phase 9's own new list endpoints plus the Phase 7 invoice retrofit — not retroactively to every other existing admin list endpoint** (Roles, Staff, Treatments, Document Templates, Legal Documents, Prescription Templates from earlier phases). Those endpoints' existing ad-hoc filtering, where present, was left as-is; broadening the query-builder convention to them is a separate, explicit follow-up, not something this phase's checklist asked for.
- **"Viewing the audit log is itself logged"** — `Admin\PhiAccessLogController::index()` and `::export()` both call `logPhiAccess(PhiAccessAction::Viewed`/`Exported, PhiAccessLog::class, null, null, [...])` on *themselves*, so reviewing the audit trail can never become an unaudited blind spot in that same trail. Both routes additionally carry the stricter `throttle:admin-export` limiter (10/min, from Phase 0) layered on top of the admin group's default `throttle:authenticated`, since the audit log is the single most sensitive read in the system.

### Phase 12 — Patient-app directory, reviews, self-service records & delivery channels ✅ complete (2026-09-15)

Closes the gap between the patient app's UI (which assumes these features exist) and the API surface actually built in Phases 0–9 — none of the items below were ever implemented in any prior phase, including the two (`GET /providers`, `reviews`) that §5.1/§4.11 originally sketched.

- [x] `GET /patient/providers`, `GET /patient/providers/{doctor}` — patient-facing doctor directory browse/search (by name, city, specialization, treatment, "available today", and lat/lng "near me") and detail, backed by a new public-safe `ProviderResource` (no PHI — see Phase 4.12's list, which directory browsing was never on). `DoctorProfile` gained `displayName()`/`startingFee()`/`officeHoursRange()`/`isAvailableToday()`/`scopeAvailableOn()` decoration helpers; "near me" reuses `Address::distanceKmTo()` (Phase 8) and the same in-PHP sort-then-paginate pattern `Doctor\RequestController::open()` already established for the same reason (the matching set is never large enough for DB-level distance sorting to matter).
- [x] `reviews` table/model/policy + `POST /patient/appointments/{appointment}/review`, `GET /patient/providers/{doctor}/reviews` — §4.11's spec, finally built. A review recomputes `doctor_profiles.average_rating`/`patients_treated_count`/`recommended_percentage` synchronously in `Review::booted()`'s `created` hook (not a queued job — cheap indexed aggregate queries at this MVP's scale). One review per completed appointment, enforced by both a unique constraint and `ReviewPolicy`.
- [x] Patient self-service "Records" tab: `GET /patient/invoices`, `GET /patient/prescriptions`, `GET /patient/medical-reports` — the patient's own read side of what `Admin\PatientController` already exposed about a patient to an admin (Phase 9), scoped to every profile (self + dependents) the account manages via `managedPatientProfiles()`. Prescriptions/medical-reports are `LogsPhiAccess`-instrumented; invoices are not, consistent with Phase 7's "billing isn't PHI" decision.
- [x] Real phone-OTP delivery: `TwilioSmsChannel`, selected over the Phase 0 `SmsLogChannel` stand-in via `services.sms.driver` (reusing the `SMS_PROVIDER` env var already scaffolded in `.env.example`) — a config-only cutover, `OtpCodeNotification`'s issue/verify/expiry logic is untouched.
- [x] Push notifications: `device_tokens` table/model, `POST/DELETE /device-tokens` (role-agnostic, alongside the other shared notification routes), and a generic `FcmChannel` (FCM legacy HTTP API, `FCM_SERVER_KEY` — already scaffolded) added to all 5 Phase 8 notification classes' `via()` lists alongside `database`. Generic by design: it reads whatever `title` a notification already puts in its own `toArray()`, so adding push to a notification is a one-line `via()` change, no per-notification `toFcm()` method.
- [x] Addendum (2026-09-15, found while writing `docs/plans/PATIENT_APP_FRONTEND_PLAN.md`): `UserResource` (`/me`, `/login`, `/register`, `PUT /phone`) now exposes `phone_number`/`phone_verified_at` — previously absent from every resource in the codebase, which would have blocked the patient app's phone-verify screen and its booking "phone verified" gate check (§5.1/§5.4) from ever reading that state without a workaround.
- [x] Addendum (2026-09-15): `POST /patient/appointments` now accepts `treatment_ids: int[]` (was a single `treatment_id`) — a visit can cover more than one procedure (e.g. a cleaning + a filling) in one booking. `BookingService::book()` still writes the first selected id to `appointments.treatment_id` (kept as the "primary" treatment every existing one-treatment display already reads) and attaches every selected id to the `appointment_treatments` pivot — the same relation `InvoiceService` (Phase 7) already summed across for its `standard_amount`/`treatment_description`, so invoicing needed no changes at all. `AppointmentResource` gained a `treatments` array (the full set) alongside the existing singular `treatment` field, and every controller that eager-loads `treatment` for an `AppointmentResource` now also loads `treatments`.
- [x] Addendum (2026-09-15): `PATCH /notifications/read-all` — marks every unread notification for the caller read in one call, via the `Notifiable` trait's own `unreadNotifications()` relation. No per-notification loop needed client-side anymore.
- [x] Addendum (2026-09-16): `POST /patient/appointments` gained two optional fields, `tooth_number` (1-32, same universal numbering `ToothChartEntry.tooth_number` already uses) and `problem_description` (free text, encrypted like `visit_instructions`) — the emergency flow's tooth selector + complaint description, distinct from `visit_instructions` (logistics/access notes). Not restricted to `visit_type: emergency` — a routine booking may still want to flag a specific tooth. `AppointmentResource` exposes both, unmasked for every viewer (same treatment as `visit_instructions`).
- [x] Addendum (2026-09-16): `GET /patient/dashboard` — a single composite read for the home screen (profile, consent-pending flag, up to 5 upcoming appointments across every managed profile, unread notification count, and the quick-book treatment catalog), replacing what would otherwise be 4-5 separate calls on every app open. Every fact in it already had its own endpoint; this is purely a read-time aggregation with no new PHI exposure and no `LogsPhiAccess` calls of its own.
- [x] Addendum (2026-09-16): `POST /forgot-password`/`POST /reset-password` switched from Laravel's mail-link password-reset broker to the same OTP infrastructure every other verification step already uses. A new `OtpChannel::PasswordReset` case keeps a pending password-reset code from colliding with (or silently invalidating) a pending email-verification code for the same account, since `OtpService`'s lookup key is `(user_id, channel)`. `POST /reset-password` now takes `{ email, code, password, password_confirmation }` (a 6-digit code, not a mail-link token). `OtpCodeNotification` gained password-reset-specific mail copy (`otp_mail_subject_password_reset`/`otp_mail_intro_password_reset`) alongside the existing generic verification copy.
- [x] Addendum (2026-09-16): `PatientProfile` gained `gender`/`blood_type`/`bio` (mirroring `DoctorProfile.bio`'s pattern for the free-text field, encrypted like the existing `profile_note`; `gender`/`blood_type` are new enums, plain like every other categorical column) — settable via the existing `PATCH /patient/profile` and dependent CRUD endpoints, no new routes needed for these three. Separately, `POST /patient/profile/photo` (upload/replace) and `DELETE /patient/profile/photo` fill in `users.profile_image`, a column that existed on `User` since Phase 0 but had no writer anywhere in the codebase for any of the three apps. `UserResource.profile_image` now resolves to a short-lived signed URL onto the private `local` disk (the same pattern `InsuranceDetailResource`/`LegalDocumentResource`/`MessageResource` already use), rather than returning a raw, unusable storage path.

- [x] Addendum (2026-09-13): `GET /admin/patients` and `GET /admin/patients/{patient}` now expose `email`, `phone_number`, `is_email_verified`, `is_mobile_verified` on `PatientProfileResource`, read through the same `manager` relation `account_status` already uses — an admin viewing the patient directory previously had no way to see a patient's login contact details or verification state from either the list or detail view.

- [x] Addendum (2026-09-14): Every "send OTP" endpoint (`register`, `email/resend`, `PUT /phone`, `phone/resend`, `forgot-password`) can now echo the plaintext code back as `data.otp_code`, gated by `OTP_EXPOSE_IN_RESPONSE`/`config('services.otp.expose_in_response')` — off by default, and hard-blocked in production via `app()->isProduction()` regardless of the config value, so it can't leak codes even if left on by mistake. `OtpService::issue()` now returns the plaintext code directly (previously the `OtpVerification` model, whose `code` attribute is unreadable after creation since it's a hashed cast) — the only prior way to recover it was intercepting the notification in tests, which `tests/Pest.php`'s `issueOtpCode()` helper still does and is unaffected by this change. Exists purely to let the patient/doctor apps be manually tested against a real backend before SMS/email delivery is trustworthy in a given environment.

Verified: `vendor/bin/pint --dirty` clean, `phpstan analyse` (Larastan level max) clean, full Pest suite green (419 tests, +7 from this addendum), `route:list` confirms the new routes, every new migration applied cleanly against the real `teethi` MySQL database (no new routes or migrations were needed for the 2026-09-13/2026-09-14 addenda, which only touch `PatientProfileResource`, `AuthController`, and `OtpService`).

- [x] Addendum (2026-09-16): fixed a real production bug — any unauthenticated request whose client didn't happen to send `Accept: application/json` (a bare `Accept: */*`, which is what curl and some HTTP clients send by default unless told otherwise) crashed with a 500 `RouteNotFoundException: Route [login] not defined.` instead of a normal 401. Root cause: Laravel's own default bootstrapping (`ApplicationBuilder::withMiddleware()`) unconditionally registers `Authenticate::redirectUsing(fn () => route('login'))` *before* this app's own `bootstrap/app.php` closure ever runs — there was no previous override, and this API-only app (see `routes/web.php`) has no `login` route at all. `Illuminate\Auth\Middleware\Authenticate::unauthenticated()` calls that callback eagerly whenever `Request::expectsJson()` is false, crashing while constructing the `AuthenticationException` itself — before the exception handler (or the also-empty `withExceptions()`) ever got a chance to decide the response should be JSON. This had been silently masked the whole time: every test in this suite uses `getJson()`/`postJson()`, which always set `Accept: application/json`, so `expectsJson()` was always true in every test run and this path was never exercised — it only surfaced once a real client's token was invalidated (a full `migrate:fresh --seed`) and it made a plain unauthenticated request. Fixed with two changes in `bootstrap/app.php`: `$middleware->redirectGuestsTo(fn (): ?string => null)` (overrides the framework default so the `Authenticate` middleware never attempts a redirect at all) and `$exceptions->shouldRenderJsonWhen(fn (Request $request): bool => true)` (so every exception renders as JSON regardless of `Accept`, closing the same hole for 404s and any other exception type, not just auth failures). New `tests/Feature/UnauthenticatedResponseTest.php` exercises this with the framework's raw `get()`/`withHeader()` (not `getJson()`) so it can't be silently masked the same way again; confirmed it fails with the exact original exception when either override is reverted.

Verified: full Pest suite green (426 tests, +3 from this addendum), `phpstan analyse` clean, `vendor/bin/pint --dirty` clean, reproduced the original crash live against a running server (`curl` with a default `Accept: */*` and an invalid bearer token) before the fix and confirmed a clean `401 {"message":"Unauthenticated."}` after.

- [x] Addendum (2026-09-19, found by auditing the Flutter dentist app `TEETHI-Dentist-App` against the doctor API): six backend gaps blocking the dentist app's features, closed together.
  - **Visit lifecycle.** `POST /doctor/appointments/{appointment}/start` (accepted → in_progress, new `AppointmentPolicy::start`). `Doctor\VisitOutcomeController::store()` now closes the visit — it walks an `accepted` visit through `in_progress` to `completed` (the transition table has no direct accepted → completed edge); any other status is left alone since resubmitting an outcome is supported. Previously `in_progress`/`completed` were unreachable through any endpoint.
  - **Pending requests.** `GET /doctor/requests/pending` — direct bookings addressed to this doctor awaiting accept/decline. Neither `requests/open` (unassigned pool only) nor `requests/accepted` ever surfaced them, so a doctor had no way to discover one.
  - **Patient directory.** `GET /doctor/patients` (`?q=` name search) — every patient the doctor has engaged with, using the same status whitelist as `DoctorProfile::hasTreatedPatient()`.
  - **Doctor dashboard.** `GET /doctor/dashboard` — composite Home read (profile, next visit, today's schedule, open/pending counts, undocumented-visit count, expertise count, office hours, compliance items needing `sign`/`renew`, unread notifications).
  - **Patient chart.** `GET /doctor/patients/{patient}` now also returns the patient's `insurance`, the manager's email/phone/verification (the `manager` relation wasn't loaded, so those `PatientProfileResource` fields were silently absent), and the visits' `clinical_notes` (only the doctor's private `DoctorPatientNote` rows were returned before).
  - **Service radius.** New nullable `doctor_profiles.service_radius_km` (settable via `PATCH /doctor/profile`, exposed on `DoctorProfileResource`). The new `OpenRequestService` (shared by `requests/open` and the dashboard count) drops open requests beyond it, and `NotificationService` uses it instead of the hardcoded 50 km for "new nearby request" notifications (50 km remains the fallback when unset).

Verified: full Pest suite green (441 tests, +15 in `tests/Feature/Api/V1/Doctor/DentistAppSupportTest.php`), `phpstan analyse` clean, `pint --dirty` clean. **Not verified live, and pending:** the local MySQL server was down, so `php artisan migrate` (the new `service_radius_km` migration) has not been run against the dev database and `api.json` was not regenerated (`scramble:analyze` needs the DB) — run both once MySQL is up.

**Design decisions made during implementation:**
- **A doctor's "starting fee" and "hours" are computed on the fly from `DoctorTreatmentExpertise`/`DoctorAvailabilitySlot`, not new denormalized columns** — `doctor_profiles` has no `consultation_fee`/`working_hours` field, and adding one would duplicate data already derivable from tables Phases 4/6 built.
- **`available_today` in the directory is a coarse "has some recurring or one-off availability today" signal**, not a real open-slot check — a true check needs a `VisitType` (`AvailabilityService::openSlots()`, Phase 4), which a browse/search listing has no natural place to supply. The precise check still happens later, in the booking flow's own `GET /patient/availability` call.
- **Eager-load constraint closures passed to `Model::with()`/`::load()` must be typed `Illuminate\Database\Eloquent\Relations\Relation`, not `Illuminate\Database\Eloquent\Builder`** — a real bug caught by a failing test, not by PHPStan (Larastan's stubs accepted `Builder` without complaint, but Laravel invokes these closures with the actual `Relation` subtype, e.g. `HasMany`, at runtime; typing the parameter as `Builder` threw a `TypeError` on every request that triggered eager loading). No prior phase's codebase had a precedent for this pattern to copy from, since every earlier controller only ever used plain-string/array `with()`.
- **Reassigning a `DeviceToken` to a different account must be tested with only one HTTP call, seeding the "previous owner" state directly via a factory** — a second sequential `postJson()` call authenticating as a different user in the same test method hits the exact `Auth::guard('sanctum')` `RequestGuard` caching pitfall Phase 4 already documented above, silently re-authenticating as the first request's user instead of erroring.
- **`Review`'s `patients_treated_count` recompute reuses the same `AppointmentStatus` whitelist as `DoctorProfile::hasTreatedPatient()`** (`accepted`/`in_progress`/`rescheduled`/`completed`) rather than counting reviews — a doctor can have treated far more patients than have left a review, and the column's own name promises a treated-patient count, not a review count.

### Phase 13 — Timezone contract (decision #7)
Blocks reliable booking from any device not set to UTC. Client follow-ups are tracked in each frontend plan's "Timezone contract" section.
- [x] Migration: `cities.timezone` (IANA string, not null, default `America/Los_Angeles`). Seeder sets it for every city. Exposed on `CityResource`, and editable through `PATCH /admin/cities/{city}` with Laravel's `timezone:all` validation rule. Add a single config default for profiles with no city.
- [x] Zone resolution helpers: `DoctorProfile` returns its city's zone or the default. `Appointment` resolves the visit timezone (doctor's city, then the patient profile's city, then the default). Every other step below uses these helpers and nothing else.
- [x] `AvailabilityService`:
  - `openSlots()` reads `date` as a calendar date in the doctor's zone, including `day_of_week` and the exception date.
  - Booked appointments' `scheduled_at` is converted to the doctor's zone before counting them by date and start time (today it uses the UTC date and the UTC `H:i:s`).
  - `isSlotOpen()` converts the incoming instant to the doctor's zone before matching.
  - Each slot gains `starts_at`/`ends_at` (ISO-8601 with the offset in force on that date). The response gains `timezone`. `start_time`/`end_time` stay for display.
- [x] `StoreAppointmentRequest` and `RescheduleAppointmentRequest`: `scheduled_at` must carry an offset or `Z`. Reject a time with no zone with `422`, so a client bug can't silently fall back to the server's UTC.
- [x] "Today" uses the doctor's zone instead of the server's UTC date:
  - Doctor dashboard's today's visits (`Doctor/DashboardController`, `whereDate('scheduled_at', Carbon::today())`).
  - Directory `available_today` (`Patient/ProviderController`, and `DoctorProfile`'s today and day-of-week checks).
- [x] `AppointmentResource`, and every composite that embeds appointments (patient and doctor dashboards, admin directory): add `timezone` and `scheduled_at_local` (`YYYY-MM-DDTHH:MM:SS`, no offset, in the visit zone; `null` when `scheduled_at` is `null`). `scheduled_at` stays UTC.
- [x] `GET /doctor/availability/open-slots?date=&visit_type=`: the doctor's own open slots, same shape as `GET /patient/availability`. Reschedule already requires an exact open slot start, but the dentist app has no way to list one, so its free date/time picker can only succeed by coincidence.
- [x] No change needed: `SendAppointmentReminders` uses an absolute window (`now()+23h…24h`), which is correct for UTC instants.
- [x] Tests in a non-UTC zone (`America/Los_Angeles`):
  - Booking and rescheduling with a `starts_at` round trip.
  - A late-evening slot whose UTC date is the next day.
  - A daylight-saving date (2026-11-01).
  - Dashboard "today".
  - The `422` for a time with no zone.
- [x] Update the availability and appointment shapes in the patient, dentist and admin plans.
- [x] Regenerate `api.json` and refresh `docs/api/openapi.json`. **Verified (2026-10-18):** `php artisan migrate` applied `2026_10_18_000001_add_timezone_to_cities_table` to the local `teethi` database. That was the only pending migration; Phase 12's `service_radius_km` had already been applied. All 8 seeded cities now report `America/Los_Angeles`. `scramble:export` regenerated `api.json` (TEETHi-Backend `c8a8092` on `feat/timezone-contract`), and the same file was copied here. The spec now has 119 paths, including `GET /doctor/availability/open-slots`, and the `timezone`/`scheduled_at_local`/`starts_at` fields.

Built 2026-10-18 on TEETHi-Backend branch `feat/timezone-contract` (`25a38b4`, plus `9253d74` for the exception-date check). **Not merged.**

Verified: full Pest suite green (471 tests, +30: `tests/Feature/Api/V1/TimezoneContractTest.php` and 3 city tests in `Admin/CityManagementTest.php`), `phpstan analyse` (level max) clean, `pint --test` clean. Every new test freezes the clock in Los Angeles and covers: the `starts_at` round trip for booking (patient availability → `POST /patient/appointments`) and for reschedule (`GET /doctor/availability/open-slots` → `PATCH /appointments/{id}/reschedule`); a 22:00 slot stored as 06:00 UTC on the next date that counts against its local date only; the 2026-11-01 daylight-saving change (`-07:00` on 10-31, `-08:00` on 11-01, stored as 17:00 UTC); the doctor dashboard's today and the directory's `available_today` at 18:00 local (already the next day in UTC); the visit zone fallbacks; and the `422` for a `scheduled_at` with no zone on both endpoints. A mutation check (putting back the UTC `whereDate` in the dashboard and booked counts, and removing the UTC mutator) failed 9 of these tests. The migration was checked up → rollback → up on a scratch SQLite database (column added with its default, existing rows backfilled, `CitySeeder` sets every city, rollback drops only the column), not on MySQL. `composer test` as a whole stops at its first step, `rector --dry-run`, which flags 141 files, 119 of them untouched by this branch, so it already fails on `master`; its other steps (Pint, PHPStan, Pest) were run one by one.

**Design decisions made during implementation:**
- **`data` stays a plain list, and each slot carries its own `timezone`** (with `starts_at`/`ends_at`) instead of wrapping the list in an object. The patient app already reads `data` as a list, so the change is additive and nothing breaks before the clients move to `starts_at`.
- **`isSlotOpen()` compares instants, not clock strings.** The incoming time is converted to the doctor's zone only to find its local date; it must then equal one open slot's `starts_at` as an instant. Any offset naming the same moment is accepted (`…T17:00:00.000Z` books the `09:00-08:00` slot), and the ambiguous hour on the fall-back date can't match the wrong occurrence. Booked appointments are found by the local day's UTC bounds (`Appointment::scopeScheduledOnLocalDate()`, also used by the dashboard) and counted by local `H:i:s`, as specified.
- **A real storage bug, found while writing the round-trip test:** Eloquent's `datetime` cast formats a value in its own offset, so `2026-11-02T09:00:00-08:00` was stored as `2026-11-02 09:00:00` (read back as 09:00 UTC). `Appointment` now has a `scheduled_at` set-mutator that converts any string or Carbon to UTC first. `BookingService` and the reschedule controller needed no change.
- **`Appointment::visitTimezone()` reads two `HasOneThrough` relations, `doctorCity` and `patientCity`**, rather than `doctorProfile.city`. Eager-loading `doctorProfile` would have made `AppointmentResource` start rendering `doctor_profile` on endpoints that deliberately leave it out, so every list endpoint now eager-loads `doctorCity`/`patientCity` instead (no N+1, no shape change).
- **The directory's `?available_today=1` groups cities by their current local date** (`DoctorProfile::scopeAvailableToday()`) and applies the existing `availableOn()` per group, since doctors in different zones can be on different dates at once. `ProviderController` eager-loads exceptions for UTC yesterday to tomorrow, and `isAvailableToday()` picks the one for the doctor's own date.
- **Two more UTC "today" checks were moved to the doctor's zone, beyond the checklist's list** (decision #7 covers every "today" check): `GET /patient/availability`'s `date` (was `after_or_equal:today`, which rejected the doctor's own today from 4-5 pm in Los Angeles) and `POST /doctor/availability/exceptions`'s `date` (a doctor couldn't close the rest of their own day). Both now return `422 booking.date_in_past` against the doctor's local today. Date-of-birth and credential expiry checks still use the UTC date; they aren't visit times.
- **The zone-required rule is a regex on `scheduled_at`** (`Appointment::SCHEDULED_AT_PATTERN`: ISO-8601 date-time ending in `Z` or `±HH:MM`/`±HHMM`) with its own message, `booking.scheduled_at_needs_timezone`. The slot check is skipped when `scheduled_at` already failed a rule, so a zoneless time gets only that one error.
- **Tests:** factory cities default to `America/Los_Angeles`, the same as the migration and config, so the existing booking, reschedule and availability tests now build their slot times in Los Angeles rather than UTC. The new tests freeze the clock with a UTC instant (`losAngelesTime()`): Carbon keeps the test clock's zone for `now()`, and freezing with a Los Angeles Carbon made the app's `now()` run in Los Angeles, which hides exactly the UTC-server bugs this phase is about.

**Follow-ups found, not fixed:**
- `Admin\AppointmentController`'s `filter[scheduled_from]`/`filter[scheduled_to]` compare the raw string with the UTC `scheduled_at`, so a `YYYY-MM-DD` bound is a UTC date, not a local one (documented in the admin plan §4.8).
- `AppointmentReminderNotification`'s payload carries only the UTC `scheduled_at`, not `timezone`/`scheduled_at_local`.
- Open slots for today still include slots whose start has already passed; booking one then fails `after:now`.
- On the spring-forward date, a window covering the skipped hour (02:00-03:00 local) yields a slot whose `starts_at` PHP shifts to 03:00, the same instant as the next slot. Harmless unless a doctor works at 2 am.

### Phase 10 — HIPAA hardening close-out
- [ ] Reflection-based Pest test asserting encrypted casts on every flagged column (regression guard)
- [ ] Audit `LogApiRequests` middleware output — scrub PHI from logged request/response bodies
- [ ] `POST /sessions/revoke-all` (log out all devices), extending the existing token-revocation pattern from `changePassword`/`resetPassword`
- [x] Implement `withExceptions` in `bootstrap/app.php` — partially done (2026-09-16): `shouldRenderJsonWhen()` now forces every exception to render as JSON regardless of the request's `Accept` header, fixing a real production bug this uncovered (see Phase 12 addendum below) rather than the originally-scoped "clean JSON error envelope" wrapping. Every exception now reliably returns *some* JSON body, but 404/405/500 still render Laravel's raw `{message, exception, file, line, trace}` debug shape, not this app's own `{success, message, data}` envelope — wrapping those consistently is still open.
- [ ] Verify every file-serving endpoint uses signed/temporary URLs, never a public path
- [ ] Confirm soft-delete vs. hard-delete matrix (§4) is implemented consistently
- [ ] Run a security review pass before production rollout

### Phase 11 — Real payment gateway integration (deferred/optional)
- [ ] Choose gateway (Stripe recommended for combined card + Apple Pay support)
- [ ] Implement real `PaymentGatewayService` binding, swap out `FakePaymentGateway`
- [ ] Apple Pay merchant domain verification + session endpoint (if native Apple Pay pursued)
- [ ] Webhook handling for async payment status updates
- [ ] Tests: real gateway sandbox integration tests

---

## 7. Open questions (not blocking, revisit as they come up)

1. **Treatment catalog divergence** — the proposal lists slightly different item names under the doctor app's catalog vs. the patient app's (e.g. "root scaling" vs. not mentioned patient-side). Plan assumes one shared catalog; confirm before Phase 3 seeding if the business wants two audience-specific lists.
2. **Clinical note revision history** — plan assumes "last write wins + audit log," not a full `clinical_note_revisions` table. Revisit if legal defensibility requires seeing exact prior wording, not just that a change occurred.
3. **City semantics** — assumed to be a simple browse filter, not a jurisdiction implying different pricing/legal document versions per city. Confirm if regulatory variation by city is real.
4. **Admin token scoping** — admin currently reuses the same Sanctum token/login as patients/doctors, gated by `role:admin`. Revisit if admin needs tighter Sanctum token abilities/scopes for defense-in-depth.

# Teethi Dentist App — Flutter Frontend Wiring Plan

This document tells a Flutter engineer/agent **what exists in the Teethi backend today for the dentist (doctor) app, exactly how to call it, and in what order to wire the existing dentist-app UI to it.** It mirrors `docs/plans/PATIENT_APP_FRONTEND_PLAN.md` (same backend, same conventions) and intentionally contains no Dart/Flutter code or state-management guidance — only the API contract, the business rules behind it, a mock → real field mapping, and a phase-tracked scope.

**Starting point, precisely**: `D:\Projects\TEETHI-Dentist-App` already has its networking foundation — `ApiClient` (Dio), `TokenStorage`, `Result<T>`/`Failure`, `get_it` DI, `EnumsRepository`, all copied from the patient app — and **only the `auth` feature is wired to the real backend** (`POST /register` with `role: "doctor"`, `/login`, `/me`, email/phone verify, password reset). Every other feature (`home`, `requests`, `patients`, `patient_detail`, `schedule`, `dentist_profile`, `expertise`, `documents`, `messages`) reads from a `Mock<Feature>Repository` returning canned data from `features/common/data/mock_dentist_data.dart`. The mock repositories/entities were deliberately shaped to be swapped one feature at a time: implement `<Feature>Repository` against a new `<Feature>RemoteDataSource` and repoint that feature's `injector.dart` (see `docs/architecture/DENTIST_APP_ARCHITECTURE.md`). This plan is that swap, feature by feature.

**Backend status**: Phases 0–9 and 12 of `docs/plans/IMPLEMENTATION_PLAN.md` are complete, plus the **2026-09-19 dentist-app addendum** (Phase 12) that closed the six gaps found while auditing this app: visit start/complete, the pending-requests list, the patient list, the doctor dashboard, a fuller patient chart, and a per-doctor service radius. Phase 10 (HIPAA close-out) and Phase 11 (real payment gateway) are **not built** — there is no payout/earnings system of any kind, so anything payout-related in the mockup has nothing to call (§11, §16).

**How to use this file**: work phase by phase, top to bottom (§15). Check items off (`- [x]`) and add a short "Verified" note under each phase. If a backend detail below turns out wrong, fix this doc in the root docs repo as part of the same change that discovers it.

---

## 1. API fundamentals

Everything in `docs/plans/PATIENT_APP_FRONTEND_PLAN.md` §1 applies unchanged (Sanctum bearer tokens that never expire on their own, the four response envelopes, `?per_page=`/`?page=`, `Accept-Language`, rate limits). Doctor-specific points:

- **Base path**: `ApiConfig.baseUrl` defaults to `http://127.0.0.1:8000/api/v1` (matches `php artisan serve`). Doctor routes live under `/api/v1/doctor/...`; routes shared with the patient/admin apps sit at the top level and are called out below.
- **`role:doctor` gate**: every `/doctor/*` route returns a custom-envelope `403` for a non-doctor account. The app should never see it if login routes by `role`, but treat it as "wrong app for this account".
- **Every list endpoint is paginated (default 15, max 100).** This bites several screens here — the schedule grid, the expertise list, and any "all requests" list can exceed 15 rows. **Send `?per_page=100` for those** (§6, §5) or page properly; never assume the first page is everything.
- **Distances are kilometres on the wire.** The mockup shows miles (`Visit.distanceMiles`, `radiusMiles`). Convert at the edge (`mi = km × 0.621371`) and **send km** when saving a radius (§3). There is no distance field on `AppointmentResource` — compute it client-side (§7).
- **Doctor-side masking**: `AppointmentResource` deliberately returns `insurance: null` and `invoice: null` to a doctor viewer (minimum-necessary). Insurance for a patient is available on the patient chart instead (§9).
- **PHI**: chart/clinical endpoints write an audit row server-side as a side effect. No client action needed.

---

## 2. Authentication & onboarding

Already wired: `POST /register` (`role: "doctor"`), `POST /login`, `GET /me`, `POST /logout`, email verify/resend, `PUT /phone` + phone verify/resend, forgot/reset password, `POST /change-password`. Same contract as `docs/plans/PATIENT_APP_FRONTEND_PLAN.md` §2 (including the opt-in `data.otp_code` echo for QA, `OTP_EXPOSE_IN_RESPONSE`, never in production).

**The remaining piece is doctor onboarding.** `RegisterScreen` collects a license number, practice city and specialties, currently parked on `AuthViewModel.saveDentistApplicationExtras` with a `TODO(api)`. Every one of those has a real endpoint now — call them right after register (the token is issued immediately, before verification):

| Collected on Register | Endpoint | Body |
|---|---|---|
| Name (first/last), mobile | `PATCH /doctor/profile` | `{ first_name, last_name, mobile_no }` — `register`'s single `name` is not split server-side |
| License / registration number | `PATCH /doctor/profile` | `{ health_authority_registration_no }` — **must be globally unique** (`422` on a duplicate) |
| Practice city | `PATCH /doctor/profile` | `{ city_id }` — from `GET /cities` (only `is_available` cities) |
| Specialties | `POST /doctor/specializations` | `{ specialization_ids: [1, 4] }` — **replaces** the whole set; ids from `GET /specializations` |

- **Two different phone numbers**: `PUT /phone` sets and verifies the *account's* number (OTP), while `mobile_no` on `PATCH /doctor/profile` is the doctor's *profile* contact number (no verification). The Register screen's single mobile field should feed both.
- **A city that isn't offered**: `POST /doctor/city-requests` `{ city_name }` (fire-and-forget, `201`); admins see the queue. There is no status callback to the doctor.
- **Profile completeness is tracked server-side**: `DoctorProfileResource.missing_fields` lists what's still absent among `first_name, last_name, date_of_birth, city_id, mobile_no, health_authority_registration_no`; `profile_completed_at` is set once none are missing. **`date_of_birth` is required for completeness but the Register screen never collects it** — add it to onboarding or the profile screen (`PATCH /doctor/profile { date_of_birth: "1985-04-12" }`, must be before today) or the profile will never complete.
- **There is no approval gate in the API**: a registered doctor is `active` immediately. Whether an admin should approve a doctor before they see requests is a product question (§16).

---

## 3. Profile, practice address & service radius

| Endpoint | Purpose | Body |
|---|---|---|
| `GET /doctor/profile` | Own profile (with `city`, `address`, `specializations`) | — |
| `PATCH /doctor/profile` (also `POST`) | Partial update — send only what changed | any of `title, first_name, middle_name, last_name, suffix, date_of_birth, mobile_no, health_authority_registration_no, bio, education[], experience, skills[], honors_awards, social_link, languages[], city_id, years_experience, service_radius_km` |
| `PUT /doctor/address` | Create/replace the single **practice** address | `{ street, apartment?, note?, city, state, zip, latitude?, longitude? }` |

`DoctorProfileResource` (own-profile shape): `id, title, first/middle/last_name, suffix, date_of_birth, mobile_no, health_authority_registration_no, bio, education, experience, skills, honors_awards, social_link, languages, city, address, years_experience, service_radius_km, average_rating, patients_treated_count, recommended_percentage, specializations[], profile_completed_at, missing_fields, created_at, updated_at`.

- **`service_radius_km`** (new, nullable integer 1–500): how far from the practice address the doctor will travel. It now genuinely drives the app — `GET /doctor/requests/open` drops requests beyond it, and "new nearby request" notifications use it (falling back to 50 km when unset). This is the mockup's "radius" control on **Schedule** and **Home**. Send `null` to clear it (no limit).
- **Practice-address coordinates matter.** "Nearby" sorting/filtering only works if the practice `address` has `latitude`/`longitude`. **There is no geocoding service anywhere in this stack** — the same limitation the patient app hit. Use the device's location (`geolocator`, already a dependency) to pin the practice address, or the open-pool list silently falls back to newest-first with no radius applied.
- `average_rating`, `patients_treated_count`, `recommended_percentage` are **read-only** aggregates from patient reviews; never send them.
- The profile screen's `about`/`travel`/`payments`/`license`/`education` blocks are label/value rows assembled client-side from this resource plus §4's documents. **`payments` (payout method/bank details) has no backend at all** — see §16.

**Verified**: _(fill in once wired)_

---

## 4. Documents & compliance

The mockup's single "Documents" list is a **client-side merge of four existing endpoints** — no combined endpoint exists or is needed.

| Item | List | Submit |
|---|---|---|
| Business agreement | `GET /doctor/agreements` | `POST /doctor/agreements` `{ typed_name, agree: true }` |
| Non-discrimination notice | `GET /doctor/nondiscrimination` | `POST /doctor/nondiscrimination` `{ typed_name, agree: true }` |
| Dental license | `GET /doctor/licenses` | `POST /doctor/licenses` — **multipart**: `file` (pdf/jpg/png ≤ 5 MB), `license_number, issuing_authority, expiration_date` (must be in the future) |
| Liability insurance | `GET /doctor/liability-insurance` | `POST /doctor/liability-insurance` — **multipart**: `file`, `insurer_name, policy_number, coverage_amount, expiration_date` |

- **Agreements** are template-based: the list returns each active template (`DocumentTemplateResource`: `id, type, audience, title, body, version, effective_at, signed, signed_at`) — render `body`, and `signed` says whether *this exact version* is signed. Re-signing a newer version supersedes the old signature.
- **Credentials** are upload-based and return the doctor's **whole submission history** (`LegalDocumentResource`: `id, type, status, metadata, file_url, submitted_at, verified_at, rejection_reason, version, ...`), newest version first — including `rejected`/`superseded` rows. Use `rejection_reason` to explain a rejection.
- **Statuses**: `submitted` (awaiting admin) → `verified` | `rejected`; `signed` for agreements; `superseded` for replaced rows. **An admin verifies credentials** (`permission:verify_legal_documents`); nothing in the doctor app can trigger it, and the API doesn't gate the doctor's activity on verification (§16).
- **Deriving the mockup's `Sign` / `Renew` / `On file` badge** (no server field for it):
  - `Sign` — agreements: no signed row for the active template; credentials: no `submitted`/`verified` row (or the latest is `rejected`).
  - `Renew` — a credential whose `metadata.expiration_date` is in the past (or close). `expires_at` on the resource is **not populated** for uploads today; read `metadata.expiration_date`.
  - `On file` — otherwise.
- `file_url` is a **signed, ~30-minute URL** — never cache it; refetch to view. It is served from `/storage/...`, which has CORS enabled (relevant on Flutter web).
- `GET /legal-documents/{id}` (shared) fetches one of the doctor's own documents.
- `GET /doctor/dashboard` already returns a `compliance_pending` summary (§10) — use it for the Home banner instead of four calls.

**Verified**: _(fill in once wired)_

---

## 5. Expertise ("services")

`apiResource('services')` under `/doctor/services`: `GET` (list), `POST`, `GET/PATCH/DELETE /doctor/services/{id}`.

- **Catalog entry**: `{ is_custom: false, treatment_id, expertise_level? }`. **Custom (unlisted) entry**: `{ is_custom: true, custom_title, custom_description?, expertise_level? }`. Mixing the two is a `422`. `expertise_level` is 1–5 (the mockup's "level").
- A doctor may claim each catalog treatment **once** (`422` on a duplicate) — toggling a service off = `DELETE`, on = `POST`.
- `DoctorTreatmentExpertiseResource`: `id, is_custom, treatment{...TreatmentResource, incl. category}, custom_title, custom_description, expertise_level, is_active, created_at, updated_at`.
- The screen's two lists (`preventative`, `cosmetic`) come from `GET /treatments` (shared catalog, **also paginated — use `per_page=100`**), grouped by `category.group` (`preventative`/`cosmetic`, exactly the mockup's tabs). `selected` = the treatment ids present in `GET /doctor/services?per_page=100`.
- **Expertise drives what patients can find**: the directory's `starting_fee` is the cheapest catalog price across the doctor's active catalog entries, and directory filtering by treatment uses these rows. A doctor with no expertise rows shows no fee and matches no treatment filter.
- `standard_amount` is a flat catalog price — there is no per-doctor fee.

**Verified**: _(fill in once wired)_

---

## 6. Schedule / availability

| Endpoint | Purpose | Body |
|---|---|---|
| `GET /doctor/availability` | Recurring weekly windows — **paginated: send `per_page=100`** | — |
| `POST /doctor/availability` | Add a window | `{ visit_type, day_of_week, start_time, end_time, capacity? }` |
| `PATCH /doctor/availability/{id}` | Edit / deactivate | any of the above plus `is_active` |
| `DELETE /doctor/availability/{id}` | Remove | — |
| `POST /doctor/availability/exceptions` | One-off override for a date (upsert by date) | `{ date, is_available, start_time?, end_time?, reason? }` |
| `GET /doctor/availability/open-slots?date=&visit_type=` | This doctor's own **bookable** slots for one date — same shape as `GET /patient/availability` (below). Not paginated. What the reschedule picker offers (§7) | — |

- **Timezone (backend Phase 13, decision #7):** every window and exception time is a wall-clock time in the **doctor's city's zone** (`DoctorProfileResource.city.timezone`, an IANA name such as `America/Los_Angeles`; a doctor with no city yet uses the platform default, `America/Los_Angeles`). It is not the phone's zone, so a doctor travelling with their phone still edits their city's hours. Exception `date`s are calendar dates in that zone too.
- **Open-slot shape** (`open-slots`, and the patient app's `GET /patient/availability`): `data` is a plain list of `{ "start_time": "09:00:00", "end_time": "10:00:00", "starts_at": "2026-11-02T09:00:00-08:00", "ends_at": "2026-11-02T10:00:00-08:00", "timezone": "America/Los_Angeles" }`. `date` (`YYYY-MM-DD`) is a date in the doctor's zone; one before the doctor's own today is a `422` on `date` (`booking.date_in_past`). `starts_at`/`ends_at` carry the offset in force on that date (daylight saving included); a slot already at capacity is left out, including the one the visit being moved currently holds.
- `visit_type`: `at_home` / `emergency` (`GET /enums`). `day_of_week`: **0 = Sunday … 6 = Saturday** (Carbon), not Monday-first. Times are `HH:MM` (24 h) on write, `HH:MM:SS` on read.
- **A window is a range, not a slot.** The backend splits each window into **fixed 60-minute bookable slots** (`capacity` = how many bookings a slot holds; default 1) — there is no per-treatment duration, so `Visit.durationMins` is always 60. Booking availability is computed from these windows minus existing bookings and exceptions.
- **Mapping the mockup's grid** (`week` toggles + `timeSlots` + `closedSlots`): the mock models "a day is on/off, and individual hours can be closed". The API models ranges. Convert on load — for each `(visit_type, day)` expand active windows into hour cells — and on save, **coalesce the open hours back into contiguous windows** (a closed hour in the middle = two windows). Save by diffing against the loaded rows (`POST` new, `PATCH`/`DELETE` changed) rather than rewriting everything.
- **`atHomeEnabled` / `emergencyEnabled`**: there is no per-visit-type flag. "On" = the doctor has active windows of that `visit_type`; "off" = deactivate/delete them (`PATCH is_active:false` per window — **no bulk endpoint**, so N calls). This is a candidate backend convenience (§16).
- **Exceptions**: `is_available:false` closes the whole date (times ignored); `is_available:true` with `start_time`/`end_time` **adds** extra hours on a date (even one with no recurring window). `date` must be today or later **in the doctor's zone** (`422 booking.date_in_past` otherwise). **There is no `GET` for exceptions** — the app can write them but cannot list what it already created (§16).
- `baseAddress` / `radiusMiles` on the Schedule screen are the practice address and `service_radius_km` (§3), not availability fields.
- Patients see these windows through `GET /patient/availability` — a doctor with **no active windows is unbookable** for that visit type, exactly like the mock's empty state.

**Verified**: _(fill in once wired)_

---

## 7. Requests (open pool, direct requests, accepted work)

The Requests tab's three buckets (`RequestBuckets`: `open` / `accepted` / `pending`) now map 1:1 onto three endpoints — the `pending` one is new (2026-09-19); before it, direct bookings were invisible to the doctor.

| Bucket | Endpoint | What it returns |
|---|---|---|
| Open | `GET /doctor/requests/open?scope=nearby\|all` | Unassigned open-pool requests. `nearby` (default) sorts by distance from the practice address and **drops anything beyond `service_radius_km`** (if set); `all` skips distance logic (newest first). Requests with no coordinates sort last and are kept unless a radius is set and they can't be measured |
| Pending | `GET /doctor/requests/pending` | **Direct** requests addressed to this doctor, awaiting accept/decline, soonest `scheduled_at` first |
| Accepted | `GET /doctor/requests/accepted` | This doctor's active work: statuses `accepted`, `in_progress`, `rescheduled`, soonest first |

Actions:

| Action | Endpoint | Rules |
|---|---|---|
| Accept | `POST /doctor/requests/{id}/accept` | Open-pool: atomically claims it (`409` "already claimed" if another doctor won the race — handle this as a normal outcome, refresh the list). Direct: `pending → accepted` |
| Decline | `POST /doctor/requests/{id}/decline` `{ reason? }` | **Direct requests only** — an unclaimed open request can't be declined (there's nobody to decline to; just leave it). `reason` ≤ 500 chars |
| Start visit | `POST /doctor/appointments/{id}/start` | `accepted → in_progress`; `409` from any other status; `403` if not the assigned doctor |
| Reschedule | `PATCH /appointments/{id}/reschedule` *(shared)* | `{ scheduled_at, reason }`, reason required. `scheduled_at` must be **exactly an open slot's `starts_at`** from `GET /doctor/availability/open-slots` (§6), sent unchanged — any other time is `422 booking.slot_not_available`, and a time with no offset or `Z` is `422 booking.scheduled_at_needs_timezone` |
| Cancel | `PATCH /appointments/{id}/cancel` *(shared)* | `{ reason }`, required |
| Detail | `GET /appointments/{id}` *(shared)* | Full `AppointmentResource` |

`AppointmentResource` (doctor view): `id, status, visit_type, request_mode, treatment, treatments[], patient_profile, doctor_profile, requested_at, scheduled_at, timezone, scheduled_at_local, visit_address, visit_instructions, tooth_number, problem_description, insurance (always null for a doctor), invoice (always null for a doctor), consent_confirmed_at, cancellation_reason, rescheduled_reason, completed_at, created_at, updated_at`.

- **Visit times (backend Phase 13, decision #7):** `scheduled_at` is the UTC instant (e.g. `"2026-11-03T06:00:00+00:00"`); `timezone` is the visit's IANA zone (the doctor's city, then the patient profile's city, then the default) and `scheduled_at_local` is the wall-clock time there with no offset (`"2026-11-02T22:00:00"`, `null` when `scheduled_at` is `null`). **Show `scheduled_at_local`** and use its date for "is today"; keep `scheduled_at` for sorting and countdowns. Never `scheduled_at.toLocal()`.
- **Status → the mockup's `VisitStatus`**: `open` → `open` (or `urgent` when `visit_type = emergency`), `pending` → `pending`, `accepted`/`rescheduled` → `accepted` (or `today` when `scheduled_at_local`'s date is today), `in_progress` → `today`/active, `completed` → `completed`. `cancelled`/`declined` never appear in these three lists.
- **`Visit.cta`** ("Accept", "Start visit", "Reschedule", "Add reason") is derived client-side from status: open/pending → Accept (+ Decline for pending); accepted → Start visit; in_progress → open the visit workflow (§8).
- **`Visit.service`** = `treatment.name`, or the joined names of `treatments[]` when a booking has several (the first is the "primary"). `whenText`/`time` are formatted from `scheduled_at_local`; an **open-pool request has `scheduled_at: null`** ("ASAP" — emergency requests are always this).
- **`Visit.address`** = `visit_address` (street/city/...); **`distanceMiles` is not returned** — compute Haversine between the practice `address` lat/lng (§3) and `visit_address.latitude/longitude`, convert to miles. If either lacks coordinates, hide the distance rather than show 0.
- **Emergency requests**: a patient's emergency booking arrives as an `open` request with `visit_type: emergency` and usually `tooth_number` + `problem_description` — surface both on the card/detail. A push (`new_nearby_open_request`, §12) fires for doctors within their radius.
- Pull-to-refresh / polling is the only freshness mechanism (no websockets).

**Verified**: _(fill in once wired)_

---

## 8. Visit workflow & clinical documentation

All of these are scoped to **the assigned doctor** for that appointment (`403` otherwise) and write a PHI audit row.

| Step | Endpoint | Body |
|---|---|---|
| Start visit | `POST /doctor/appointments/{id}/start` | — (§7) |
| Clinical note | `POST /doctor/appointments/{id}/clinical-notes` | `{ chief_complaint?, clinical_finding?, recommendation?, quick_sketch_transcript?, additional_notes? }` (each ≤ 5000) — the mockup's three free-text fields map to the first three |
| Treatment plan | `POST /doctor/appointments/{id}/treatment-plan` | `{ category: restorative\|cosmetic, notes?, items: [{ treatment_id? , custom_title?, custom_description?, restorative_type?, notes? }] }` — ≥ 1 item; each item is a catalog pick **xor** a custom entry |
| Finalize plan | `PATCH /doctor/treatment-plans/{id}` | `{ status: draft\|finalized, notes? }` — an empty plan can't be finalized (`422`) |
| Tooth chart (bulk) | `POST /doctor/treatment-plans/{id}/tooth-chart` | `{ teeth?: [{ tooth_number 1-32, surfaces[], condition_code, notes? }] }` — duplicate tooth numbers are rejected |
| Tooth (single) | `PUT /doctor/tooth-charts/{chartId}/teeth/{toothNumber}` | `{ surfaces[], condition_code, notes? }` — upsert |
| Prescription | `POST /doctor/appointments/{id}/prescriptions` | `{ dosage, medication_name?, instructions?, prescription_template_id? }` |
| Visit outcome | `POST /doctor/appointments/{id}/visit-outcome` | `{ services_completed[], services_pending[], medications_added[], summary? }` — **also completes the visit** |
| Recommendations | `POST /doctor/appointments/{id}/recommendations` | `{ rank: 1-3, treatment_id?, custom_title?, reason? }` — the "recommended next" list the patient sees |

- **Submitting the visit outcome closes the visit**: an `in_progress` visit becomes `completed`; a visit still `accepted` is walked through `in_progress` first. Resubmitting overwrites the outcome in place and never errors on an already-completed visit. There is **no separate "complete visit" endpoint** — the outcome form *is* the complete action, so a "Complete visit" button should submit it. Only a `completed` visit can be reviewed by the patient.
- **Tooth numbering**: the API uses the **universal system, 1–32**. The mockup's FDI-style ids (`UR6`, `LL3`) are a display concern — keep a static two-way lookup table client-side. Round-tripping through it must be lossless.
- **`condition_code`** is a free string (≤ 50), not an enum — the backend imposes no vocabulary. The mockup's `ToothFlag { none, decay, watch, restored }` should be persisted as agreed codes (suggest `decay`, `watch`, `restored`; `none` = don't write / delete client-side). Agree the codes once and document them here; **there is no way to delete a tooth entry** (§16), so "clear" must be a written `none`-equivalent or simply omitted on the next bulk save.
- **Treatment plan categories** are `restorative`/`cosmetic` (`GET /enums` → `treatment_plan_category`) — an exact match for the mockup's `TreatmentCategory`. `restorative_type` values: `ceramic_restoration`, `direct_restoration`, `implant_crown`, `bridge_3unit`, `bridge_4unit`. Plan items have their own status (`planned`/`in_progress`/`completed`/`pending`), returned but not writable through these endpoints.
- **The mockup's `TreatmentPlanItem.added` toggle** ("Add to plan") = including that item in the `items` of a plan `POST`; a plan is created per appointment + category, so a patient's restorative and cosmetic lists are two plans.
- **Plans belong to an appointment**; the chart's `treatment_plans` (§9) return every plan across the doctor's visits with that patient.
- **`ClinicalNote.isComplete`** (mock) has no server field; derive it (all three text fields non-empty) or drop it.
- Order the UI enforces is a UX choice — the API does not require a note before a prescription, etc. It only requires the appointment to belong to the doctor.

**Verified**: _(fill in once wired)_

---

## 9. Patients

| Endpoint | Purpose |
|---|---|
| `GET /doctor/patients?q=` | The Patients tab: every patient this doctor has engaged with (paginated, ordered by last name). `q` matches first/last name |
| `GET /doctor/patients/{patientProfileId}` | One patient's full chart |
| `POST /doctor/patients/{patientProfileId}/notes` | Private doctor note about the patient `{ note }` (≤ 5000) |

- **"Engaged with"** = the doctor has at least one appointment with that patient in `accepted`/`in_progress`/`rescheduled`/`completed`. A merely `pending`/`declined`/`cancelled` request does **not** make someone a patient, and opening their chart is `403`. The list and the chart gate use the same rule, so a listed patient is always openable.
- `{patient}` is a **`PatientProfile` id**, not a user id — a family account has one profile per person, and each is its own patient.
- The list items are `PatientProfileResource` (with `city` and the managing account's `email`, `phone_number`, `is_email_verified`, `is_mobile_verified`, `account_status`). `Patient.phone`/`phoneVerified` come from these; `age` = compute from `date_of_birth`; `languages`, `gender`, `blood_type`, `bio` are included.
- **The mockup's `patientStats`** (e.g. counts) have no endpoint — derive from `meta.total` and the loaded lists, or drop them.

`GET /doctor/patients/{id}` returns:
```json
{
  "patient": { /* PatientProfileResource incl. city, address, email, phone_number, is_mobile_verified */ },
  "insurance": [ /* InsuranceDetailResource[] — provider_name, policy_number, group_number, subscriber_name, ... */ ],
  "medical_history": { /* conditions, allergies, current_medications, notes, ... */ } | null,
  "appointments": [ /* this doctor's appointments with this patient, newest first */ ],
  "treatment_plans": [ /* with items[] and tooth_chart{ entries[] } */ ],
  "prescriptions": [ /* newest first */ ],
  "clinical_notes": [ /* ClinicalNoteResource[], newest first */ ],
  "notes": [ /* this doctor's private DoctorPatientNoteResource[] */ ]
}
```

Mapping to `PatientDetailBundle`:
- `patient` ← `patient` + `insurance[0]` (prefer the `is_primary` one): `insuranceProvider` ← `provider_name`, `insuranceMemberId` ← `policy_number`, `insurancePlan` ← `group_number` (there is no separate "plan name" field). No insurance = empty strings, not an error.
- `allergies` / `medications` ← `medical_history.allergies` / `.current_medications`. **These are free-text strings on the wire**, not arrays — split/format client-side for the chip UI.
- `history: [VisitHistoryEntry]` ← `appointments` (title = treatment names, date = `scheduled_at`/`completed_at`, provider = this doctor), with `findings` ← the `clinical_notes` entry whose `appointment_id` matches (its `clinical_finding`). Join client-side.
- `note: ClinicalNote` ← the most recent `clinical_notes` entry (or the one for the visit being documented).
- `teeth` ← the `tooth_chart.entries` of the most recent plan that has a chart (universal → FDI, §8). `restorative` / `cosmetic` ← items of the plans with that `category`.
- **Insurance appears here but is masked on `AppointmentResource`** — this is deliberate: the chart is the one place a treating doctor sees coverage details.
- `photoUrl`: patient profiles have **no photo on the wire** (`profile_image` lives on the patient's `UserResource`, which the doctor doesn't receive). Show initials/avatar placeholder (§16).

**Verified**: _(fill in once wired)_

---

## 10. Home dashboard

`GET /doctor/dashboard` is the doctor-side counterpart of the patient app's composite read — the mock's `HomeRepository.dashboard()` comment already says it "mirrors the shape a real `GET /doctor/dashboard` would return". One call replaces ~6.

```json
{
  "profile": { /* DoctorProfileResource */ },
  "unread_notifications_count": 2,
  "next_visit": { /* AppointmentResource */ } | null,
  "todays_schedule": [ /* AppointmentResource[] — accepted/in_progress/rescheduled scheduled today in the doctor's zone, by time */ ],
  "open_requests_count": 3,
  "open_requests_radius_km": 8,
  "pending_requests_count": 1,
  "undocumented_visits_count": 1,
  "expertise_service_count": 6,
  "office_hours": "9:00 AM - 5:00 PM" | null,
  "compliance_pending": [ { "type": "dental_license", "action": "sign" }, { "type": "liability_insurance", "action": "renew" } ]
}
```

- **`unread_notifications_count` is a real number here** (unlike the patient plan's note about computing it client-side, which predates the dashboards).
- `open_requests_count` uses the same radius/distance logic as `GET /doctor/requests/open`, so the tile and the list agree. `open_requests_radius_km` is `null` when no radius is set — convert to miles for the mockup's label; render "Any distance" for `null`.
- **"Today" is the doctor's own calendar date** in their city's zone (backend Phase 13), not the server's UTC date, so the schedule no longer empties at 4-5 pm in Los Angeles. `next_visit` and every `todays_schedule` entry carry `timezone`/`scheduled_at_local` (§7).
- `undocumented_visits_count` = accepted/in-progress visits whose time has passed **with no visit outcome yet** — the "you have visits to document" nudge.
- `compliance_pending` types are `business_agreement`, `nondiscrimination_notice`, `dental_license`, `liability_insurance`; `action` is `sign` (never signed/uploaded) or `renew` (expired). It is empty when everything is in order. Titles/notes for the banner come from client copy keyed on `type` — the API returns no display strings. This replaces the mock's `complianceSignaturePreview`.
- `greetingDate`, `dentistName`, `summaryLine` are client-composed from `profile` and the counts (there is no server-side summary text).
- `office_hours` is a display span across all active windows regardless of day (`null` if none) — the mock's `availabilityHours`.

**Verified**: _(fill in once wired)_

---

## 11. Messages inbox

The mockup's Messages tab is one list of four kinds (`InboxMessageType`: `chat`, `emergency`, `reminder`, `payout`) — a **client-side merge of two feeds**:

| Kind | Source |
|---|---|
| `chat` | `GET /conversations` (shared) + `GET/POST /conversations/{id}/messages` — same contract as `docs/plans/PATIENT_APP_FRONTEND_PLAN.md` §10 (polling with `?since=`, text and/or attachment, viewing marks read, `403` if not a participant) |
| `emergency` | `GET /notifications` items with `data.type = new_nearby_open_request` |
| `reminder` | `GET /notifications` items with `data.type = appointment_reminder` (also `appointment_status_changed`, `new_message`, `document_expiring` — render generically off `data.type` + `data.title`, fall back gracefully for unknown types) |
| `payout` | **Nothing** — no payout/earnings system exists (Phase 11, deferred). Drop this type from the UI until it does |

- A conversation exists only once an appointment links this doctor and a patient (created on accept/claim). There is no way to message an arbitrary patient.
- **Conversation previews**: `ConversationResource` carries no last-message body, only `last_message_at` — same limitation as the patient app; show the patient name and a relative time.
- Notifications: `PATCH /notifications/{id}/read`, `PATCH /notifications/read-all` (shared). The mockup's "patientId" on a message maps to the conversation's patient profile.

**Verified**: _(fill in once wired)_

---

## 12. Push notifications

Shared with the patient app: `POST /device-tokens` `{ token, platform: ios|android }` after login and on token refresh; `DELETE /device-tokens` `{ token }` on logout (or pass `device_token` to `POST /logout`). See `docs/plans/PATIENT_APP_FRONTEND_PLAN.md` §11.2.

Doctor-relevant pushes: `new_nearby_open_request` (a new open-pool request inside this doctor's `service_radius_km`, or 50 km if unset), `appointment_status_changed` (patient cancelled/rescheduled), `appointment_reminder`, `new_message`, `document_expiring` (a credential nearing expiry). Delivery uses FCM (`FCM_SERVER_KEY`, configured server-side); the in-app notification feed is the source of truth either way.

---

## 13. `GET /enums`

One call returns every enum's `{ value: label }` map in the request locale — already fetched by `EnumsRepository` in this app. Doctor-relevant groups: `appointment_status`, `visit_type`, `appointment_request_mode`, `treatment_plan_category`, `treatment_plan_status`, `treatment_plan_item_status`, `restorative_type`, `treatment_category_group`, `legal_document_type`, `legal_document_status`, `document_template_type`, `otp_channel`, `user_status`. Use these labels instead of hardcoding English; keep the raw `value` for logic.

---

## 14. Mock data → real API field mapping (quick reference)

| Mock (`features/common/...`, `MockDentistData`) | Real source |
|---|---|
| `Visit` (`id, patientId, patientName, service, durationMins, status, address, distanceMiles, whenText, time, cta`) | `AppointmentResource` (§7). `patientId` → `patient_profile.id`; `patientName` → its names; `durationMins` → constant 60; `distanceMiles` → **computed client-side**; `status`/`cta` → derived from `status` + `visit_type` + `scheduled_at` |
| `Patient` (`id, name, age, languages, insurance*, dob, address, phone, phoneVerified, allergies, medications`) | `PatientProfileResource` + `insurance[]` + `medical_history` (§9); `photoUrl` has no source |
| `VisitHistoryEntry` | `appointments[]` joined with `clinical_notes[]` on `appointment_id` (§9) |
| `ClinicalNote` (`chiefComplaint, clinicalFinding, recommendation, isComplete`) | `ClinicalNoteResource` (§8); `isComplete` derived |
| `ToothChartEntry` (`id "UR6", label, flag`) | `ToothChartEntryResource` (`tooth_number` 1–32, `surfaces[]`, `condition_code`, `notes`) — needs a universal ↔ FDI lookup (§8) |
| `TreatmentPlanItem` (`title, note, category, added`) | `TreatmentPlanItemResource` (`treatment` \| `custom_title`, `notes`, `restorative_type`, `status`) under a `TreatmentPlanResource.category` (§8) |
| `HomeDashboard` | `GET /doctor/dashboard` (§10) |
| `RequestBuckets` (open/accepted/pending) | `requests/open`, `requests/accepted`, `requests/pending` (§7) |
| `ScheduleBundle` (`week`, `atHomeEnabled`, `emergencyEnabled`, `timeSlots`, `closedSlots`, `baseAddress`, `radiusMiles`) | availability windows + exceptions (§6) + practice address & `service_radius_km` (§3) |
| `ExpertiseBundle` (`preventative`, `cosmetic`, `selected`, `addedSpecialty*`) | `GET /treatments` grouped by `category.group` + `/doctor/services` (§5); the "added specialty" = the `is_custom` entry |
| `ComplianceDocument` (`status: Sign\|Renew\|On file`) | merge of agreements, non-discrimination, licenses, liability insurance (§4) — status derived |
| `DentistProfileBundle` (`about`, `travel`, `payments`, `license`, `education`, `alertText`, `menu`) | `DoctorProfileResource` + documents; `alertText` from `compliance_pending`; **`payments` has no source** |
| `InboxMessage` (`chat/emergency/reminder/payout`) | conversations + notifications merge (§11); `payout` has no source |
| Hardcoded `418207` OTP | Gone — real code from email/SMS (patient plan §2) |

---

## 15. Phased build plan

The app's structure (per-feature `domain/repositories` interface + `injector.dart`) means each phase is: add a `<Feature>RemoteDataSource`, implement the repository, swap the injector, delete that feature's slice of `MockDentistData`.

### Phase 0 — API foundation
- [x] `ApiClient` (Dio), env-configured base URL (`ApiConfig`), secure token storage, `Result<T>`/`Failure`, `get_it` DI — copied from the patient app
- [x] Bearer + `Accept-Language` on every request; central envelope handling
- [x] Confirm global `401` → clear token → login and `429` handling behave as in the patient app (inherited, but verify against this app's `AuthViewModel`)
- [x] **A pagination helper** (`core/network/paginated.dart`: `Paginated<T>`, `ApiClient.getPage`/`getAll`) reading `meta`/`links` — every list screen below needs it (and `per_page=100` for the schedule/expertise reads, §1)
- [ ] `EnumsRepository` consumers: swap any hardcoded labels for `GET /enums` (§13)
- [x] A shared **km ↔ miles** helper and a **Haversine** helper (visit distance, §7), and the **universal ↔ FDI tooth lookup** (§8) — pure utilities with unit tests
- [ ] Replace each mock entity with a `fromJson`-backed model incrementally, per phase (do not big-bang it)

**Verified (2026-09-19)**: `401` → `ApiClient.onUnauthorized` → `AuthViewModel.handleUnauthorized` (clears token, returns to login with a toast); `429` → `RateLimitFailure`. Helpers live in `lib/core/network/paginated.dart`, `lib/core/utils/geo.dart`, `lib/core/utils/tooth_numbering.dart` (universal 1–32 ↔ `UR8…LR8`, round-trip tested in `test/core/utils_test.dart`; 8/8 pass, analyzer clean). The `EnumsRepository` label swap and mock-entity replacement are per-feature and happen inside Phases 1–10. Note: `geolocator` is **not** actually a dependency of this app yet — add it in Phase 2 (practice-address pinning).

### Phase 1 — Authentication & onboarding
- [x] Register / verify email / verify phone / login / forgot-reset / logout / keep-signed-in (already real)
- [x] Post-register onboarding writes: `PATCH /doctor/profile` (names, mobile, registration no., `city_id`) and `POST /doctor/specializations`, replacing `saveDentistApplicationExtras` (§2)
- [x] Collect `date_of_birth` (missing from Register but required for profile completion, §2)
- [x] Surface a duplicate `health_authority_registration_no` (`422`) clearly on the Register/Profile screen
- [x] "City not listed" → `POST /doctor/city-requests`
- [ ] Route on `profile_completed_at`/`missing_fields` to a "finish your profile" state if the product wants one (§16)

**Built (2026-09-20, not yet verified against a live backend)**: register no longer collects licence/city/specialties. `GET /cities` and `GET /specializations` need the bearer token, so they cannot populate a pre-register form. A new `AuthPhase.onboarding` step (`DoctorOnboardingScreen`) follows register and precedes email verification. It collects registration no., date of birth, city (or "my city isn't listed" -> `city-requests`) and specialties from the API, then sends one `PATCH /doctor/profile` (names + mobile held from Register + these) and `POST /doctor/specializations`. The step has a "Skip for now" that leaves the profile incomplete until Phase 2. Duplicate registration numbers surface via `Failure.displayMessage` on that screen. **Not done**: an app restart mid-onboarding restores straight into the app with an incomplete profile (no `missing_fields` routing yet).

### Phase 2 — Profile & practice
- [x] Profile screen → `GET /doctor/profile`; edit → `PATCH /doctor/profile` (§3)
- [x] Practice address → `PUT /doctor/address`, **pinned with device coordinates** (no geocoder, §3)
- [~] Service radius control (Schedule + Home) → `service_radius_km`, km ↔ miles at the edge — built on the Travel & coverage screen; the Schedule/Home entry points land with Phases 5 and 9
- [x] Specialties editor → `POST /doctor/specializations` (replace-all semantics)
- [x] Assemble `about`/`travel`/`license`/`education` rows client-side; leave `payments` as an honest "coming soon" (§16)

**Built (2026-09-20, not yet verified against a live backend)**: `GET /doctor/profile` feeds the profile screen through `DentistProfileRepositoryImpl` + `buildProfileBundle` (unit-tested, `test/features/profile_bundle_mapper_test.dart`). New screens: **Edit profile** (name, mobile, years, languages, bio, specialties) and **Travel and coverage** (address, `Pin my current location` via `geolocator`, radius slider in miles -> whole km). An incomplete profile shows a banner from `missing_fields`. Payments shows "Coming soon"; the payout-alerts switch and `payment` menu row are removed. Mock profile repository and its `MockDentistData` rows deleted. `geolocator` was added to `pubspec.yaml` with Android/iOS permissions. **Gaps**: the compliance banner (`compliance_pending`) is Phase 3/9, so the old "agreement/licence outstanding" alert is gone for now. `education`/`skills`/`honors_awards`/`social_link`/`city_id` are read but not editable. Logging out resets the profile view model, but a `401` sign-out does not, so a second account signing in on the same session could briefly see stale profile data until the next load (fix in Phase 11 by resetting per-account view models when `AuthPhase` leaves `app`).

### Phase 3 — Documents & compliance
- [x] Merge the four document endpoints into the Documents list with derived `Sign`/`Renew`/`On file` (§4)
- [x] Sign agreements → `POST /doctor/agreements`, `/doctor/nondiscrimination`
- [x] Upload license / liability insurance (multipart, ≤ 5 MB pdf/jpg/png), show `rejection_reason` on a rejected row
- [x] Never cache `file_url` (signed, ~30 min) — the app never reads it

**Built (2026-09-20, not yet verified against a live backend)**: `DocumentsRepositoryImpl` merges the four paginated endpoints (`getAll`); `buildAgreement`/`buildCredential` derive `Sign`/`Renew`/`On file` and are unit-tested (`test/features/compliance_mapper_test.dart`, 12 cases). Rules used: an agreement is `sign` when its active template is unsigned; a credential is `sign` when never uploaded, or its newest row is `rejected` (the reason is shown), and `renew` when `metadata.expiration_date` is past **or within 30 days** (30 days is my choice; the plan said "or close"). Credentials with no submission show the badge `Upload` rather than `Sign`. New screens: **Sign agreement** (template body, typed name, agree box) and **Upload credential** (`file_picker`, 5 MB check, expiry date picker, kind-specific fields). Mock repository and mock rows deleted; `file_picker` added to `pubspec.yaml`. **Not done**: viewing an uploaded file (needs `url_launcher` plus a fresh fetch for the signed URL), and the Home compliance banner from `compliance_pending` (Phase 9). The old mock `hygiene-attestation` row has no backend and is gone.

### Phase 4 — Expertise
- [x] Two-tab catalog from `GET /treatments?per_page=100` grouped by `category.group`; selected set from `/doctor/services?per_page=100` (§5)
- [~] Toggle a service → `POST`/`DELETE /doctor/services`; set `expertise_level` — toggling done (batched behind Save); `expertise_level` is only settable on custom specialties, not on catalog entries
- [x] The "added specialty" → a `is_custom: true` entry (create/edit/delete)

**Built (2026-09-20, not yet verified against a live backend)**: `ExpertiseRepositoryImpl` reads every page of `/treatments` and `/doctor/services`. Catalog ticks are held locally and written by **Save expertise** via `planExpertiseChanges` (unit-tested, `test/features/expertise_test.dart`): new ids -> `POST`, unticked -> `DELETE`, and a treatment the doctor already has an *inactive* row for -> `PATCH is_active:true` (a `POST` would be a duplicate `422`). Custom specialties are saved immediately from a new **Add/Edit specialty** screen (any number, 1–5 star level optional, removable). A treatment category group other than `preventative`/`cosmetic` would appear under an "Other" section. Mock repository and its data deleted. `MockDentistData.defaultSelectedServices` is kept only for the still-mock Home tile (Phase 9).

### Phase 5 — Schedule
- [x] Load windows (`per_page=100`) → expand into the hour grid per `(visit_type, day)` (§6)
- [x] Save by diffing and coalescing hours into contiguous windows (`POST`/`PATCH`/`DELETE availability`)
- [x] Day on/off and closed hours; remember `day_of_week` 0 = Sunday
- [x] At-home / Emergency toggles (built as delete/re-add through the same diff, not `is_active` PATCHes — N calls, §6)
- [x] One-off closures / extra hours → `POST /doctor/availability/exceptions` (write-only; no list, §16)
- [ ] Confirm a saved schedule is bookable by fetching `GET /patient/availability` from a patient test account

**Built (2026-09-20, not yet verified against a live backend)**: `ScheduleRepositoryImpl` + a pure grid layer (`domain/entities/availability.dart`, unit-tested in `test/features/schedule_test.dart`, 13 cases). The screen edits **recurring weekly hours** per visit type: a Monday-first weekday strip, an at-home / emergency selector (tap a row to choose which type the grid edits; the switch turns the whole type on/off), and an hour grid 07:00–20:00. Save writes only the (visit type, weekday) groups that changed, reusing existing rows (`PATCH` re-time + `is_active:true`), adding (`POST`) or removing (`DELETE`, run first) as needed. Windows the grid can't represent (off-hour boundaries) are preserved unless that weekday is edited, in which case they are replaced. Turning a type **on** from empty seeds Mon–Fri 09:00–17:00 to adjust. New **One-off change** screen posts an exception (close a date / add hours; write-only, nothing lists them). The base address and radius card now read the profile (Phase 2) and link to Travel and coverage. **Deviation from the mock**: the mock's strip showed specific dates; the API is weekly-recurring, so the strip is weekdays (the number is just the next occurrence's day-of-month) and the title reads "Every Monday". Mock repository and mock schedule data deleted; logging out also resets the schedule view model.

### Phase 6 — Requests
- [x] Three buckets → `requests/open` (`scope`), `requests/pending`, `requests/accepted` (§7)
- [x] Client-side distance (Haversine from practice address) and status/`cta` derivation
- [x] Accept (handle `409` already-claimed as a normal refresh) / Decline (direct only, optional reason)
- [x] Start visit → `POST /doctor/appointments/{id}/start`
- [x] Reschedule / Cancel with required reason → shared `PATCH /appointments/{id}/reschedule|cancel`
- [x] Emergency cards show `tooth_number` + `problem_description`

**Built (2026-09-20, not yet verified against a live backend)**: `RequestsRepositoryImpl` reads the three lists (every page; Open takes `scope`, chosen by a Nearby/All chip) and `visitFromAppointmentJson` (unit-tested, `test/features/visit_mapper_test.dart`, 9 cases) maps each `AppointmentResource` to the shared `Visit`: status by list + `visit_type` + whether it's scheduled today, joined treatment names, 60-min duration, `ASAP` for a null `scheduled_at`, distance from the practice pin (hidden when either end has no coordinates). `Visit` gained `apiStatus`, `isDirect`, `isEmergency`, `toothNumber`, `problemDescription`, `scheduledAt`, and `distanceMiles` is now nullable. Actions: Accept (open + pending), Decline with optional reason (pending only), Start visit, Reschedule (date + time + required reason) and Cancel (required reason) on accepted visits. Every action refreshes all three lists afterwards, which is also how a lost open-pool race (`409`) resolves: the server's message is toasted and the card disappears. Pull-to-refresh is the only freshness mechanism. **Gaps**: the `GET /appointments/{id}` detail screen isn't built (the card already carries everything the list returns); an in-progress visit's button opens the patient chart until the visit workflow (Phase 8) exists; **See profile** on an accepted card opens the still-mock patient detail (Phase 7). Open and pending cards deliberately have no See profile, because the chart is `403` until a visit is accepted. Mock repository deleted (the `MockDentistData` request rows remain only for the still-mock Home tab); logging out resets the requests view model.

### Phase 7 — Patients & chart
- [x] Patients tab → `GET /doctor/patients?q=` with pagination and search (§9)
- [x] Patient detail bundle → `GET /doctor/patients/{id}` mapped per §9 (insurance, allergies/medications as text, history ⨝ clinical notes, teeth, restorative/cosmetic plans)
- [x] Private note → `POST /doctor/patients/{id}/notes`
- [x] Handle `403` (no treating relationship) as "not your patient" — shown as the server's message with Try again

**Built (2026-09-20, not yet verified against a live backend)**: **Patients tab**: `PatientsRepositoryImpl` pages `GET /doctor/patients` (20 per page, a Load more button), server-side name search (`q`, 350 ms debounce), a total tile from `meta.total`; the mock's per-patient tags/summary/when have no source and are gone. **Chart** (`patientDetailFromJson`, unit-tested in `test/features/patient_mapper_test.dart`, 7 cases): patient + primary insurance + free-text allergies/medications split into chips (empty insurance shows "None on file"), visit history joined to clinical findings by `appointment_id`, latest clinical note, all 32 teeth in chart order (universal 1–16 across the top, 32–17 across the bottom) flagged from the newest plan's tooth chart via `condition_code` (`decay`/`watch`/`restored`; anything else unflagged), and plan items by category. **Private notes** are listed and added from a bottom sheet. **The chart is read-only**: the mock's local-only editing of clinical notes, plan items and the tooth flags is removed rather than faked (Phase 8 brings the writes), as are the mock "Reschedule / Mark completed" bar and the sketch/quick-add chips. **Gaps**: prescriptions are returned by the chart but not shown yet (Phase 8); patients have no photo (initials only); the visit history's "provider" slot now shows the visit status. The mock patients repositories and all their `MockDentistData` rows are deleted, and logging out resets the patients view model.

### Phase 8 — Visit workflow (clinical documentation)
- [x] Clinical note form → `POST .../clinical-notes` (§8)
- [x] Treatment plan builder → `POST .../treatment-plan` (one per category), finalize → `PATCH /doctor/treatment-plans/{id}`
- [~] Tooth chart → bulk `POST` (built) and single-tooth `PUT` (not used), through the universal ↔ FDI lookup; `condition_code` values agreed and recorded below
- [x] Prescription → `POST .../prescriptions`
- [x] Recommendations → `POST .../recommendations` (rank 1–3)
- [~] **Complete visit** = submit `POST .../visit-outcome` (this closes the visit, §8) — built; "verify the visit leaves the accepted list" needs a live run

**Built (2026-09-20, not yet verified against a live backend)**: a new `visit_workflow` feature. **Start visit** on an accepted card now opens the **visit workflow** screen (and **Continue visit** re-opens it for an in-progress visit). Five independent collapsible sections, any order: **Clinical note** (complaint / finding / recommendation / additional notes), **Treatment plan and tooth chart** (Restorative | Additional services tabs, services from the searchable catalog or custom, a `restorative_type` prompt for restorative items, Save plan then Finalize; a tap-to-cycle tooth chart saved against the showing category's plan), **Prescriptions** (dosage required; several allowed), **Recommended next** (rank 1–3 assigned in order, optional reason) and **Complete visit** (one-per-line lists prefilled from the visit's service and this session's prescriptions, plus a summary). Submitting the outcome completes the visit, refreshes Requests and closes the screen. Request bodies are pure functions unit-tested in `test/features/visit_payloads_test.dart` (8 cases).

**`condition_code` vocabulary (agreed)**: `decay`, `watch`, `restored`. An unflagged tooth (`none`) is simply not written (the API can't delete an entry). The chart's `surfaces` is required and non-empty but the mockup flags whole teeth, so every entry is written with `surfaces: ["whole"]`; the chart reader (Phase 7) ignores surfaces.

**Gaps / known limits**: what counts as "saved" is tracked for this screen session only — the API has no read for one appointment's documentation, so re-opening the screen shows every section fresh. A plan's items can't be edited after saving (no endpoint to change them; saving the same category twice is not guarded and the server's response would surface as an error). Recommendation ranks come from a per-session count, so re-opening the screen restarts at #1 and a rank clash or the cap of three would come back as a `409` message. The single-tooth `PUT` isn't used. The Home tab's mock "Open visit" still goes to the patient chart until Phase 9.

### Phase 9 — Home dashboard
- [x] One call, `GET /doctor/dashboard` (§10) → greeting/summary (client-composed), next visit, today's schedule, open/pending counts, undocumented-visit nudge, compliance banner from `compliance_pending`
- [x] Radius label from `open_requests_radius_km` (`null` → "Any distance")

**Built (2026-09-21, not yet verified against a live backend)**: `HomeRepositoryImpl` makes the single `GET /doctor/dashboard` call and `dashboardFromJson` (unit-tested, `test/features/home_mapper_test.dart`, 9 cases) shapes it. Composed client-side: the greeting date, the doctor's name (from `profile`), and the summary line ("2 visits today, 3 open requests near you, 1 request awaiting you."). Next visit and today's schedule reuse the requests mapper (distance from the practice pin inside `profile.address`). The radius reads "N within X miles" (km -> miles) or "N at any distance" when `null`; office hours show "Not set" when `null`. The compliance section now comes from `compliance_pending` (type + action -> client copy) and **disappears when nothing is pending**; the Profile compliance banner is *not* wired to it yet. The **Clinical notes** tile shows "N visits to document" and opens Requests > Accepted; tapping a today's-schedule row (or Open visit) opens the visit workflow for an in-progress visit and the patient chart otherwise. Pull-to-refresh, an error/retry state and a "No visits today" empty state added; the hero avatar initials come from the real name. The Home mock repository and every remaining `MockDentistData` request/schedule/compliance row are deleted (only the Messages inbox mock remains, until Phase 10). Logging out resets the Home view model. **Known limit**: Home is not refreshed when you accept, start or complete a visit elsewhere; pull down to refresh.

### Phase 10 — Messages & notifications
- [x] Merge `GET /conversations` and `GET /notifications` into the inbox (§11); drop the `payout` type
- [x] Thread view with polling (`?since=`), send text/attachment (patient plan §10)
- [x] Mark read (single + all)
- [~] Register/deregister the FCM token (`/device-tokens`) on login/logout (§12) — the plumbing is built; **there is no FCM SDK yet, so no token is ever produced**

**Built (2026-09-21, not yet verified against a live backend)**: **Inbox** (`MessagesRepositoryImpl`, mapper unit-tested in `test/features/inbox_mapper_test.dart`, 9 cases): conversations (all pages) merged with the 30 most recent notifications, newest first. Chat rows show the patient name and an unread badge (there is no last-message text on the wire, so the body is "N new messages" or "Conversation"); `new_nearby_open_request` notifications are emergency rows, `appointment_reminder`/`document_expiring` are reminders, everything else renders generically off `data.title`; `new_message` notifications are left out because chat rows already cover them. The `payout` type is gone. Tapping marks a notification read (optimistic) and jumps to Requests (Open for an emergency, Accepted otherwise); **Mark all read** is in the nav bar. **Chat** (`ChatViewModel`): loads the thread, polls `?since=<last message time>` every 5 s while open and merges by id, sends text and/or one pdf/jpg/png attachment (10 MB check), pull-to-refresh on the inbox. **Push** (`core/push/push_service.dart`): `PushRegistrar` posts `POST /device-tokens {token, platform}` when the auth phase becomes `app` and on every token refresh, and the logout call passes the device token so `POST /logout` deregisters it. The token comes from a `PushTokenSource`, currently a no-op, because delivery is FCM and that needs `firebase_messaging` plus per-platform Firebase config (`google-services.json`, `GoogleService-Info.plist`) that this project doesn't have. **To finish push**: add those, then register a `PushTokenSource` backed by `FirebaseMessaging` in `core/di/injector.dart`. **Gaps**: attachments show as "Attachment" text (opening one needs `url_launcher`); the Home bell dot isn't refreshed after reading notifications; polling stops when the thread closes (no background sync). The mock messages repository and the last `MockDentistData` file are deleted.

### Timezone contract — backend decision #7 (2026-10-17)
Backend Phase 13 is built on TEETHi-Backend branch `feat/timezone-contract` (`25a38b4`, `9253d74`) and merged to `master`. The contract is in §6 (hours, open slots) and §7 (visit times, reschedule). Fixes consistency report C1.
- [x] Show visit times from `scheduled_at_local` instead of `toLocal()`, in requests, schedule, home and patient detail (`features/requests/data/mappers/visit_mapper.dart`, `core/utils/date_format.dart`). Compute "is today" from `scheduled_at_local`'s date, not from the device clock. **Verified (2026-10-18, TEETHI-Dentist-App `1487519` on branch `feat/timezone-contract`, unit tests only):** `visitFromAppointmentJson` shows `scheduled_at_local` and computes "today" as now plus the zone offset read off `scheduled_at`/`scheduled_at_local`, so requests, Home's next visit and today's schedule, and the visit workflow header all use the city date. Patient-detail history dates use the same offset (`historyDate`). The schedule tab shows no visit times, so nothing changed there. A payload without `scheduled_at_local` falls back to the device zone. Home's greeting date still uses the device date, because the app has no tz database.
- [x] Reschedule: replace the free date/time picker (`pickFutureDateTime` in `request_dialogs.dart`) with the open-slot list from `GET /doctor/availability/open-slots`, and send the chosen slot's `starts_at` as `scheduled_at`. The backend accepts only an exact open slot start, so today's free picker gets `422 slot_not_available` unless the time happens to match a slot. **Verified (`1487519`, unit tests only):** `pickOpenSlot` shows a date (starting from the doctor's today in the visit's zone) and that date's slots from `open-slots` (`visit_type` from the visit), then asks for the reason. `reschedulePayload` sends the slot's `starts_at` string unchanged. `pickFutureDateTime` is gone.
- [x] Schedule editor: say that hours are in the city's timezone, showing `city.timezone`, so a doctor travelling with their phone doesn't misread their hours. **Verified (`1487519`):** `DoctorProfile.cityTimezone` reads `city.timezone`. The weekly grid and the date-exception screen both say "Hours are in Los Angeles time (America/Los_Angeles), your city's timezone…", using the platform default when there's no city.

### Reschedule confirmation — backend decision #8 (2026-10-18)
- [x] Pending lists a patient's reschedule (`status: rescheduled`) with a "New time" badge, the patient's reason and **Accept new time**. Declining cancels the visit and requires a reason. A doctor's own reschedule comes back `accepted`. In-progress visits are labelled "In progress" instead of "Today" (E8).

**Verified live (2026-10-18):** TEETHi-Dentist-App `c59e68a` on `feat/reschedule-confirmation`.
1. The patient moved visit #2 to 12:00. The dentist's Pending tab showed it as "New time" with the reason, while the patient saw "New time · awaiting dentist" with only Cancel.
2. Accept new time returned it to Accepted.
3. The dentist's own reschedule to 15:00 stayed Accepted. After a refresh, the patient saw "Confirmed".

### Patient display name — backend decision #10 (2026-10-18)
- [x] Requests, Patients and Messages show `full_name`, then first + last name, then "Patient", through one `patientDisplayName()` helper. (TEETHI-Dentist-App `b14a731`, branch `feat/decisions-9-11`; verified live: "Sample Patient" instead of "Patient")

### Phase 11 — Polish & hardening
- [ ] Empty/error/loading states on every list; pull-to-refresh as the only freshness mechanism
- [ ] Contract test against a live backend for each repository (the mocks currently make every screen look fine)
- [ ] End-to-end run: register → onboard → sign docs → set schedule + radius → receive a patient's booking → accept → start → document → complete
- [ ] Remove `MockDentistData` and every `Mock*Repository` once nothing references them

---

## 16. Open items to raise with the backend / product

Backend candidates (none block the phases above):

1. **No `GET` for availability exceptions** — the app can write a one-off closure but never list or delete one (§6). A `GET /doctor/availability/exceptions` (and `DELETE`) would complete the Schedule screen.
2. **No bulk enable/disable of a visit type's windows** (§6) — "turn off Emergency" is N `PATCH` calls today.
3. **No way to delete a tooth-chart entry** (§8) — "clear this tooth" can't be expressed; needs either a `DELETE` or an agreed "no finding" `condition_code`.
4. **No distance on `AppointmentResource`** (§7) — the open list is *sorted and filtered* by distance server-side but the value isn't returned, so the client recomputes it. A `distance_km` on the open-request items would remove the duplication.
5. **No patient photo for the doctor** (§9) — chart shows initials only unless a doctor-safe avatar URL is added.

Product decisions:

6. **Payouts / earnings** — the `payments` profile block and `payout` inbox type have no backend at all (Phase 11, deferred). Hide them or label them "coming soon".
7. **Doctor approval gate** — a registered doctor is `active` immediately and can receive requests without verified credentials. Should an admin verify a license before a doctor is bookable/visible in the directory?
8. **Profile-completion gating** — should an incomplete profile (`missing_fields`) block the Requests/Schedule tabs?

If a new gap turns up while building against this plan, add it here with a §-reference to where it affects the plan above.

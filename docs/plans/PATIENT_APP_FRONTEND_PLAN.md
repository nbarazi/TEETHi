# Teethi Patient App — Flutter Frontend Wiring Plan

This document tells a Flutter engineer/agent **what exists in the Teethi backend today, exactly how to call it, and in what order to wire the existing patient-app UI to it.** It intentionally contains no Dart/Flutter code or state-management guidance — that's an implementation detail left to whoever builds it. What follows is the API contract, the business rules behind it, a field-by-field mapping from the app's current mock data to the real API, and a phase-tracked scope.

**Starting point, precisely**: `D:\Projects\TEETHi-Patient-App` today has **zero backend integration of any kind** — no HTTP client in `pubspec.yaml`, no token storage, no models with `fromJson`/`toJson`. Every screen reads from one static file (`lib/data/mock_data.dart`) via an in-memory `ChangeNotifier` (`lib/state/app_state.dart`); both files say outright that they're stand-ins for a future backend. This is not a "finish the last 20%" job — it's building the entire networking layer from scratch and then swapping every mock read/write for a real call, screen by screen.

**Source of truth for the backend itself**: `docs/plans/IMPLEMENTATION_PLAN.md`, next to this file in the root docs repo. Phases 0–9 (core domain) and Phase 12 (patient directory/reviews/self-service records/SMS/push — the work this plan assumes exists) are complete. Phase 10 (HIPAA hardening close-out: "log out all devices," scrubbed exception envelopes) and Phase 11 (real payment gateway — Stripe/Apple Pay actually charging a card) are **not built yet**; the payment endpoints below exist and work end-to-end, but every card/Apple Pay charge is currently a fake gateway that always succeeds. Do not build UI that assumes a real charge failure path exists yet.

**How to use this file**: work phase by phase, top to bottom. Check items off (`- [x]`) as they land, and add a short "Verified" note under each phase, the same way `docs/plans/IMPLEMENTATION_PLAN.md` and `docs/plans/ADMIN_DASHBOARD_FRONTEND_PLAN.md` track their own progress. If a backend detail below turns out to be wrong or has changed, fix this doc in the root docs repo as part of the same change that discovers it.

---

## 1. API fundamentals

- **Base path**: `http://localhost/api/v1/...` in local dev. Patient routes live under `/api/v1/patient/...`; a handful of routes the patient app shares with the doctor/admin apps live at the top level (`/api/v1/...`) — called out explicitly below.
- **Auth transport**: Laravel Sanctum **personal access tokens**, not cookies/sessions. Every authenticated request sends `Authorization: Bearer <token>`. Tokens **do not expire** on their own — a token is valid until the user logs out (`POST /logout`, deletes just that token) or changes/resets their password (deletes **all** of that user's tokens everywhere).
- **A token is issued immediately on `POST /register`, before any verification.** The app can move straight to an authenticated state after registering; email/phone verification happen as separate, later, already-authenticated steps (§2). This matches the app's own existing screen order (register → verify email → verify phone) — nothing about that flow needs to change, only what each step actually calls.
- **Locale**: send `Accept-Language: en` or `Accept-Language: ar` on every request to get `message` strings and enum labels in that language. Fetch enum display labels from `GET /enums` (§12) rather than hardcoding English/Arabic text.
- **Rate limits**: `POST /register` / `POST /login` — 5/min per IP. Every other authenticated route the patient app uses — 120/min per user. OTP send/verify endpoints (`email/verify`, `email/resend`, `phone` (PUT), `phone/verify`, `phone/resend`) — 6/min per user. A `429` is returned past these; the OTP resend countdown the app's UI already has is a reasonable client-side pacing on top of this.
- **Response envelopes** — four distinct shapes, exactly as documented in `docs/plans/ADMIN_DASHBOARD_FRONTEND_PLAN.md` §1.1 (same backend, same behavior for every app):
  - **Success**: `{ "success": true, "message": "...", "data": {...} | null }`
  - **Paginated success** (every list endpoint): adds `meta` (`current_page`, `per_page`, `last_page`, `total`, ...) and `links` (`first`/`last`/`prev`/`next`) alongside `data` as the array of items. Every list endpoint accepts `?per_page=N` (default 15, max 100) and `?page=N`.
  - **Hand-written app error** (403/404/409, some 400/422s): `{ "success": false, "message": "..." }`, sometimes with an `errors` object (e.g. booking's missing-requirements gate, §7).
  - **Laravel's default shapes (NOT the custom envelope)** — a client must handle these separately: **422** validation failure (`{ "message": "The given data was invalid.", "errors": { "field": ["..."] } }`), **401** unauthenticated (`{ "message": "Unauthenticated." }`). A **403** from the `role:patient` middleware (wrong app/role) or `active` middleware (deactivated account) DOES use the custom envelope, since those are app-level checks, not the raw guard failing.
  - **204 No Content** for every `DELETE` and a couple of other endpoints — don't try to parse a body.

---

## 2. Authentication & session

| Endpoint | Purpose | Body | Auth |
|---|---|---|---|
| `POST /register` | Create account, auto-issues an email OTP, returns a token immediately | `{ name, email, password, password_confirmation, role: "patient" }` → `{ user: UserResource, token }` | No |
| `POST /login` | | `{ email, password }` → `{ user: UserResource, token }` | No |
| `POST /logout` | Deletes the current token only. Pass `device_token` (the FCM token registered in §11.2, if any) to also deregister this device from push in the same call | `{ device_token?: string }` | Yes |
| `GET /me` | Current `UserResource` | — | Yes |
| `POST /change-password` | Deletes **every** token for the user (forces re-login everywhere) | `{ current_password, new_password, new_password_confirmation }` | Yes |
| `POST /email/verify` | | `{ code }` (6-digit string) | Yes |
| `POST /email/resend` | | `{ email }` | Yes |
| `PUT /phone` | Sets/changes the phone number and issues a new OTP; clears any prior verification | `{ phone_number }` | Yes |
| `POST /phone/verify` | | `{ code }` | Yes |
| `POST /phone/resend` | Fails `400` if no phone on file, `400` if already verified | — | Yes |
| `POST /forgot-password` | Sends a 6-digit reset code by email (OTP-based, same as every other verification step — not a mail link) | `{ email }` | No |
| `POST /reset-password` | | `{ email, code, password, password_confirmation }` | No |

**`UserResource` shape** (from `/me`, `/login`, `/register`, and every endpoint that returns one — `PUT /phone`, `POST /phone/verify` does not return a body):
```json
{
  "id": 1, "name": "Amina K.", "email": "amina@example.com",
  "role": "patient", "status": "active", "profile_image": null,
  "phone_number": "+15551234567", "phone_verified_at": null,
  "email_verified_at": "2026-09-15T10:00:00+00:00",
  "created_at": "...", "updated_at": "..."
}
```

**Read before building the auth screens:**
- **There is no such thing as a real "paste code" shortcut, magic value, or dev bypass on the backend.** The app's current `VerifyScreen`/`ForgotPasswordScreen` hardcode `418207` as a fake OTP for the UI preview — this must be deleted entirely once wired; the real code comes from the email/SMS the backend actually sends (§4 in `docs/plans/IMPLEMENTATION_PLAN.md`'s Phase 12 covers real SMS delivery via Twilio; email always went through a real mailer).
- **For manual/QA testing before delivery is trustworthy in a given environment**, every endpoint in this table that issues a code (`register`, `email/resend`, `PUT /phone`, `phone/resend`, `forgot-password`) can echo it back as `data.otp_code` — a real 6-digit string, not a fake constant. This is off by default and controlled server-side (`OTP_EXPOSE_IN_RESPONSE`), never by anything the client sends, and is hard-disabled in production regardless of that setting — don't build any client-side logic around its presence/absence, it's a backend/QA switch only, not a feature to branch on.
- **Registration does not accept a mobile number.** `POST /register` is `{ name, email, password, role }` only — the app's `RegisterScreen` collects a mobile number on the same screen, but phone capture/verification is a separate step via `PUT /phone` (already authenticated, using the token `register` just returned). Split this in the wiring: register with name/email/password, then immediately call `PUT /phone` with the mobile number the user typed, before moving to the verify-phone step.
- **`name` is one field, not first/last/middle/suffix/title separately.** Concatenate the register screen's first/last name inputs into `name` for this one call; the *richer* structured name (title/first/middle/last/suffix) lives on `PatientProfile` (§3) and is set via a separate `PATCH /patient/profile` call, not at registration.
- **There is no "city" field on registration.** The app's `RegisterScreen` has a city dropdown; that value should be saved to the patient profile via `PATCH /patient/profile` (`city_id`) after registration, not sent to `/register` itself (which doesn't accept it).
- **"Keep me signed in"** has no server-side counterpart to call — it only governs whether the frontend persists the bearer token to secure storage across app restarts vs. keeping it in memory for the session. This is the very first piece of state persistence the app needs (§13 Phase 0) since there is currently *none*.
- ~~**Known gap**: `UserResource` doesn't expose `phone_number`/`phone_verified_at`~~ — resolved: both fields are now on `UserResource` (shown above), returned from `/me`, `/login`, `/register`, and `PUT /phone`. Use `phone_verified_at !== null` directly for the phone-verified check the booking gate needs (§7), no separate lookup required.
- ~~**Password reset was mail-link based, not code-based**~~ — resolved: `POST /forgot-password` now sends a 6-digit code by email and `POST /reset-password` takes `{ email, code, password, password_confirmation }`, the exact same shape/UX as every other verification step in this table. The app's `ForgotPasswordScreen` code-entry step already matches this; wire it directly, no redesign needed. The code is a distinct `password_reset` OTP channel from email verification, so requesting one never invalidates a pending email-verification code (or vice versa) for the same account.

**Verified**: _(fill in once wired — note the Flutter/http-client version used, and confirm the OTP round-trip works against a real inbox/phone once Twilio is configured, not just the dev "log channel" fallback)_

---

## 3. Patient profile & family (dependents)

| Endpoint | Purpose | Body |
|---|---|---|
| `GET /patient/profile` | The account holder's own care profile | — |
| `POST`/`PATCH /patient/profile` | Partial update (send only changed fields) | any of `title, first_name, middle_name, last_name, suffix, gender, blood_type, date_of_birth, profile_note, bio, languages[], city_id` |
| `PUT /patient/address` | Upsert the home address | `{ street, apartment?, note?, city, state, zip, latitude?, longitude? }` |
| `POST /patient/profile/photo` | Upload/replace the avatar (`multipart/form-data`) — replacing an existing one deletes the old file, never leaves two on disk | `{ photo: file }` (image, max 5MB) → `UserResource` |
| `DELETE /patient/profile/photo` | Remove the avatar | — → `204` |
| `GET /patient/dependents` | Paginated list of family members this account manages (excludes the account holder's own profile) | — |
| `POST /patient/dependents` | Add a dependent | `{ relationship_to_account_holder, title?, first_name, middle_name?, last_name, suffix?, gender?, blood_type?, date_of_birth?, profile_note?, bio?, languages[]?, city_id? }` |
| `GET/PATCH/DELETE /patient/dependents/{id}` | Single dependent CRUD | PATCH: same fields as store, all optional |

`PatientProfileResource`:
```json
{
  "id": 12, "is_account_holder": true, "relationship_to_account_holder": null,
  "title": "Mr.", "first_name": "Amina", "middle_name": null, "last_name": "K.", "suffix": null,
  "gender": "female", "blood_type": "o_positive",
  "date_of_birth": "1990-04-12", "profile_note": null, "bio": "Loves flossing.", "languages": ["en", "ar"],
  "city": { "id": 3, "name": "Dubai", "state": "Dubai", "timezone": "Asia/Dubai", "is_available": true },
  "address": { "id": 5, "type": "home", "street": "...", "apartment": null, "note": null, "city": "Dubai", "state": "Dubai", "zip": "00000", "latitude": 25.2, "longitude": 55.3 },
  "profile_completed_at": "2026-09-15T00:00:00+00:00",
  "missing_fields": [],
  "created_at": "...", "updated_at": "..."
}
```

- **`profile_completed_at`/`missing_fields`** are exactly what drives the app's "Before your first visit" home-screen checklist — `missing_fields` is a list of the required-field names still empty (from `PatientProfile::REQUIRED_FOR_COMPLETION`: `first_name`, `last_name`, `date_of_birth`, `city_id`); render "complete your profile" whenever it's non-empty, and hide it once `profile_completed_at` is set. `gender`/`blood_type`/`bio` are **not** part of profile completion — they're optional medical/personal metadata, not required-for-booking fields.
- **`gender`** (`male`/`female`/`other`) and **`blood_type`** (`a_positive`, `a_negative`, `b_positive`, `b_negative`, `ab_positive`, `ab_negative`, `o_positive`, `o_negative`) are both fixed enums — fetch their labels from `GET /enums` (§12: `gender`, `blood_type`) rather than hardcoding dropdown text.
- **The avatar is a real uploaded file now, not a color-tone stand-in.** `UserResource.profile_image` (§2) is a short-lived (~30 min) signed URL onto the private disk, regenerated on every fetch — same pattern as insurance cards/legal documents/message attachments elsewhere in this doc — never cache it, re-fetch `GET /me` or `GET /patient/dashboard` to get a fresh one. `null` until a photo is uploaded. There is no avatar endpoint for a dependent — a dependent has no `User` of its own (`user_id: null`) to hang a photo on, matching the mock's own `avatarTone` being an account-level-only concept.
- **A dependent has `user_id: null`** and never logs in — there's no separate account for a family member, only a profile row this account books/manages on their behalf. `relationship_to_account_holder` is a small fixed enum (`GET /enums` → `patient_relationship`) — use that for the dropdown, not hardcoded strings.
- **`{dependent}` in every dependent route is a `PatientProfile` id.** Deleting a dependent (`DELETE`) is a hard delete with no confirmation step server-side — confirm in the UI before calling.
- **Every booking/insurance/consent/medical-history/records endpoint that supports "on behalf of a dependent" does so via an optional `?patient_profile_id=`/`patient_profile_id` field** defaulting to the account holder's own profile when omitted (§7–§9) — the family-member switcher the app will need is one query param threaded through, not a separate set of endpoints per profile.

**Verified**: _(fill in once wired)_

---

## 4. Insurance

| Endpoint | Purpose | Body |
|---|---|---|
| `GET /patient/insurance` | Every insurance record across the account (self + dependents) | — |
| `POST /patient/insurance` | Add a policy — `multipart/form-data` if uploading card photos | `{ patient_profile_id?, provider_name, policy_number, group_number?, subscriber_name, subscriber_dob, relationship_to_subscriber, is_primary?, card_front?: file, card_back?: file }` |
| `GET/PATCH/DELETE /patient/insurance/{id}` | Single-record CRUD; PATCH accepts a partial subset + optionally re-uploading either card image | — |

`InsuranceDetailResource`:
```json
{
  "id": 4, "patient_profile_id": 12,
  "provider_name": "Delta Dental", "policy_number": "DD-99213", "group_number": null,
  "subscriber_name": "Amina K.", "subscriber_dob": "1990-04-12",
  "relationship_to_subscriber": "self",
  "card_front_url": "http://localhost/storage/...(signed, ~30 min TTL — never cache, refetch to view)",
  "card_back_url": null,
  "is_primary": true, "verified_at": null,
  "created_at": "...", "updated_at": "..."
}
```
- **Setting `is_primary: true` on one record automatically unsets it on every other insurance record for that same patient profile** — no separate "make primary" action needed.
- **`relationship_to_subscriber`** is a fixed enum (`GET /enums` → `insurance_relationship`), same dropdown-from-enum pattern as everywhere else.
- ~~**There is no insurance eligibility/coverage-check integration.**~~ — resolved (product decision): there is no third-party eligibility API wired up and none is planned for this pass, so the flat client-computed coverage estimate on the confirm/payment screens stays exactly as it is today (same client-side math, no backend call) — but every screen that shows it must visibly label it as a rough estimate, not a real quote (e.g. "Estimated coverage — not a quote from your insurer"), so it can't be mistaken for a real eligibility check. The "We check your cover before your visit" copy should be softened/reworded to match (it currently implies a real check that doesn't exist).
- **`verified_at`** exists on the model/resource but nothing currently sets it — no admin/backend workflow verifies an insurance record yet. Don't build UI around a "verified" badge changing state.

**Verified**: _(fill in once wired)_

---

## 5. Consent forms

| Endpoint | Purpose | Body |
|---|---|---|
| `GET /patient/consent-forms?patient_profile_id=` | Every currently-active patient-facing consent-form template, each annotated with whether the target profile (self, or the given dependent) has already signed *this exact version* | — |
| `POST /patient/consent-forms/{documentTemplate}/sign` | Sign one | `{ typed_name, agree: true, patient_profile_id? }` |

Each item (`DocumentTemplateResource`, in the signed-flow shape):
```json
{
  "id": 3, "type": "consent_form", "audience": "both", "title": "Patient Consent Form",
  "body": "Full legal text to render/scroll...", "version": 2, "is_active": true,
  "effective_at": "2026-09-11T00:00:00+00:00",
  "signed": true, "signed_at": "2026-09-15T09:00:00+00:00",
  "created_at": "..."
}
```
- **Signing is "type your full name + a checkbox," not a drawn signature** — matches the app's `SignFormScreen` exactly (typed name ≥ 2 chars + agree checkbox). `signed_at` on the response is the real signature timestamp — use it, don't fabricate one client-side.
- **A new consent-form version means everyone must re-sign.** If the backend publishes version 2 of a template a patient already signed version 1 of, `signed` comes back `false` again for the new version — this is intended, not a bug, and the booking gate (§7) checks against the *current* version specifically.
- **Booking is blocked until the current consent form is signed** (§7) — the app's booking flow's own consent-related step should call this endpoint (or check cached state from it) before letting the user proceed, rather than relying on the client-side "signedByDefault"/local `Set<String>` the mock currently uses, which is never persisted and resets every app restart.

**Verified**: _(fill in once wired)_

---

## 6. Doctor directory & reviews (backend Phase 12)

| Endpoint | Purpose |
|---|---|
| `GET /patient/providers` | Browse/search doctors. Query: `q` (name substring), `city_id`, `specialization_id`, `treatment_id`, `available_today=1`, or `latitude`+`longitude` (+ optional `radius_km`, default 50) for "near me" — switches to distance-sorted results |
| `GET /patient/providers/{doctor}` | One doctor's detail |
| `GET /patient/providers/{doctor}/reviews` | Paginated public reviews for that doctor |

`ProviderResource` (`{doctor}` here is a `DoctorProfile` id — **a real id exists now**, unlike the app's current mock `Doctor` model which has none and keys doctors by name):
```json
{
  "id": 9, "name": "Dr. Sara M.", "bio": "...", "years_experience": 12,
  "average_rating": 4.6, "patients_treated_count": 320, "recommended_percentage": 94,
  "starting_fee": 120.0, "hours": "9:00 AM - 5:00 PM", "available_today": true,
  "city": { "id": 3, "name": "Dubai", "state": "Dubai", "timezone": "Asia/Dubai", "is_available": true },
  "specializations": [ { "id": 2, "name": "Orthodontics" } ],
  "latitude": 25.2, "longitude": 55.3,
  "distance_km": 3.42
}
```
`ReviewResource` (public — never the full reviewer identity):
```json
{ "id": 55, "reviewer_name": "Amina K.", "doctor_rating": 5, "would_recommend": true, "comment": "Great visit.", "created_at": "..." }
```
- **`distance_km` is only present when the request included `latitude`/`longitude`.**
- **`starting_fee`/`hours`/`available_today` are all computed, best-effort signals**, not guaranteed bookable facts: `starting_fee` is the cheapest catalog treatment this doctor has marked active expertise in (`null` if none), `hours` spans every active weekly availability slot regardless of day, and `available_today` is a coarse "has *some* slot template or one-off exception today" check (today in the doctor's own zone since backend Phase 13, so it no longer flips to tomorrow at 4-5 pm in Los Angeles; `?available_today=1` filters the same way) — **not** the real per-slot availability the booking flow's calendar step needs. Always call `GET /patient/availability` (§7) for the actual bookable times once a specific doctor + date + visit type is chosen; don't try to derive real slots from the directory response.
- **No PHI is in this response** — it's safe to cache/log/show in analytics unlike almost everything else in this document.

**Verified**: _(fill in once wired)_

---

## 7. Booking & appointment management

### 7.1 Catalogs the booking flow needs first

| Endpoint | Purpose |
|---|---|
| `GET /cities` | City picker (only `is_available: true` cities — same shared endpoint as registration) |
| `GET /specializations` | `{ id, name }[]` |
| `GET /treatments?quick_select=1` | The handful of treatments flagged for the home screen's "quick book" tiles |
| `GET /treatments?treatment_category_id=` | Procedures under one category tab |
| `GET /patient/availability?doctor_profile_id=&visit_type=&date=` | Real open slots for one doctor/date/visit type, not paginated — shape below |

**Availability slots** (timezone contract, backend Phase 13, decision #7). `data` is still a plain list; each slot gained `starts_at`, `ends_at` and `timezone`:
```json
{
  "data": [
    {
      "start_time": "09:00:00", "end_time": "10:00:00",
      "starts_at": "2026-11-02T09:00:00-08:00", "ends_at": "2026-11-02T10:00:00-08:00",
      "timezone": "America/Los_Angeles"
    }
  ]
}
```
- **`date` is a calendar date in the doctor's zone** (their city's `timezone`), not the device's. Send the date the patient picked as `YYYY-MM-DD`. A date before the doctor's own today is a `422` on `date` (`booking.date_in_past`); "today" is the doctor's, so at 6 pm in Los Angeles that date is still bookable even though UTC is already on the next day.
- **`starts_at`/`ends_at` are exact instants** with the offset in force on that date, so daylight saving is already handled (e.g. `-07:00` on 2026-10-31, `-08:00` on 2026-11-01 in Los Angeles). **Send the chosen slot's `starts_at` back unchanged as `scheduled_at`.** Never rebuild a time from `start_time` plus the device's clock.
- **`start_time`/`end_time` stay for display** (the Morning/Afternoon/Evening buckets in §7.2 item 3 can keep reading `start_time`). They are wall-clock times in `timezone`.

`TreatmentResource`:
```json
{
  "id": 10, "treatment_category_id": 1,
  "category": { "id": 1, "name": "Preventative Dentistry", "group": "preventative", "sort_order": 1, "is_active": true },
  "name": "Teeth Cleaning", "description": "...", "standard_amount": "250.00",
  "is_quick_select": true, "requires_tooth_chart": false, "is_active": true
}
```
- **`treatment_category.group` (`preventative`/`cosmetic`, via `GET /enums` → `treatment_category_group`) is exactly the app's two `procedureGroups` tabs** — fetch categories (there's no standalone categories-list endpoint for the patient app today; derive the two group tabs from the distinct `category.group` values across a `GET /treatments` call, or ask the backend for a lightweight `GET /treatment-categories` read if that's cleaner — currently only the admin-gated CRUD version exists).
- **`standard_amount` is a flat catalog price, the same for every doctor** — there is no per-doctor, per-treatment fee. The directory's `starting_fee` (§6) is the closest thing to "what will this actually cost with this doctor," and even that's just the cheapest catalog item they've claimed expertise in, not a real quote for a specific service.

### 7.2 ⚠️ Booking-flow contract mismatches to resolve before wiring

The app's booking flow was designed against no real backend and makes assumptions the API doesn't support as-is. Resolve each of these (a product/UX call, not just a wiring detail) before building this phase:

1. ~~**Multi-select procedures vs. one `treatment_id` per appointment.**~~ — resolved: `POST /patient/appointments` now takes `treatment_ids: int[]` (§7.3), so `procedures_step.dart`'s existing multi-select can be wired as-is, one array entry per selected procedure. No UI redesign needed after all.
2. ~~**Tooth chart / problem picker (emergency flow) has no matching booking field.**~~ — resolved: `POST /patient/appointments` now takes optional `tooth_number` (1-32) and `problem_description` (free text) fields (§7.3), distinct from `visit_instructions` (still logistics/access notes, e.g. "ring the doorbell twice"). `tooth_step.dart`'s tooth selector + problem/description fields map directly onto these — no string-folding workaround needed. Not restricted to the emergency `visit_type` — send them whenever the user filled them in, regardless of branch.
3. ~~**Fixed Morning/Afternoon/Evening slot lists must be replaced entirely**~~ — resolved (product decision): keep the three time-of-day buckets in the UI, but derive them client-side from each real slot's own `start_time` (§7.1) instead of hardcoding a fixed list of times inside each bucket — e.g. before noon → Morning, noon–5pm → Afternoon, after 5pm → Evening. A bucket with no real slots that day/doctor simply doesn't render, rather than showing three buckets of fake availability for every doctor and date like the mock does today.
4. ~~**Insurance "coverage estimate" and card-photo capture**~~ — resolved (product decision), see §4's note: keep the client-side rough estimate, explicitly labeled as not a real quote. Card-photo capture itself is unaffected by this decision — `POST /patient/insurance`'s `card_front`/`card_back` upload already works end-to-end (§4), this was only ever about the coverage-estimate math next to it.
5. **`confirmBooking()` today never adds the new appointment to any visible list** (the mock's "Upcoming Visits" is static). Once wired, after a successful `POST /patient/appointments`, either optimistically insert the returned `AppointmentResource` into the visits list or simply refetch `GET /patient/appointments` — don't keep the current no-op behavior.

### 7.3 Booking, list, detail, and lifecycle

| Endpoint | Purpose | Body |
|---|---|---|
| `POST /patient/appointments` | Create a booking | see below |
| `GET /patient/appointments?patient_profile_id=` | Paginated, newest `requested_at` first — this is both "Upcoming" and "Past" in the app; split client-side by `status`/`scheduled_at` (see note below) | — |
| `GET /appointments/{id}` *(shared, no `/patient` prefix)* | One appointment's full detail | — |
| `PATCH /appointments/{id}/reschedule` *(shared)* | | `{ scheduled_at, reason }` — `scheduled_at` is an open slot's `starts_at` (§7.1), sent unchanged; `reason` required, ≤500 chars, matching the app's `ReasonSheet` |
| `PATCH /appointments/{id}/cancel` *(shared)* | | `{ reason }` — required, ≤500 chars |
| `GET /patient/appointments/{id}/recommendations` | Post-visit "recommended next" procedures for a completed visit | — |
| `POST /patient/appointments/{id}/payment` | Pay (or defer) — see §9 for the shape | |
| `POST /patient/appointments/{id}/review` | Rate the visit — see §6 for review shape; `{ doctor_rating: 1-5, app_rating?: 1-5, comment?, would_recommend? }`, only once, only after `status === "completed"` |

**Booking request body** (`request_mode: "direct"` requires `doctor_profile_id` + `scheduled_at`; `"open_pool"` must omit `doctor_profile_id` and leave the request unassigned for any doctor to claim):
```json
{
  "patient_profile_id": 12,
  "treatment_ids": [10, 14],
  "visit_type": "at_home",
  "request_mode": "direct",
  "doctor_profile_id": 9,
  "scheduled_at": "2026-09-20T10:00:00+04:00",
  "visit_instructions": "Ring the doorbell twice.",
  "tooth_number": 14,
  "problem_description": "Sharp pain when chewing.",
  "insurance_id": 4,
  "address": {
    "street": "...", "apartment": null, "note": "...",
    "city": "Dubai", "state": "Dubai", "zip": "00000",
    "latitude": 25.2, "longitude": 55.3
  }
}
```
- **`treatment_ids`** is a non-empty array — one entry per selected procedure, no duplicates (rejected with a `422`), each must be an active catalog id. `procedures_step.dart`'s existing multi-select maps directly onto this, order preserved (the first entry becomes the appointment's "primary" treatment, see below).
- **`visit_type`** (`at_home` / `emergency`, via `GET /enums`) is the routine-vs-emergency branch the app's `branch_step.dart` already captures — map it directly.
- **`address.latitude`/`address.longitude` are required**, not optional, despite other address fields being nullable — the app's `AddressEditor` needs a way to resolve/confirm coordinates (geocoding or the device's current location, same `geolocator` package already used for the map) before submitting, not just free-text fields.
- **Before this call ever succeeds, the account must clear three prerequisites** (`booking.missing_requirements` — a `422` with an `errors.missing_requirements` array of human-readable strings if not): phone verified (§2), the target profile's date of birth present (§3), and the current consent-form version signed for that profile (§5). Check/surface these proactively in the UI (the app's `confirm_step.dart` already has a pre-booking checklist screen — wire its three checks directly to these three facts) rather than only reacting to the 422.
- **`scheduled_at` is the chosen slot's `starts_at` (§7.1), sent unchanged.** It must carry an offset or `Z`: a time with no zone (`2026-09-20T10:00:00`) is a `422` on `scheduled_at` (`booking.scheduled_at_needs_timezone`), never silently read as UTC. Any offset naming the same instant is accepted (`2026-09-20T06:00:00Z` books the same slot as `2026-09-20T10:00:00+04:00`), but a slot's clock time sent as UTC is a different instant and fails with `booking.slot_not_available`. Applies to reschedule too.
- **A direct-mode `scheduled_at` is re-validated server-side against the doctor's real availability** — if the slot filled up between the app showing it and the user submitting, expect a `422` on `scheduled_at` (`booking.slot_not_available`) and re-fetch availability.
- **`tooth_number` (1-32) and `problem_description` are both optional and independent of `visit_type`** — the emergency flow's `tooth_step.dart` maps directly onto them, but nothing stops sending them on a routine `at_home` booking too if the UI ever wants that. `tooth_number` outside 1-32 is a `422`.

**`AppointmentResource`** (fields present depend on what's eager-loaded, but always includes at least):
```json
{
  "id": 42, "status": "accepted", "visit_type": "at_home", "request_mode": "direct",
  "treatment": { "id": 10, "name": "Teeth Cleaning", "standard_amount": "250.00", "...": "..." },
  "treatments": [
    { "id": 10, "name": "Teeth Cleaning", "standard_amount": "250.00", "...": "..." },
    { "id": 14, "name": "Composite Filling", "standard_amount": "180.00", "...": "..." }
  ],
  "patient_profile": { "id": 12, "first_name": "Amina", "...": "..." },
  "doctor_profile": { "id": 9, "first_name": "Sara", "...": "..." },
  "requested_at": "...", "scheduled_at": "2026-09-20T06:00:00+00:00",
  "timezone": "Asia/Dubai", "scheduled_at_local": "2026-09-20T10:00:00",
  "visit_address": { "street": "...", "...": "..." },
  "visit_instructions": "...",
  "tooth_number": 14, "problem_description": "Sharp pain when chewing.",
  "insurance": { "...": "..." },
  "invoice": { "...": "...", "status": "pending" },
  "consent_confirmed_at": "...", "cancellation_reason": null, "rescheduled_reason": null,
  "completed_at": null, "created_at": "...", "updated_at": "..."
}
```
- **`scheduled_at` is the UTC instant; show `scheduled_at_local`** (decision #7). `timezone` is the visit's IANA zone: the doctor's city, then the patient profile's city (an unclaimed open-pool request has no doctor), then the platform default. `scheduled_at_local` is `YYYY-MM-DDTHH:MM:SS` with no offset, in that zone — display it as-is (add the zone abbreviation when `timezone` differs from the device's), and keep `scheduled_at` for sorting and countdowns only. Both are `null`-safe: `scheduled_at_local` is `null` whenever `scheduled_at` is. Every endpoint that embeds `AppointmentResource` (list, detail, home dashboard's `upcoming_appointments`, booking/reschedule/cancel responses) carries both fields.
- **`treatment` is the first-selected/"primary" treatment; `treatments` is the full set actually covered** — for a single-procedure booking they contain the same one item; render `treatments` (not just `treatment`) anywhere the UI should show everything covered by the visit (e.g. the confirm/summary screen, invoice line items). `treatment` is kept only because it's what every existing one-treatment display already reads.
- **`status`** (`open`/`pending`/`accepted`/`in_progress`/`completed`/`rescheduled`/`cancelled`/`declined`, via `GET /enums` → `appointment_status`) is the single source of truth for "Upcoming" vs. "Past" in the Visits screen — treat anything not `completed`/`cancelled`/`declined` as upcoming, the rest as past (an `open`/unclaimed request should probably render as "finding a dentist..." rather than either bucket, a UI decision the current mock doesn't have to make since nothing is ever really pending).
- **Cancel/reschedule return `409`** (`booking.invalid_transition`) if the appointment isn't in a state that allows it (e.g. already completed/cancelled) — the app's `ReasonSheet` should only be reachable from an appointment whose current `status` is `pending`/`accepted`/`rescheduled` in the first place, same "hide the button rather than let the error surface" pattern used throughout the admin plan.
- **`recommendations`** backs the past-visit "recommended next" cards with "Schedule" buttons that start a new booking — map each `PostVisitRecommendationResource` (`{ id, appointment_id, treatment, custom_title, rank, reason, created_at }`) into that card, pre-filling a new booking flow with `treatment.id` when present.

**Verified**: _(fill in once wired)_

---

## 8. Medical history & self-service records (backend Phase 12)

| Endpoint | Purpose |
|---|---|
| `GET /patient/medical-history?patient_profile_id=` | The patient-reported history (or `data: null` if never filled in) |
| `PUT /patient/medical-history?patient_profile_id=` | Update (send only changed fields) |
| `GET /patient/invoices?patient_profile_id=` | Every invoice across the account's profiles |
| `GET /patient/prescriptions?patient_profile_id=` | Every doctor-issued prescription |
| `GET /patient/medical-reports?patient_profile_id=` | Every doctor-authored clinical note across every appointment |

`MedicalHistoryResource`: `{ id, patient_profile_id, conditions, allergies, current_medications, notes, updated_by, updated_at }` — `updated_by` is a `User` id (who last edited it — could be the patient themselves or a doctor); resolve a display name for it separately if needed, it's not embedded.

`InvoiceResource` (same shape used everywhere in the app — §9 for `PaymentResource`):
```json
{
  "id": 7, "appointment_id": 42, "treatment_description": "Teeth Cleaning",
  "standard_amount": "250.00", "advance_paid": "0.00", "remaining_amount": "250.00",
  "status": "pending", "due_at": null, "payments": [], "created_at": "..."
}
```
`PrescriptionResource`: `{ id, patient_profile_id, doctor_profile_id, appointment_id, prescription_template_id, medication_name, dosage, instructions, prescribed_at }`

`ClinicalNoteResource` (what the app's "Reports" tab actually shows — there is no patient-uploaded-file concept on the backend at all):
```json
{ "id": 5, "appointment_id": 42, "doctor_profile_id": 9, "chief_complaint": "...", "clinical_finding": "...", "recommendation": "...", "quick_sketch_transcript": null, "additional_notes": null, "created_at": "...", "updated_at": "..." }
```

- **Money fields (`standard_amount`, `advance_paid`, `remaining_amount`) are decimal strings** (`"250.00"`), never a JS/Dart double straight off the wire — parse deliberately for display/currency formatting, don't do arithmetic on the raw string.
- **⚠️ The app's "Reports" tab (`record_detail_screen.dart`) is built around patient-uploaded files/x-rays with an "Upload a new report" action — there is no such feature on the backend.** `GET /patient/medical-reports` returns doctor-authored `ClinicalNote` entries, not patient-uploaded documents, and there is no file-upload endpoint for this at all (unlike insurance cards or legal-document credentials, which do support uploads). Rebuild this tab's content model around clinical notes (read-only, doctor-authored) rather than a file list, and either remove the "Upload a new report" action or treat it as a future backend request (§15) — don't wire it to anything that doesn't exist.
- **`GET /patient/invoices` has a "Pay $X now" action in the mock (`payInvoice`)** — wire that button to `POST /patient/appointments/{invoice.appointment_id}/payment` (§9), not a dedicated "pay this invoice" endpoint (there isn't one; payment is always initiated against the appointment, and the invoice is what comes back updated).

**Verified (2026-09-17):** Built as `lib/features/records/{data,domain,presentation}` (the Records tab covers only Reports/Prescriptions/Invoices — `medical-history` itself is still Profile's Phase 2 territory, untouched here). None of `ClinicalNoteResource`/`PrescriptionResource`/`InvoiceResource` embed a doctor name on the wire, so each detail screen resolves one by matching `appointment_id` against the already-loaded `AppointmentsViewModel` list instead of adding a second lookup call. `RecordsViewModel.load()` starts all three list fetches as independent futures and awaits them sequentially (same reasoning as `BookingViewModel.loadTreatments`, §14 Phase 5's note) rather than combining them in one `Future.wait` literal. `Invoice.isPaid` is derived from `remainingAmount <= 0` rather than matching an undocumented `status` string. "Pay $X now" opens a lightweight payment-method sheet (the same three options as the booking flow's `payment_step.dart`, same `payment_method_type` mapping) and calls `AppointmentsViewModel.payAppointment` scoped to `invoice.appointment_id`, then refreshes just the Invoices sub-tab. The mock's "Upload a new report" action is removed outright (§15 item 1 tracks the missing backend feature) rather than deferred with placeholder UI.

---

## 9. Payment & saved payment methods

| Endpoint | Purpose | Body |
|---|---|---|
| `GET /patient/payment-methods` | Saved cards/wallets across the account | — |
| `POST /patient/payment-methods` | Save a new one | `{ gateway_token, is_default? }` |
| `DELETE /patient/payment-methods/{id}` | | — |
| `POST /patient/appointments/{id}/payment` | Charge (or defer) for a specific appointment | `{ method, payment_method_id?, token?, amount?, save_payment_method? }` |

`PaymentMethodResource`: `{ id, provider, brand, last_four, is_default, created_at }` — the vault token itself is never exposed.

**`method`** is `apple_pay` / `credit_card` / `pay_later` (via `GET /enums` → `payment_method_type` — this is exactly the app's three fixed `PaymentOptionsScreen` choices, "CareCredit" is just the app's branding of `credit_card`). Rules:
- `pay_later` needs no `token`/`payment_method_id` at all — it just records intent to defer.
- `credit_card`/`apple_pay` need **either** a fresh one-time `token` (from whatever gateway SDK eventually gets integrated — there is none yet, see the warning below) **or** an existing `payment_method_id` this account owns.
- `amount` is optional — omit it to charge the full remaining balance, or send a specific amount for a partial payment.
- `save_payment_method: true` alongside a fresh `token` saves it for later reuse — always attached to the *paying* account's own profile, reusable across every dependent, never per-dependent.

Response: `{ "invoice": InvoiceResource, "payment": PaymentResource }`. `PaymentResource`: `{ id, invoice_id, amount, method, provider_reference, status, paid_at, created_at }` — `status` (`initiated`/`succeeded`/`failed`/`refunded`) is real for the `pay_later` (`initiated`) case, but for `credit_card`/`apple_pay` it will **always** currently come back `succeeded` — see the warning below.

- **⚠️ There is no real payment gateway wired up yet (backend Phase 11, not built).** Every `credit_card`/`apple_pay` charge is handled by a fake gateway that always succeeds and never actually touches a card network. This is fine to wire against today (the request/response contract above is real and will not change shape when Phase 11 ships), but **do not build any "payment declined"/retry UI against real gateway error codes yet** — there aren't any to receive. The `token` field can be any non-empty string during this phase (there is no real Stripe/Apple Pay SDK integration in the Flutter app to produce a real one either — that's a separate, larger frontend task tracked alongside backend Phase 11, not part of this plan).

**Verified**: _(fill in once wired)_

---

## 10. Messaging (shared with the doctor/admin apps)

| Endpoint | Purpose |
|---|---|
| `GET /conversations` | Every conversation this account is party to (one per patient-doctor pair, across every profile the account manages), each with `unread_count` |
| `GET /conversations/{id}/messages?since=` | Polling-based — `since` is an ISO timestamp, narrows to messages newer than it. Viewing a thread as a real participant marks the other side's messages read as a side effect |
| `POST /conversations/{id}/messages` | `{ body?, attachment? (file) }` — at least one of the two is required |

`ConversationResource`: `{ id, patient_profile, doctor_profile, appointment_id, last_message_at, unread_count, created_at }`
`MessageResource`: `{ id, conversation_id, sender_id, body, attachment_url, read_at, created_at }` — `attachment_url` is a short-lived (~30 min) signed URL, regenerated on every fetch, never cache/store it.

- **There is no "start a new conversation" endpoint.** A thread is created automatically the moment a direct booking is made or an open-pool request is claimed — by the time a patient has an appointment with a doctor, `GET /conversations` already has the thread. The app's two hardcoded threads (`sarah`, `support`) don't map to this model at all: there is no "support" conversation concept on the backend. **Resolved (2026-09-17, product decision): drop it.** No support thread will be built — "Message support" is out of scope for Phase 8; only real per-appointment doctor conversations get wired.
- **This is polling, not push/websockets** — the app should re-`GET .../messages?since=<last message's created_at>` on an interval while a thread is open, matching the "Online now" idea loosely (there's no real presence/online-status signal from the backend at all — that label in the app is decorative and has nothing to wire to).
- **Sending requires an actual conversation id** the account is a genuine participant in — `MessagePolicy` will `403` anything else.

**Verified (2026-09-17):** Built as `lib/features/messaging/{data,domain,presentation}`. `ConversationResource`'s embedded `doctor_profile` isn't fully pinned down (same caveat as the appointment's own embedded doctor, §7.3's note) — `ConversationModel` reuses the identical name-fallback chain `AppointmentModel._doctorNameFrom` already uses. Polling lives in the thread screen's own `Timer.periodic` (5s while open), not in `MessagingViewModel` itself, so the view model stays agnostic of widget lifecycle; `pollMessages` narrows with `since` set to the last held message's `created_at` and merges rather than replaces. Opening a thread's initial `loadMessages` call zeroes that conversation's local `unreadCount` to match the documented "viewing marks read" side effect, without a second round trip. Attachments reuse `ApiClient.postMultipart` (already built for the avatar upload) rather than hand-rolling `FormData`, and the composer's picker is the same `image_picker` gallery flow `insurance_step.dart` uses. The conversation list has no message-preview text to show (`ConversationResource` carries no last-message body, only `last_message_at`) — each row shows the dentist and a relative timestamp instead of a snippet.

---

## 11. Notifications & push

### 11.1 In-app notification feed

| Endpoint | Purpose |
|---|---|
| `GET /notifications` | Paginated, this account's own notifications only |
| `PATCH /notifications/read-all` | Mark every currently-unread notification for this account read, in one call |
| `PATCH /notifications/{id}/read` | Mark one read |

`NotificationResource`: `{ id, data, read_at, created_at }` — `data` is a generic payload, shape varies by `data.type` (`appointment_reminder`, `appointment_status_changed`, `new_message`, `document_expiring`, `new_nearby_open_request` — the last one is doctor-app-only and will never appear for a patient account). Each carries at least `type` and a pre-localized `title`; render generically off those two fields rather than hardcoding per-type copy, and fall back gracefully for any `type` value not in this list (more may be added later).

- **The bell badge/unread count is not returned as a single number anywhere** — compute it client-side from the paginated list (`read_at === null`) or maintain a running local count as notifications are marked read, the same way the app's `AppState.unreadNotifications` already works, just backed by real data now.
- ~~**No "mark all read" endpoint**~~ — resolved: `PATCH /notifications/read-all` (above) marks every unread notification for the account read in a single call; no per-item loop needed.

### 11.2 Push notifications (device registration)

| Endpoint | Purpose | Body |
|---|---|---|
| `POST /device-tokens` | Register/refresh this device's push token | `{ token, platform }` — `platform` is `ios` or `android` |
| `DELETE /device-tokens` | Deregister (also happens automatically via `POST /logout`'s optional `device_token`, §2) | `{ token }` |

- **This requires an actual FCM SDK integration in the Flutter app** (`firebase_messaging` or equivalent) that isn't in `pubspec.yaml` today — obtaining the device's FCM token is a prerequisite step this plan doesn't cover (it's Firebase/native-platform setup, not a Teethi API call). Once the app has a token, register it right after login (and re-register on token-refresh events the SDK fires periodically) and deregister on logout.
- **Every notification type in §11.1 is also pushed** (backend Phase 12 added `FcmChannel` to all five notification classes) — a registered device gets a native push for the same events that appear in the in-app feed, so there's no separate "push preferences" list to build against different event types; it's all-or-nothing per device today.

**Verified**: _(fill in once wired)_

---

## 12. `GET /enums` — enum label catalog

Same shared, unauthenticated-content endpoint documented in full in `docs/plans/ADMIN_DASHBOARD_FRONTEND_PLAN.md` §5 — not paginated, respects `Accept-Language`, fetch once per session/locale and cache. The groups the **patient app** actually needs: `user_role`, `otp_channel`, `gender`, `blood_type`, `patient_relationship`, `treatment_category_group`, `document_template_type`, `document_audience`, `insurance_relationship`, `visit_type`, `appointment_request_mode`, `appointment_status`, `invoice_status`, `payment_method_type`, `payment_status`, `device_platform`. Ignore every other group in the response (doctor/admin-domain enums) rather than trying to render them somewhere. Note: `otp_channel` now includes a third value, `password_reset` — internal to how the backend tracks OTP codes (§2), never something the app needs to send or display itself.

---

## 13. Mock data → real API field mapping (quick reference)

For whoever wires each screen — the mock model on the left, what actually backs it on the right. Use this as a checklist for what to delete from `lib/data/mock_data.dart` and `lib/models/models.dart` as each phase below lands.

| Mock (`lib/models/models.dart` / `mock_data.dart`) | Real source |
|---|---|
| `Doctor` (`name, qualification, fee, years, treated, rating, recommended, hours, availableToday, nearby, lat, lng`) — **no id** | `ProviderResource` (§6) — has a real `id`; `qualification` has no direct equivalent (drop it or derive from `specializations`); `treated`/`rating`/`recommended` → `patients_treated_count`/`average_rating`/`recommended_percentage`; `fee` → `starting_fee` (nullable now); `nearby` → compute from `distance_km` presence/threshold instead of a static flag |
| One hardcoded review per doctor | `GET /patient/providers/{id}/reviews` (§6) — real, paginated, one per completed appointment |
| `PatientProfile` (`title, first, middle, last, suffix, dob, gender, blood, address, email, mobile, bio, languages, avatarTone, familyMembers`) | `PatientProfileResource` + `UserResource` (§2–§3) — `gender` → `gender` (`male`/`female`/`other`, via `GET /enums`), `blood` → `blood_type` (`a_positive`…`o_negative`, via `GET /enums`), `bio` → `bio`, all real now. `avatarTone` (fake photo stand-in) → a real uploaded photo via `POST /patient/profile/photo`, exposed as `UserResource.profile_image` (a signed URL, not a color tone) — the avatar-tone picker UI can be replaced with an actual camera/gallery photo picker |
| `FamilyMember` (`name, relation, age` — all strings, no id) | `PatientProfileResource` dependents (§3) — real ids, structured `first_name`/`last_name`/`date_of_birth`/`relationship_to_account_holder`, not free-text `age` |
| `UpcomingVisit` / `PastVisit` (no ids, no doctor/address reference) | `AppointmentResource` (§7.3) — has real ids and full relations; the mock's separate "upcoming" vs. "past" lists become one endpoint filtered client-side by `status` |
| `MessageThread`/`ChatMessage` (`sarah`, `support` hardcoded) | `ConversationResource`/`MessageResource` (§10) — real threads only exist once an appointment connects a patient and doctor; "support" has no backend equivalent (product decision) |
| `NotificationItem`/`NotificationGroup` | `NotificationResource` (§11.1) — grouping by "Today"/"This week" becomes a client-side bucketing of `created_at`, not a server-provided grouping |
| `ConsentForm` (`key, title, note, pages, signedByDefault, date?, body?`) | `DocumentTemplateResource` in the signed-flow shape (§5) — `key` → `id`, `signedByDefault` has no equivalent (drop it), `pages` isn't tracked (the `body` is one long string to paginate/scroll client-side if needed) |
| `RecordItem` (reports, `icon, title, meta`) | `ClinicalNoteResource` (§8) — **not a file list**, see the warning in §8 |
| `Prescription` (`title, meta, date`) | `PrescriptionResource` (§8) — `medication_name`/`dosage`/`instructions`/`prescribed_at` |
| `InvoiceItem` (`id, title, meta, total, due?, status`) | `InvoiceResource` (§8–§9) — `total`→`standard_amount`, `status` is now a real enum with real transitions, not a locally-toggled `paidInvoiceIds` set |
| `Address` (`street, apt, city, state, zip, note`) | `AddressResource` (§3, §7) — same field names, good match; `latitude`/`longitude` are required by booking (§7.3) even though nullable on the resource generally |
| Fixed `PaymentOptionsScreen` choices (`carecredit`/`apple`/`later`) | `payment_method_type` enum: `credit_card`/`apple_pay`/`pay_later` (§9) |
| Hardcoded `418207` OTP | Gone entirely — real code from email/SMS (§2) |

---

## 14. Phased build plan

### Phase 0 — API foundation (nothing today; build all of this first)
- [x] Add an HTTP client dependency (`http` or `dio`) to `pubspec.yaml` — currently absent entirely
- [x] Environment config for the API base URL per build target (local/staging/prod)
- [x] Secure token storage (`flutter_secure_storage` or equivalent — currently no persistence package of any kind) so "Keep me signed in" (§2) actually survives an app restart; without this, the app still resets to onboarding every launch exactly as it does today
- [x] A single place that attaches `Authorization: Bearer <token>` and `Accept-Language` to every request
- [x] Central handling for all four response envelope shapes (§1) — one consistent "did this fail, and why, and what do I show" model every screen can rely on
- [x] Global `401` handling → clear stored token, return to the login screen — the app-launch-restore case was already handled; a mid-session `401` from any other call now goes through the same `ApiClient` choke point (`onUnauthorized`, wired in `main.dart` to `AuthViewModel.handleUnauthorized`), which clears the token, drops back to the login screen, and shows a toast ("Your session has expired — please sign in again.") there once the screen mounts.
- [~] Global `429` handling → a generic "too many attempts, try again shortly" surface (relevant especially to the OTP resend buttons, §2) — `RateLimitFailure` exists and its message reaches the screen via the same per-screen error banner every other failure uses; there's no separate app-wide toast/snackbar service, so this is "handled" but not "global" yet.
- [x] A pagination helper reading `meta`/`links` consistently — every list screen from Phase 3 onward needs it — done (2026-10-17): `lib/core/network/paginated.dart` (`getPage`/`getAll`/`getLatest` on `ApiClient`, ported from the dentist app) is used by every list data source — fetch-all for treatments/insurance/conversations/dependents, load-more for visits/records/the dentist directory/reviews, and newest-first (from `meta.last_page`) for message threads
- [ ] Fetch `GET /enums` (§12) once per session/locale and cache it — not started; Phase 1 needed no enum labels (`user_role`/`user_status` are fixed 3/2-value sets, hardcoded as a plain Dart enum instead of round-tripping through `/enums`)
- [~] Replace every `Doctor`/`PatientProfile`/etc. mock model with a real class carrying `fromJson`/`toJson` matching §13's mapping — done incrementally per phase below, not all at once: `User`/`UserModel` (§2's `UserResource`) is real now; every other mock model is untouched until its own phase

**Verified (2026-09-13):** Built as a feature-first Clean Architecture (`lib/core/` + `lib/features/<name>/{data,domain,presentation}`), decided together with this same pass — see `ARCHITECTURE.md` in the frontend repo for the folder convention and the phase-mapping table. Stack choices: `provider` kept for state (per-feature `ChangeNotifier` view models, not one `AppState` god-class going forward), `get_it` for DI, a hand-rolled `Result<T>`/`Failure` pair (`lib/core/error/`) instead of a functional package. HTTP client is `dio` (not `http`) for its interceptor support. `flutter pub add` bumped `flutter_secure_storage` to `^11.1.1` to resolve a transitive `win32` version conflict with the existing `geolocator` dependency — noted here since it's a version floor future `flutter pub get` runs should respect, not something to downgrade.

### Phase 1 — Authentication & session
- [x] Onboarding carousel — unchanged, no backend involved
- [x] Register screen → `POST /register`, immediately followed by `PUT /phone` with the collected mobile number (§2) — remove the mobile-number-at-registration assumption
- [x] Verify-email screen → `POST /email/verify` / `POST /email/resend`; delete the hardcoded `418207` shortcut entirely
- [x] Verify-phone screen → `POST /phone/verify` / `POST /phone/resend`
- [x] Login screen → `POST /login`; surface invalid-credentials vs. inactive-account distinctly (both currently just call `AppState.login()` unconditionally)
- [x] Forgot/reset password → `POST /forgot-password`, `POST /reset-password` (§2) — the same 6-digit code-entry UX as every other verification step; the app's existing `ForgotPasswordScreen` flow already matches this
- [x] Wire "Keep me signed in" to the Phase 0 secure-storage decision
- [x] Sign out → `POST /logout`, clear stored token (+ device token if push is already wired, §11.2)

**Verified (2026-09-13):** Flutter web (`flutter run -d web-server`), Dio 5.11.1. Full round-trip (register → verify email → verify phone → login → persisted-session restart → logout) has **not** been exercised against a real running backend in this pass — no Laravel server was available in this session — only compiled/rendered against the UI with `flutter analyze` (0 errors) and a manual click-through of Onboarding → Login → Register in the browser preview. Re-verify against a live backend (real OTP inbox, real Twilio SMS once configured) before calling this phase done.

Two corrections to the mock UI discovered while wiring, beyond what §2 already flagged:
- `POST /login` and `POST /forgot-password` take `{ email, ... }` only, no phone-number identifier — but the mock `LoginScreen`/`ForgotPasswordScreen` had a "mobile number or email" field (`Validators.identifier`). Changed both to an email-only field (`Validators.email`); logging in or resetting by phone number was never something the backend could do.
- `POST /register` requires `password_confirmation` — the mock `RegisterScreen` only ever collected one password field. Added a "Confirm password" field (reusing the existing `Validators.confirmPassword` from `ForgotPasswordScreen`).

Not wired (no mock screen assumed it, so out of this phase's scope, but the repository/use-case/data-source methods exist and are ready): `POST /change-password`. No settings screen currently offers changing a password, so there's nothing to wire it to yet — flag as a real gap if a "change password" screen is ever added to Settings.

Social ("Continue with Google/Apple") buttons on Login/Register are unchanged visually but no longer fake-succeed a login — there's no social-auth endpoint in this backend at all, so tapping either now shows a "not available yet" notice instead.

### Phase 2 — Profile & family
- [x] Profile view/edit → `GET`/`PATCH /patient/profile`, `PUT /patient/address` (§3), including `gender`/`blood_type`/`bio` (§3) — all three now real
- [x] Avatar photo → `POST`/`DELETE /patient/profile/photo` (§3) replaces the mock's 5-color-tone picker with a real camera/gallery upload
- [x] Family members list/add/edit → `GET/POST/PATCH/DELETE /patient/dependents` (§3) — real ids replace the mock's name-keyed list
- [ ] Insurance list/add/edit/delete, incl. card photo upload → `/patient/insurance` (§4) — **not done this pass**, see split note below
- [ ] Payment options screen (saved methods only — the actual pay action is Phase 6) → `GET/POST/DELETE /patient/payment-methods` (§9) — **not done this pass**, see split note below
- [ ] Legal/help/language screens — confirmed no backend involvement (§13); leave as static content — unchanged, not touched this pass

**Verified (2026-09-13):** Built as `lib/features/profile/{data,domain,presentation}`, same Clean Architecture pattern as Phase 1's Auth slice (get_it DI, `Result<T>`/`Failure`, a `ProfileViewModel` `ChangeNotifier`). `PatientProfileResource` backs both "my own profile" and each dependent (`isAccountHolder` distinguishes them) — one entity, no separate "Dependent" model, matching §3. `Gender`/`BloodType` are hardcoded Dart enums with the exact wire values from §3's text (`male`/`female`/`other`, `a_positive`…`o_negative`) — same reasoning as Phase 1's `UserRole`/`UserStatus`, since the `GET /enums` cache still isn't built (Phase 0 tracking). `relationship_to_account_holder` stayed a free-text `String` rather than a guessed enum, since the plan doesn't list its actual values. Avatar upload uses `image_picker` (new dependency) + `ApiClient.postMultipart`, sending raw bytes rather than a file path — `XFile.path` is a `blob:` URL on Flutter web and unusable with `dart:io`-based file APIs, so bytes are the only cross-platform-safe option. `profileImage` lives on `UserResource` (Auth's `User`), not `PatientProfileResource` — `ProfileViewModel.uploadPhoto`/`deletePhoto` return the outcome to the calling screen, which pushes it into `AuthViewModel.setCurrentUser`/`User.withProfileImage` itself, rather than `ProfileViewModel` reaching into a second `locator<AuthViewModel>()` instance (which `get_it`'s `registerFactory` would hand back as a *different* object than the one `main.dart`'s `Provider` holds).

Two feature-shaped gaps intentionally not closed this pass — flagged for whoever finishes Phase 2 rather than guessed at:
- **No `city_id` picker.** `PatientProfileResource.city` (§3, the service-area reference) is a different concept from `address.city` (a free-text mailing-address field, which *is* editable now) — setting it needs a `GET /cities` catalog fetch that doesn't exist anywhere in the app yet. `EditProfileScreen` omits a city-service-area field entirely rather than guess at ids. The register screen's own city dropdown (`RegisterScreen`, already flagged as unwired in Phase 1) has the same blocker — building one small shared Cities lookup would unblock both call sites at once.
- **`PUT /patient/address`'s own response shape is unconfirmed.** The plan doc doesn't pin down what that call returns, so `ProfileRepositoryImpl.upsertAddress` ignores its body and refetches `GET /patient/profile` instead. Works, but does one extra round-trip; revisit if the real response turns out to already carry the updated profile/address.

Booking's `confirm_step.dart` (date-of-birth checklist item) and `visit_step.dart` ("Use profile" address button) were updated to read the real `ProfileViewModel` instead of the retired `AppState.profile`/`AppState.dob` — both are still pre-Phase-5 mock screens otherwise, this was the minimum change to keep them compiling and correct against real data.

**Split rationale:** Insurance (`/patient/insurance`, §4) and payment methods (`/patient/payment-methods`, §9) are separate REST resources reused later by booking (Phase 5) and records (Phase 7), not sub-fields of `PatientProfileResource` — splitting them into their own pass (rather than guessing at scope to fit everything under one "Phase 2" commit) keeps each `lib/features/<name>/` slice reviewable on its own. Do this as "Phase 2b" before or alongside Phase 5, since booking needs both.

**Follow-up fix (2026-09-14):** The first pass left no way to actually change the mobile number after registration — the Contact card only showed it read-only with a "Send me a new code" resend button, which only helps when the number is already correct but unverified. Added `AuthViewModel.updatePhone` (a second call site for the existing `SetPhoneUseCase`/`PUT /phone`, alongside the one already used at registration) and a "Change mobile number" sheet on the main Profile screen; since §2 says `PUT /phone` clears any prior verification, a successful save routes straight into `VerifyScreen(isEmail: false)` rather than leaving a stale "Verified" badge on screen.

That last navigation itself needed one more fix: `VerifyScreen` has no `Scaffold`/`Material` of its own — `main.dart`'s `_AppGate` is the only place that ever renders it today, and it supplies the wrapping `Scaffold` for every pre-login phase. Pushing `VerifyScreen` directly via `Navigator.push` from inside the already-authenticated app shell (as the new "Change mobile number" flow does) skipped that wrapper entirely, so `CodeEntryField`'s `TextField` threw "No Material widget found" the moment the screen opened. Fixed by wrapping the pushed route in the same `Scaffold(backgroundColor: AppColors.white, body: SafeArea(...))` `_AppGate` uses, at the call site — `VerifyScreen` itself is unchanged. Any future code that pushes an auth screen (`LoginScreen`, `RegisterScreen`, etc.) directly rather than through `_AppGate`'s phase switch will need the same wrapper.

Also fixed two Provider-scoping bugs discovered wiring `EditProfileScreen` against a real backend (both now corrected in the code, noted here since they're easy to reintroduce by accident in a later phase): a `ChangeNotifierProvider` scoped to one route's widget subtree is invisible to anything reached via `Navigator.push` from inside it, since a pushed route is a sibling `Overlay` entry, not a descendant — `ProfileViewModel` had to move into the app-wide `MultiProvider` next to `AuthViewModel`. And a `ChangeNotifierViewModel`'s `load()`-style method that calls `notifyListeners()` before its first `await` cannot be invoked directly from `initState()` (that runs mid-build) — it must be deferred with `WidgetsBinding.instance.addPostFrameCallback`.

### Phase 3 — Home dashboard
- [x] One call, `GET /patient/dashboard` (§16), backs the whole screen: "Next visit" hero + "Upcoming treatments" list (`upcoming_appointments`), notification bell badge (`unread_notifications_count`), "Before your first visit" checklist (`profile.missing_fields`/`profile.profile_completed_at`/`consent_pending`), and the quick-book bento tiles (`quick_treatments`) — a close, ready-made match for the mock's `homeServices`

**Verified (2026-09-14):** Built as `lib/features/home/{data,domain,presentation}`, same pattern as Auth/Profile. `DashboardSummary` reuses Profile's `PatientProfile` entity for the `profile` field (same `PatientProfileResource` shape, §3/§16) rather than a duplicate type; `AppointmentSummary`/`QuickTreatment` are new, deliberately trimmed entities (id/status/title/doctor/schedule; id/name/price) — the full `AppointmentResource`/`TreatmentResource` shapes belong to Visits (Phase 6) and Booking (Phase 5), not Home. `consent_pending` is a plain boolean on the wire (§16), so the task-strip copy dropped the mock's "N consent forms to sign" count in favor of one fixed "Sign your consent form" line. `quick_treatments` has no duration field (unlike the mock `Procedure.mins`) — the bento tile's "· N min" clause is now conditional and simply omitted for API-backed tiles. The "Dental emergency" tile is not backed by any treatment record on the wire; it stays a fixed local shortcut into the booking flow's emergency branch (unchanged from the mock). An `open`/still-unassigned appointment has no `scheduled_at` yet — the hero card renders "Finding a dentist for you" for that case, and an empty `upcoming_appointments` list renders a "Book your first visit" prompt instead of an empty card.

### Phase 4 — Doctor directory
- [x] Dentists list/search/filter tabs → `GET /patient/providers` (§6); switch the "Nearby" tab's map pins from `MockData.doctors` to real results, still using the device's actual GPS (the map/geolocation itself already works, only the plotted data was mock)
- [x] Doctor detail screen → `GET /patient/providers/{id}` + `GET /patient/providers/{id}/reviews` (§6); replace the doctor model's name-based identity with the real `id` everywhere it's referenced (booking flow selection, favorites if ever added, etc.)

**Verified (2026-09-14):** Built as `lib/features/doctors/{data,domain,presentation}` (folder named `doctors`, not `providers`, to avoid clashing with the `provider` package's own name in imports). New `DoctorProvider`/`DoctorReview` entities carry the real `id` and every `ProviderResource`/`ReviewResource` field (§6) — `qualification` has no equivalent on the wire (§13), so the detail/grid views derive a display line by joining `specializations` instead. Search/filter is server-side now (`q`, `available_today`, `latitude`/`longitude` query params on every keystroke/tab change, debounced 350ms for typing), replacing the mock's client-side `.where()` filter. Not paginated — same "first page only" simplification `ProfileRepositoryImpl.getDependents` already uses, since the Phase 0 pagination helper isn't built yet.

**Deliberately left on mock data this pass, flagged rather than guessed at** (same reasoning as Phase 2's own split note): the booking flow's dentist step (`lib/screens/booking/dentist_step.dart`) still reads `MockData.doctors`/the mock `Doctor` model — it's Phase 5 scope, not touched here, and today's `DoctorDetailScreen`'s "Book a visit" button doesn't carry the tapped doctor into the booking flow at all (it never did, even against the mock). The mock `Doctor` model gained an optional `id` field (`null` for every mock instance) purely so the Nearby tab's map — a widget shared, unmodified, with the booking flow's own map — could plot real `DoctorProvider` pins through the same `NearbyDentistsMap` without a parallel widget. Wiring dentist selection into a real booking submission is Phase 5's job, once `POST /patient/appointments`'s `doctor_profile_id` is actually being sent.

### Phase 5 — Booking flow
- [x] Resolve the remaining items in §7.2 (insurance estimate, fixed slots) as explicit product decisions before writing screen code — the multi-select-procedures and tooth-chart mismatches are already resolved (§7.2 items 1-2); the slot buckets and insurance estimate are now resolved too (§7.2 items 3-4, decided 2026-09-14)
- [x] Branch/procedures/tooth/visit steps → `GET /treatments` variants (§7.1), `GET /patient/availability` (§7.1) replacing the fixed slot lists; `procedures_step.dart`'s existing multi-select maps directly onto `treatment_ids: int[]`, and `tooth_step.dart` maps directly onto `tooth_number`/`problem_description` (§7.3)
- [x] Dentist-in-booking step → same `GET /patient/providers` call as Phase 4, scoped/filtered to the chosen treatment/city
- [x] Insurance step → select from `GET /patient/insurance` or add new inline (§4)
- [x] Confirm step → check the three real booking prerequisites (§7.3) instead of the current local `Set`-based checklist
- [x] Submit booking → `POST /patient/appointments` (§7.3), handling the `missing_requirements` and `slot_not_available` errors explicitly
- [x] Payment step → `POST /patient/appointments/{id}/payment` (§9), with the Phase-11-gateway caveat noted
- [x] Confirmation step's "rate and review" → wire to `POST .../review` (§7.3) once the appointment is actually completed (i.e. this immediate post-booking screen can't submit a review yet — a booking is never `completed` at the moment it's created; either drop this step from the confirmation screen or move the review prompt to the moment a visit is later marked completed, e.g. surfaced from Phase 6's past-visit list instead)
- [x] After success, refetch/insert into the real appointments list (§7.2 item 5) instead of the current no-op

**Verified (2026-09-14):** Built as three new feature slices — `lib/features/booking/` (treatment catalog + availability reads), `lib/features/insurance/` (list/add, the split-off "Phase 2b" work this phase needed anyway), and `POST /patient/appointments`/`.../payment` added onto the existing `lib/features/appointments/` (Phase 6) rather than a new feature, since they're the same resource family as cancel/reschedule. `AppState` gained real typed fields where it only ever held mock strings before: `selectedDoctor` is now a `DoctorProvider?`, `pickedProcedures` holds stringified real treatment ids, `selectedSlot` is a real `AvailabilitySlot?`, `selectedInsurance` an `InsuranceRecord?`, and `lastBookedAppointment` the real created `Appointment` the payment/confirmation steps key off.

Several product/UX calls beyond §7.2's original two were needed to make the real contract work, each made here rather than left as an open question:
- **Booking-flow step order changed for routine (`at_home`) bookings**: dentist is now chosen *before* visit/time (`procedures → dentist → visit → insurance → confirm → payment → confirmation`), not after. `GET /patient/availability` requires a `doctor_profile_id` (§7.1), so the real per-slot Morning/Afternoon/Evening buckets (§7.2 item 3) have nothing to query until a doctor is picked — the mock's old order (visit before dentist) only ever worked because its slot lists were fake and identical for every doctor.
- **Emergency bookings are `request_mode: "open_pool"`**, not `"direct"` — branch_step's own copy ("we find the soonest dentist") already promised this; the mock's emergency route just never had a matching request mode before. This removes the dentist step from the emergency flow entirely (`open_pool` must omit `doctor_profile_id`, §7.3) and makes emergency bookings always-ASAP with no date/slot picker — there's no per-doctor availability to check against when no doctor is assigned yet.
- **Emergency bookings still need a non-empty `treatment_ids`** (§7.3) even though this flow never had a procedures picker — `requires_tooth_chart` on `TreatmentResource` (§7.1) is the one catalog flag that plausibly marks tooth-specific/emergency-shaped treatments, so `tooth_step.dart` now shows a compact single-select of treatments flagged that way (falling back to the full catalog if none are flagged, so emergency booking is never simply blocked).
- **Address coordinates come from the device's current-location GPS fix, not geocoding** — no geocoding service exists anywhere in this stack, so `AddressEditor` gained a "Confirm this address with my location" action (via `geolocator`, already a dependency) that must resolve before the visit step's Continue enables; editing the street/zip text afterwards clears the resolved coordinates again, since a GPS fix confirmed for one address says nothing about a different one just typed.
- **The insurance estimate is never actually deducted from what's charged** — since it's an explicitly-labeled rough guess with no real eligibility check behind it (§7.2 item 4's resolution), charging "total minus our own guess" would just be wrong against the real invoice. The payment step now always charges the full remaining balance (omitting `amount`, §9) and shows the estimate as a separate, clearly-labeled informational line only.
- **The confirmation screen's old "Rate and review" prompt was dropped entirely**, per this phase's own note above — Phase 6 already built the real version of this (a review prompt on a *completed* past visit, §7.3), and a booking is never `completed` the moment it's created. "Modify the schedule" now opens the Visits tab (which has the real reschedule flow, Phase 6) instead of re-entering the booking draft, since the appointment already exists as a real resource by that point.
- **`DoctorsViewModel` (Phase 4) needed its own separate instance for the booking flow.** The Dentists tab and the booking flow's dentist step both stay mounted at once (`RootShell`'s tab switcher never unmounts a tab) and search with different filters — sharing one instance would have each tab's search silently overwrite the other's results. `root_shell.dart` now wraps just the Booking tab's subtree in its own `ChangeNotifierProvider<DoctorsViewModel>`, shadowing the app-wide one from `main.dart` for that subtree only; every other shared view model in this booking flow (`AppointmentsViewModel`, `HomeViewModel`, `ProfileViewModel`, `AuthViewModel`) is safe to share as-is, since booking only reads their canonical state or performs writes that *should* be reflected app-wide (e.g. a newly created appointment showing up in Visits immediately).

Not paginated anywhere in the two new features — same "first page only" simplification used throughout. Card-photo capture for insurance re-used `image_picker` (already a dependency) with a new `ApiClient.postMultiFile` (insurance's `card_front`/`card_back` need two files in one request; the existing `postMultipart` only ever took one, for the avatar). `relationship_to_subscriber` uses the real `insurance_relationship` enum (`GET /enums`) via the same label/reverse-lookup pattern Profile's dependent form already established for `patient_relationship`.

### Phase 6 — Appointment management (Visits screen)
- [x] Upcoming/Past tabs → one `GET /patient/appointments` call, split by `status` (§7.3)
- [x] Cancel/reschedule → `PATCH /appointments/{id}/cancel`/`/reschedule` (§7.3), only enabling the action for appointments in a cancellable/reschedulable status
- [x] Past-visit detail / "recommended next" → `GET .../recommendations` (§7.3), "Schedule" buttons pre-filling a new Phase 5 booking
- [x] Rate-and-review prompt for a newly-completed past visit → `POST .../review` (§7.3, moved here per the Phase 5 note above)

**Verified (2026-09-14):** Built as `lib/features/appointments/{data,domain,presentation}`, same pattern as every prior feature. `Appointment`/`AppointmentTreatment` are trimmed to what the Visits screen and past-visit detail actually render (id/status/treatments/doctor/dates/invoice status/reasons) — deliberately not the full `AppointmentResource` shape (visit address, insurance, consent, etc.), which nothing here needs yet. Not paginated — same "first page only" simplification used by Profile's dependents and the Doctor directory. `GET /patient/appointments` already returns every field the list needs, so there was no separate `GET /appointments/{id}` detail fetch to add — the tapped list entry carries everything the past-visit screen shows.

The mock `VisitOutcome` model's "Services completed"/"Still pending"/"Medication added" sections were **entirely fabricated** — no `AppointmentResource` field distinguishes which covered treatments were actually done vs. still pending, and there's no backend concept of it. The past-visit detail screen now shows a plain "Treatments" list (name + price) plus invoice status and, for a cancelled visit, the real `cancellation_reason` — dropping the fabricated sections rather than guessing at a shape that doesn't exist on the wire (same reasoning as §8's Reports-tab note).

Reschedule needed a real new `scheduled_at`, not just a reason — the mock's "Choose a new slot" used to hand off into the (still-unwired, Phase 5) booking flow instead of actually rescheduling. It now opens the native `showDatePicker`/`showTimePicker` dialogs inline in the same sheet rather than building a bespoke calendar UI, then calls `PATCH /appointments/{id}/reschedule` directly.

There is no field indicating whether a completed visit has already been reviewed — `POST .../review`'s own docs don't describe a duplicate-submission error either. The "Rate this visit" prompt is hidden client-side, per screen-open, once a review is submitted successfully in that session; a real "already reviewed" signal (or a `422`/`409` on a second attempt) isn't handled specially yet.

"Schedule" (recommendations) and "All services" (bottom bar) still just open the generic, still-mock booking flow (`AppState.startBooking()`) without pre-filling a treatment — same as the Doctor directory's "Book a visit" button already did (§14 Phase 4's Verified note); there's no real id → mock booking-flow key mapping to build until Phase 5 itself is wired.

### Phase 7 — Records tab (Reports/Prescriptions/Invoices)
- [x] Rebuild the "Reports" sub-tab around `ClinicalNoteResource` (§8) — read-only, doctor-authored, not a file list; the "Upload a new report" action is removed outright (§15)
- [x] Prescriptions sub-tab → `GET /patient/prescriptions` (§8)
- [x] Invoices sub-tab → `GET /patient/invoices` (§8); "Pay $X now" → the same payment endpoint as Phase 5 (§9), scoped by `invoice.appointment_id`

**Verified (2026-09-17):** See §8's own "Verified" note above for the full build summary.

### Phase 8 — Messaging
- [x] ~~Decide the "support" thread's fate first~~ (§10) — resolved 2026-09-17: dropped, no support thread will be built
- [x] Thread list → `GET /conversations` (§10)
- [x] Thread view + polling → `GET /conversations/{id}/messages?since=` on an interval while open (§10)
- [x] Send message (text and/or attachment) → `POST .../messages` (§10)

**Verified (2026-09-17):** See §10's own "Verified" note above for the full build summary. The mock's "Message support" entry point (Profile → Help and contact) is removed — that screen now shows phone/email/hours only, per the resolved product decision.

### Phase 9 — Notifications & push
- [ ] In-app feed + mark-read (single and "mark all as read") → `GET /notifications`, `PATCH .../read`, `PATCH .../read-all` (§11.1); bucket by date client-side, no server-side grouping
- [ ] Integrate an FCM SDK (net-new dependency, native platform setup — not covered by this doc) and register the resulting token → `POST /device-tokens` (§11.2) right after login
- [ ] Deregister on logout → `DELETE /device-tokens` or the `device_token` param on `POST /logout` (§2, §11.2)

### Timezone contract — backend decision #7 (2026-10-17)
Backend Phase 13 is built on TEETHi-Backend branch `feat/timezone-contract` (`25a38b4`, `9253d74`), not merged yet. The contract is in §7.1 (slots) and §7.3 (booking, `AppointmentResource`). Fixes consistency report C1: today, a booking made from a device not set to UTC fails the slot check or lands in the wrong slot.
- [x] Booking (`lib/screens/booking/confirm_step.dart`): send the chosen slot's `starts_at` unchanged as `scheduled_at`. Delete the code that builds a `DateTime` from `start_time` plus the device's zone and then calls `toUtc()`.
  - Verified 2026-10-18 (TEETHi-Patient-App `eabcbd5`, branch `feat/timezone-contract`, not merged): `AvailabilitySlot` carries `startsAt`/`endsAt`/`timezone`; `scheduledAt` is a `String` from the view model down to `AppointmentsRemoteDataSource`, which sends it as-is (no `toUtc()` left). A unit test checks the request body holds `2026-11-02T09:00:00-08:00` byte for byte.
- [x] Any reschedule UI uses the same slot list and sends the slot's `starts_at`.
  - Verified 2026-10-18 (`eabcbd5`): Visits' `ReasonSheet` replaced its free date + time pickers with a date picker, then that doctor's `GET /patient/availability` slots for the visit's own `visit_type` (`BookingViewModel.fetchSlots`, which leaves the booking flow's state alone). It sends `slot.startsAt` unchanged; a unit test covers the `PATCH` body. A request with no doctor yet is no longer reschedulable (cancel only). Home's "Reschedule" button still starts a new booking, which uses the same slots.
- [x] Availability `date`: keep sending the calendar date the patient picked. It now explicitly means a date in the doctor's zone.
  - Verified 2026-10-18: unchanged, `BookingRemoteDataSource` sends the picked date as `YYYY-MM-DD`. The reschedule date picker still starts at the device's today, so a device a day ahead of the doctor can't pick the doctor's today.
- [x] Show visit times from `scheduled_at_local`, adding the zone abbreviation when `timezone` differs from the device's zone, never `scheduled_at.toLocal()`. Use `scheduled_at` (UTC) only for sorting and countdowns. This covers home (`home_screen.dart`), Visits (`visits_screen.dart`), the booking confirmation and record details.
  - Verified 2026-10-18 (`eabcbd5`): `Appointment` and `AppointmentSummary` parse `timezone`/`scheduled_at_local`; the new `lib/utils/visit_time.dart` (`VisitTime`) formats them. Used by the home hero and upcoming list, Visits cards, the reschedule sheet, past-visit summary, the booking confirmation (from the booked appointment) and the confirm step's "When" row. Clinical note, prescription and invoice details gained a "Visit" row. `scheduled_at` is only used for sorting. **Deviation:** the label is a GMT offset (`09:00 GMT-8`), not `PST`. The app has no tz database, and the device's IANA zone isn't readable without a plugin, so the check compares the visit's offset (from `scheduled_at` vs `scheduled_at_local`) with the device's offset at that instant. Adding the `timezone` package would give real abbreviations. `flutter analyze`: 38 infos, same as `master`; `flutter test` 23/23 (11 new).
- [x] Check: with the device set to UTC or a European zone, book the 09:00 slot of a Los Angeles dentist. It must show 09:00 in the patient app, the dentist app and the dashboard. **Verified live (2026-10-18):** on a UTC+3 (Syria) machine, the 09:00 Los Angeles slot showed as 09:00 GMT-7 in the patient app, 09:00 in the dentist app and 9:00 AM (America/Los_Angeles) in the dashboard, and was stored as 16:00Z. See `docs/CONSISTENCY_REPORT.md` §8. The booking itself had to be submitted through the API because the consent screen is mock (E1/C4).
  - Not run yet: needs a running backend and all three clients. Unit tests cover the patient side (`09:00 GMT-8` with a UTC device).

### Phase 10 — Polish & hardening
- [ ] Loading/empty/error states for every list screen, using the Phase 0 envelope handling consistently
- [ ] Confirm-before-destructive-action dialogs for every hard delete (dependents, insurance, payment methods — none of these are soft-deletable or undoable server-side)
- [ ] A real end-to-end pass once backend Phase 10 (HIPAA hardening) ships, specifically re-checking `POST /logout` behavior if a "log out all devices" endpoint is added (§15) and re-verifying error envelope shapes if `withExceptions` handling changes
- [ ] A real end-to-end pass once backend Phase 11 (real payment gateway) ships — re-verify the payment screens against actual gateway SDK tokens and real decline/error paths, neither of which exist to test against today

---

## 15. Open items to raise with the backend team

Discovered while writing this plan — not blockers to *starting*, but each will block the specific phase noted once reached:

Resolved as of 2026-09-15 — kept for history, not as active work:
- ~~`UserResource` doesn't expose `phone_number`/`phone_verified_at`~~ — resolved: both fields added to `UserResource` (§2).
- ~~Multi-select procedures vs. one `treatment_id` per appointment~~ — resolved: `POST /patient/appointments` now takes `treatment_ids: int[]` (§7.2, §7.3).
- ~~No "mark all notifications read" endpoint~~ — resolved: `PATCH /notifications/read-all` (§11.1).

Resolved as of 2026-09-16 — kept for history, not as active work:
- ~~Password reset was mail-link based, not code-based~~ — resolved: `POST /reset-password` now takes `{ email, code, password, password_confirmation }`, the same OTP UX as everything else (§2).
- ~~No `gender`/`blood`/`bio`/avatar-photo fields anywhere on `PatientProfile`~~ — resolved: `gender`/`blood_type`/`bio` added to `PatientProfileResource` (§3), and `POST`/`DELETE /patient/profile/photo` fill in `UserResource.profile_image` (§2, §3).

Open:
1. **No patient-facing file/report upload endpoint exists** (§8) — the app's "Reports" tab assumes one. Needs a product decision (drop the upload action vs. request a new backend feature) before Phase 7.
2. ~~**No insurance eligibility/coverage-check integration**~~ (§4) — resolved 2026-09-14 (product decision, not a backend gap): no eligibility API is planned; keep the client-side coverage estimate, explicitly labeled as a rough estimate, not a real quote.
3. ~~**No "support" conversation concept**~~ (§10) — resolved 2026-09-17 (product decision): drop it, no support thread will be built.
4. ~~**`POST`/`PATCH /patient/dependents` fails CORS preflight against the local dev backend**~~ — investigated 2026-09-14, not a backend bug: the route is registered (`patient.dependents.store`/`.update`), and both a direct `OPTIONS /patient/dependents` preflight and the actual `POST` (tried with both a valid and an invalid `relationship_to_account_holder`) return correct responses with proper `Access-Control-Allow-*` headers when the backend is actually running — confirmed live via curl, not just the test suite. The real cause of the original report was that the local Laravel dev server simply wasn't running at the time (confirmed separately: every local URL refused the connection outright). No client-side or server-side change needed here — just make sure `php artisan serve` (or your usual local vhost) is up before testing. Separately, `relationship_to_account_holder` validation was hardened from `Rule::in(array_map(...))` to `Rule::enum(PatientRelationship::class)` in both `StoreDependentRequest`/`UpdateDependentRequest` for consistency with every other enum field in the codebase — purely a cleanup, the old code already rejected invalid values with a normal 422 before this change.

5. ~~**Signed `/storage/{path}` URLs have no CORS headers, so Flutter web can't load them**~~ — fixed 2026-09-14: exactly as diagnosed, `config/cors.php`'s `paths` only listed `api/*`/`sanctum/csrf-cookie`, so the local disk's auto-registered `storage.local` route (`GET /storage/{path}`, from `serve => true` in `config/filesystems.php`) never got `Access-Control-Allow-Origin`. Added `storage/*` to `paths`; verified live (`curl` with an `Origin` header against a real signed URL, before/after) and with a new `tests/Feature/CorsTest.php` regression test that fails without the fix. This affects every signed-URL field the same way — avatars, insurance cards, legal documents, message attachments — all now covered by the same fix, no per-field change needed.

If a new gap turns up while building against this plan, add it here the same way, with a §-reference to where it affects the plan above.

---

## 16. `GET /patient/dashboard` — home screen composite read (backend Phase 12 addendum, 2026-09-16)

Added after this plan first shipped, specifically to remove the "compose 4-5 calls" workaround Phase 3 (§14) originally called for. One call returns everything the home screen needs:

```json
{
  "profile": { "id": 12, "first_name": "Amina", "missing_fields": [], "profile_completed_at": "...", "...": "..." },
  "consent_pending": false,
  "unread_notifications_count": 3,
  "upcoming_appointments": [ /* AppointmentResource[], up to 5, soonest scheduled_at first — §7.3 */ ],
  "quick_treatments": [ /* TreatmentResource[], is_quick_select only — §7.1 */ ]
}
```

- **`profile`** is the account holder's own `PatientProfileResource` (§3) — drives the "before your first visit" checklist off `missing_fields`/`profile_completed_at`, exactly as §3 already describes.
- **`consent_pending`** is a single boolean for the account holder's own profile only (not per-dependent) — `true` means the currently-active consent-form template hasn't been signed yet. For a per-dependent check (e.g. before booking for a specific family member), still use `GET /patient/consent-forms?patient_profile_id=` (§5); this flag is a coarse "does something need attention" home-screen signal, not a substitute for the real per-profile check the booking gate (§7.3) enforces.
- **`upcoming_appointments`** spans every profile the account manages (self + dependents), not just the account holder — up to 5, ordered soonest-`scheduled_at`-first. "Upcoming" here uses the same status rule as §7.3: anything not `completed`/`cancelled`/`declined`, including an unclaimed `open` request or still-`pending` direct one (both sort first, since their `scheduled_at` is `null`). The first item is the "next visit" hero card; the rest back the "upcoming treatments" list. This endpoint doesn't paginate or expose a "past visits" list — that's still `GET /patient/appointments` (§7.3), filtered client-side.
- **`quick_treatments`** is exactly `GET /treatments?quick_select=1` (§7.1) — included here so the home screen's bento-grid tiles don't need a second call.
- **`unread_notifications_count`** replaces the client-side "compute it from the paginated list" approach §11.1 originally described — use this number for the bell badge directly; still fetch `GET /notifications` (§11.1) only when the notifications screen itself is opened.
- Not paginated, not PHI-instrumented beyond what the underlying endpoints already are (no PHI is aggregated here that those endpoints don't already expose individually).

**Verified**: _(fill in once wired)_

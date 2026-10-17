# TEETHi — Cross-Project Status & Consistency Report

**Date:** 2026-10-17 · **Scope:** TEETHi-Backend, TEETHi-Frontend (admin dashboard), TEETHi-Patient-App, TEETHI-Dentist-App

**How this was checked**
- Ran each project's own test suite and static analysis on the current `master`.
- Exported the backend's live route table (`php artisan route:list`, 165 routes) and matched every HTTP call in the three clients against it.
- Matched every enum value, JSON key and permission slug the clients use against the backend's `app/Enums`, resources, requests and seeders.
- Traced the shared business rules end to end through the code: booking time slots, tooth numbering, pagination, password reset, notifications and consent. Plan text alone was not taken as proof.

---

## 1. Status at a glance

| Project | Plan progress | Build / tests (run 2026-10-17) | What's left |
|---|---|---|---|
| **Backend** (Laravel 13) | 92 ✔ / 11 open. Phases 0–9 and 12 done | ✅ **441 Pest tests pass**, PHPStan (max) clean | Phase 10 HIPAA close-out (6 items). Phase 11 real payment gateway (deferred) |
| **Admin dashboard** (Angular 21) | 53 ✔ / 7 open. Phases 0–11 done | ✅ `ng build` clean, **162/162 tests pass** | Phase 0 leftovers (429 handling, `/enums` cache). Optional Phase 12 (read-only messaging view). **Password reset is broken (§2 H1)** |
| **Dentist app** (Flutter) | 45 ✔ / 8 open. Phases 0–10 built | ✅ **80/80 tests pass**. `flutter analyze`: 49 infos, 0 warnings/errors | Phase 11 polish and an end-to-end run. **Plan says "not yet verified against a live backend"** |
| **Patient app** (Flutter) | 41 ✔ / 12 open. Phases 0–8 built | ⚠️ **1/1 test fails** (stale smoke test, see §4). `flutter analyze`: 38 infos, 0 warnings/errors | Phase 2b (insurance and payment-method management), Phase 9 (notifications, push), Phase 10 (polish). **Several mock-only screens remain (§2)** |

**Bottom line:** the backend is the most mature piece. It is fully tested, and all three clients only call endpoints that exist. The gaps are in **how the clients use the backend**. The biggest problems are an undefined timezone contract for bookings and patient-app screens that still show mock data the dentist and admin will never see.

---

## 2. Critical: breaks what a party needs

### C1. Booking time slots have no timezone contract (Backend ↔ Patient app ↔ Dentist app)
- **Backend:** `config/app.php` uses `timezone = UTC`. Doctor availability is stored as wall-clock `HH:MM:SS` with no zone. `AvailabilityService::isSlotOpen()` checks that `scheduled_at->format('H:i:s')` exactly matches a slot's `start_time` (`app/Services/AvailabilityService.php:78`).
- **Patient app:** the app builds the booking time as a *device-local* `DateTime` from the slot's `"09:00"`, then sends `scheduledAt.toUtc().toIso8601String()` (`lib/screens/booking/confirm_step.dart:46`, `appointments_remote_data_source.dart:42`).
- **Result:** on any device not set to UTC (the seed data is Los Angeles, UTC−7), the 09:00 slot reaches the backend as `16:00:00`. The backend then either rejects it with `422 slot_not_available` or **books a different slot** if 16:00 also exists. The weekday can also shift near midnight. The dentist app's reschedule has the same problem (`requests_repository_impl.dart:50`). The dentist and admin apps show times with `toLocal()` / the Angular `date` pipe, so the three parties can each see a different time for the same visit.
- **None of the four plans defines a timezone.** **Decision needed:** pick one model and write it into IMPLEMENTATION_PLAN §2. Options:
  - Slots are in the doctor's or city's local zone, a `timezone` column is added to the doctor or city, and `scheduled_at` is converted before matching.
  - Clients send the slot's literal date and time with an explicit offset.
- Backend tests didn't catch this because they build `scheduled_at` with server-side Carbon (UTC), so the wall-clock time and the UTC time are always the same there.

### C2. Patient medical history is hardcoded. Dentists never get real allergies or medications
- Profile → Medical tab shows fixed values for every patient (`Allergies: Penicillin`, `Medication: None`, `lib/features/profile/presentation/screens/profile_screen.dart:339-341`).
- The patient app never calls `GET` or `PUT /patient/medical-history`.
- The dentist app *does* read `medical_history` for each patient, so dentists will see empty data while patients see made-up data. This is a clinical-safety issue.
- The patient plan documents the endpoints (§8) but **has no phase checkbox for wiring them**, so the work isn't tracked anywhere.

### C3. Every patient-app list is silently capped at 15 items
> **Fixed (2026-10-17):** merged into TEETHi-Patient-App `master` (`82a0509`). It also fixes M8. Not yet exercised against a live backend.

- The backend paginates every list endpoint (default `per_page=15`, decision #6).
- The patient app never sends `page` or `per_page`; it reads only `data` from the first page (plan Phase 0 item "pagination helper — not started"). Effects:
  - **Booking catalog:** `GET /treatments` returns 15 of the **21 seeded treatments**, so 6 can't be booked.
  - **Chat:** `GET /conversations/{id}/messages` is ordered oldest-first, so a thread with more than 15 messages opens on the *oldest* 15 and the newest are missing.
  - Visits, dentist directory, records and conversations are also cut off after 15.
- The dentist app (`core/network/paginated.dart`) and the admin dashboard (`<app-pagination-bar>`) handle pagination correctly.

### C4. Patient consent forms are mock-only, so nothing is signed on the server
- `lib/screens/consent_screen.dart` reads `MockData.consentForms` and signs into the local `AppState`.
- `GET /patient/consent-forms` and `POST …/sign` are never called.
- Consent is a legal record the backend stores, the admin reviews and the plan treats as PHI. As things stand, no patient consent ever reaches the server.
- The patient plan documents this in §5 but **has no phase checkbox** for it.

---

## 3. High: wrong behaviour or contract drift

### H1. Admin dashboard password reset is broken (contract drift)
> **Fixed (2026-10-17):** merged into TEETHi-Frontend `master` (`b2882f6`). Not yet exercised against a live backend.

- On 2026-09-13 the backend switched to OTP codes: `POST /reset-password` requires `{ email, code (6 digits), … }` (`ResetPasswordRequest.php:29`).
- The dashboard still sends `token` read from a `?token=` link (`views/pages/reset-password/reset-password.component.ts:66`). Every admin reset attempt will fail with `422`.
- Both mobile apps were updated and send `code`.
- **Root cause:** the change was recorded only in the backend's copy of the admin plan, so the frontend's copy never received it (see H2).

### H2. The two copies of the admin plan had diverged
- `TEETHi-Backend/docs/` had the newer API contract (OTP reset; patient email/phone fields on the admin directory) but showed only 1 of 62 items done.
- `TEETHi-Frontend/docs/` had all the progress (53 done) but the old contract.
- The consolidated copy in `docs/plans/` merges both, and it is now the only copy (§6, decision 2).

### H3. Patient bookings can't be made for family members
- Family and dependent profiles are a locked v1 decision (#3), and the backend supports `patient_profile_id` on bookings, consent, insurance and records.
- The patient booking flow has no step for choosing a profile, so every booking goes to the account holder (`appointments_remote_data_source.dart:23`, a known simplification). Dependents can be created but can't be booked for.

### H4. Push notifications aren't delivered on either mobile app
- The backend sends notifications through `database` + `FcmChannel`.
- The dentist app registers a token through `NoopPushTokenSource`: the plumbing is there, but there's no Firebase SDK or config.
- The patient app has no device-token registration and **no in-app notification feed**: `notifications_screen.dart` is still mock (Phase 9).
- Patients therefore never see appointment-status changes or reminders, while dentists only see them in-app.

### H5. Patient payment and insurance screens still show placeholder data
- The payment step always shows `CareCredit card · Ending 4417` and sends a `mock-token-<timestamp>` (`lib/screens/booking/payment_step.dart:23-59`, `record_detail_screen.dart:111-221`).
- Saved methods (`/patient/payment-methods`) are never listed.
- "CareCredit" is a real healthcare-credit brand used here as a label for `credit_card`. That's misleading.
- Insurance can be added inside booking, but there's no screen to list, edit or delete it (Phase 2b open).
- The gateway is a stub by design (Phase 11), but the UI shouldn't show a fake card.

### H6. Shared routes bypass dashboard permissions (backend design point to confirm)
- `/appointments/{id}` (view, **cancel**, **reschedule**) and `/conversations*` are allowed for *any* `role=admin` user (`AppointmentPolicy.php:17-27`). No `permission:` check applies.
- So a staff member with only a billing role can read appointment details (PHI), read every patient–doctor message thread, and cancel or reschedule visits through the API, even though the dashboard UI doesn't offer that.
- This conflicts with the spirit of decision #5 (dashboard RBAC) and with "minimum necessary" access.
- **Decision needed:** gate these routes for admins behind `view_appointments` and a new `manage_appointments` / `view_messages` permission, or document the exception explicitly.

### H7. "Forgot password" reveals whether an account exists
- `ForgotPasswordRequest` uses `exists:users,email`, so an unknown email returns `422`.
- The dashboard's neutral "if an account exists…" message is undone because it shows that `422` inline. The `422` comes from the backend, so it affects every client.
- Fix on the backend: always return 200.

---

## 4. Medium: differences between the projects

| # | Area | Finding |
|---|---|---|
| M1 | **Localization** | Backend (`lang/en`, `lang/ar`) and the admin dashboard (`en.json`, `ar.json`) support Arabic. Both mobile apps are English-only: no `flutter_localizations`, and `ApiClient.locale` is hardcoded to `'en'`. Decide whether patients and dentists need Arabic in v1. |
| M2 | **Doctor photos** | There is no doctor photo upload endpoint and `ProviderResource` has no image, so the patient's dentist directory can only show placeholders. Doctors also never see patient photos (dentist plan §16, by design). Worth confirming with product, since a marketplace usually shows provider photos. |
| M3 | **`GET /enums` usage** | Every backend enum is exposed through `/enums`, and the scan found **no invalid wire values** in any client. The admin dashboard still hardcodes labels (Phase 0 open), and the patient app uses `/enums` only in places. Labels can therefore differ between apps, especially in Arabic. |
| M4 | **Error envelope** | 404, 405 and 500 still return Laravel's debug format, not `{success, message, data}` (backend Phase 10 open). Each client handles this ad hoc. Settle it before production. |
| M5 | **429 rate-limit handling** | The patient app has `RateLimitFailure`. The admin dashboard has no 429 handling (Phase 0 open). |
| M6 | **Admin coverage vs. scope** | The project scope (`AGENTS.md`) promises list **and detail** views for appointments. The dashboard has only an appointments list. The backend already allows admins to `GET /appointments/{id}`, so a detail view could be built today. The dashboard home page is a placeholder welcome card, and there is no KPI or summary endpoint. |
| M7 | **Leftover patient-app mock code** | `lib/state/app_state.dart`, `lib/data/mock_data.dart` and `lib/models/models.dart` are still used by the booking steps, consent, notifications and parts of profile. The dentist app has removed all of its mocks. |
| M8 | **Patient-app test is broken** | `test/widget_test.dart` builds the app without `configureDependencies()`, so it fails with `GetIt: AuthViewModel is not registered`. The app has no other tests; the dentist app has 80. |
| M9 | **Dentist-app documentation is stale** | `docs/ARCHITECTURE.md` still says only auth is real and everything else is mock. The dentist plan still lists "Remove `MockDentistData`" as open, but no mock files remain. |
| M10 | **Dentist-app git history** | All of Phases 0–10 is in **one commit**, so its plan can't be audited through `git log` the way the other repos can. |
| M11 | **Lint debt** | Flutter `analyze` reports info-level issues only: 30 `withOpacity` deprecations, 27 `prefer_initializing_formals`, 18 `prefer_const`, and others. Neither app has warnings or errors. |
| M12 | **Housekeeping** | `TEETHi-Backend/.claude/worktrees/mystifying-rhodes-2e8412`: its branch has nothing that isn't on `master`. `TEETHi-Patient-App/hs_err_pid*.log` and `flutter_01.log` are crash logs in the repo root. Old CoreUI demo views (`views/base`, `buttons`, `charts`, `forms`, `icons`, `theme`, `widgets`) are still in the dashboard but not routed. |

---

## 5. Verified consistent ✅

- **Endpoints:** every call in all three clients maps to an existing backend route and method (patient 42, dentist 52 + 9 via helpers, admin ≈40).
- **Enum wire values:** all values used match `app/Enums`. Notification `type`s in the dentist inbox match the backend's 5 notification classes exactly.
- **Tooth numbering:** Universal 1–32 everywhere. Backend validates `between:1,32`. The patient chart maps to Universal (`tooth_chart.dart:20`). The dentist app converts Universal ↔ quadrant labels (`core/utils/tooth_numbering.dart`).
- **Dashboard permission slugs:** every slug the dashboard checks exists in the backend seeder (`manage_*`, `view_*`, `verify_legal_documents`). The only exception is a commented `view_my_feature` example.
- **Password rules:** minimum 8 characters, client-side, in all three clients, matching `Password::defaults()`. Mobile reset uses the 6-digit `code`.
- **Cancel and reschedule:** `reason` is required by the backend and sent by both mobile apps. The dentist app enforces "up to 3" recommendations, matching the backend's rank 1–3.
- **Transport:** all three clients default to `http://127.0.0.1:8000/api/v1`, send `Authorization: Bearer` and `Accept-Language`, and clear the session on `401`.
- **Money:** amounts are decimal strings on the wire, and the dashboard explicitly avoids converting them to numbers.

---

## 6. Decisions needed to keep the four projects aligned

1. **Timezone model for availability and `scheduled_at` (C1).** This blocks reliable booking. Add the decision to IMPLEMENTATION_PLAN §2 and update both apps.
2. ~~**Single source of truth for the plans.**~~ **Decided 2026-10-17: option (a).** The root `docs/` is canonical and the workspace root is now a docs-only git repo. Every per-repo copy has been replaced by a pointer, and the backend and frontend `AGENTS.md` point here. See `docs/README.md` for the rules.
3. **Admin access to shared PHI routes (H6).** Add permissions, or document the exception.
4. **Patient-plan gaps:** add phase checkboxes for **medical history (C2)** and **consent forms (C4)**, and a **dependent picker in booking (H3)**.
5. **Arabic on mobile (M1)** and **doctor photos (M2)**: product decisions.
6. **Push notifications (H4):** set up a Firebase project and add the config to both apps. One shared setup covers both.

### Suggested order of work
1. C1 timezone contract (backend first, then both apps).
2. H1 admin reset (`token` → `code`, with a code-entry field instead of reading the link).
3. C3 patient pagination helper.
4. C2 medical history, then C4 consent wiring in the patient app, then H3 dependent picker.
5. Backend Phase 10, including H7 and M4.
6. Patient Phase 9 and dentist Phase 11, including H4 push.
7. An end-to-end run across all three apps against one live backend. **This has not been done for either mobile app.**

---

## 7. Docs consolidated in this folder

See `docs/README.md` for the index and where each file came from.

# TEETHi — Cross-Project Status & Consistency Report

**Last updated:** 2026-10-18, after fixing E11 and E12 (first written 2026-10-17) · **Scope:** TEETHi-Backend, TEETHi-Frontend (admin dashboard), TEETHi-Patient-App, TEETHI-Dentist-App

**How this was checked**
- **Static review (2026-10-17):**
  - Exported the backend's route table (165 routes) and matched every HTTP call in the three clients against it.
  - Matched every enum value, JSON key and permission slug the clients use against the backend's enums, resources, requests and seeders.
  - Traced the shared business rules through the code.
- **Live end-to-end runs (2026-10-18), §7:** backend, patient app, dentist app and dashboard all running against one MySQL database, on a UTC+3 machine with Los Angeles dentists.
- **Test suites:** every suite was re-run on each repo's merged `master` after the last fixes.

IDs are stable across updates: C = critical, H = high, M = medium (from the static review), E = found in the live run. Fixed items move to §3 and keep their ID.

---

## 1. Status at a glance

| Project | Plan progress | Build / tests on `master` (2026-10-18) | What's left |
|---|---|---|---|
| **Backend** (Laravel 13) | Phases 0–9 and 12–14 done | ✅ **497 Pest tests pass** (500 on `fix/e11-e12`), PHPStan (max) and Pint clean | Phase 10 HIPAA close-out (6 items). Phase 11 real payment gateway (deferred). Open: H6, H7, M4 |
| **Admin dashboard** (Angular 21) | Phases 0–11 done, plus the timezone and currency/name sections | ✅ `ng build` clean, **189/189 tests pass** | Phase 0 leftovers (429 handling, `/enums` cache). Optional Phase 12 (messaging view). Open: M6 (no appointment detail, placeholder home page) |
| **Dentist app** (Flutter) | Phases 0–10 plus the timezone, reschedule and display-name sections | ✅ **100/100 tests pass**. `flutter analyze`: 49 infos, 0 warnings/errors | Phase 11 polish. Open: H4 (push), M4/E7 (generic error text), M9 |
| **Patient app** (Flutter) | Phases 0–8 plus timezone, consent, reschedule, medical history, currency and name | ✅ **41/41 tests pass**. `flutter analyze`: 38 infos, 0 warnings/errors | Phase 2b (insurance and payment-method screens, H5), Phase 9 (notifications, push, H4), Phase 10. Open: E9. E11/E12 fixed on `fix/e11-e12` (45 tests) |

**Bottom line:**
- **The core visit lifecycle now works end to end and is consistent in all three apps.** A patient books (consent is signed in the app), the dentist accepts and reschedules (both directions, rule A), then starts, documents and completes the visit. The patient receives the report, and the admin sees the appointment and the invoice. Every app shows the same visit time in the visit's own timezone.
- **What's left, in order:**
  - **Patient-app mock screens:** H4, H5.
  - **Backend hardening:** Phase 10, H6, H7, M4.

---

## 2. Open issues

### Critical
None open. C2 (mock medical history) is fixed; see §3.

### High
| # | Projects | Issue |
|---|---|---|
| H4 | Patient, Dentist | **Push notifications aren't delivered.** The backend sends notifications through `database` and `FcmChannel`. The dentist app's `NoopPushTokenSource` has no Firebase SDK or config behind it. The patient app has no device-token registration, and its notification feed and bell badge are still mock: the live run showed **3 while the server had 5 unread**. Patients never see status changes; a rescheduled visit only updated after a manual reload. |
| H5 | Patient | **Payment and insurance screens show placeholders.** The payment step always shows "CareCredit card · Ending 4417" and sends `mock-token-<timestamp>`. "CareCredit" is a real brand. Saved payment methods are never listed. There's no insurance management screen (Phase 2b). |
| H6 | Backend, Dashboard | **Shared routes bypass dashboard permissions.** `/appointments/{id}` (view, cancel, reschedule) and `/conversations*` are open to *any* `role=admin` user, so a billing-only staff member can read PHI and move visits through the API. **Decision needed:** gate these routes with permissions, or document the exception. |
| H7 | Backend | **"Forgot password" reveals whether an account exists.** `ForgotPasswordRequest` uses `exists:users,email`, so an unknown email gets a `422`. Always return `200`. |

### Medium
| # | Projects | Issue |
|---|---|---|
| M4 / E7 | Backend, all apps | **Error envelope.** `404`, `405`, `409` and `500` return Laravel's debug format, not `{success, message, data}` (backend Phase 10). Confirmed live: the dentist app showed "Something went wrong" instead of the server's `409` message. |
| M1 | Mobile apps | **No Arabic on mobile.** The backend and the dashboard support Arabic; both mobile apps are English-only, with `ApiClient.locale` hardcoded to `'en'`. This is a product decision. |
| M2 | Backend, Patient | **No doctor photos.** There's no doctor photo upload and no image on `ProviderResource`, so the dentist directory shows initials only. This is a product decision. |
| M3 | Dashboard, Patient | **Hardcoded labels.** `GET /enums` exists, but the dashboard (Phase 0) and parts of the patient app hardcode labels, so wording can differ between apps, especially in Arabic. |
| M5 | Dashboard | **No 429 handling.** The dashboard has no rate-limit (`429`) surface. |
| M6 | Dashboard | **Missing appointment detail.** The scope promises list **and detail** views for appointments; there's only a list. The backend already serves `GET /appointments/{id}` to admins. The dashboard home page is still a placeholder. |
| M7 | Patient | **Leftover mock state.** `lib/state/app_state.dart`, `lib/data/mock_data.dart` and `lib/models/models.dart` are still referenced by 29 files: the booking draft, notifications, the payment options and parts of profile. Consent no longer uses them. |
| M9 | Docs | **Dentist architecture doc is out of date.** `docs/architecture/DENTIST_APP_ARCHITECTURE.md` still says only auth is real and everything else is mock. |
| M10 | Dentist | **One-commit history.** Phases 0–10 of the dentist app are a single commit, so its plan can't be audited through git history. |

### Low
| # | Projects | Issue |
|---|---|---|
| E8 | Patient, Dentist | **Relative labels use the device's date, not the visit's zone.** The patient sees "Tomorrow" for a visit two days away in Los Angeles; the dentist header shows the device's date. (The dentist's "Today" tag on an in-progress visit is fixed; it now reads "In progress".) |
| E9 | Patient | **Booking requires browser geolocation** to confirm the address, with no fallback. |
| E10 | Patient, Dashboard | **Small UI issues:** "Use profile" in the visit-address step silently does nothing when the profile has no address. The dashboard's Cities column is labelled "State/Emirate". The patient's "Nearby" dentist tab was empty while "All" listed the dentist (cause not investigated). The consent list briefly shows the previous list before it reloads. |
| M11 | Mobile apps | **Lint debt.** Info-level only: `withOpacity` deprecations, `prefer_initializing_formals`, `prefer_const`. |
| M12 | All | **Housekeeping.** Merged feature branches (`fix/*`, `feat/*` from 2026-10-17/18) can be deleted. `TEETHi-Backend/.claude/worktrees/mystifying-rhodes-2e8412` is a stale worktree. Crash logs (`hs_err_pid*.log`, `flutter_01.log`) sit in the patient app's root. Unused CoreUI demo views remain in the dashboard. |

---

## 3. Resolved

| # | Was | Fix | Where (all merged to `master`) | Verified |
|---|---|---|---|---|
| C1 | Booking slots had no timezone contract: bookings from non-UTC devices failed or landed in the wrong slot | Decision #7: one IANA timezone per city; exact `starts_at` instants on the wire; `scheduled_at_local` for display; doctor open-slots endpoint | Backend Phase 13 (`25a38b4`, `9253d74`, `c8a8092`). Patient app `eabcbd5`, dentist app `1487519`, dashboard `9af49ca` | Live: a UTC+3 machine booked 09:00 Los Angeles time, and all three apps showed 09:00. The old payload is rejected |
| C3 | Every patient-app list stopped at 15 items | Pagination helper (`getAll` / `getPage` / `getLatest`) | Patient app `82a0509` | Live: `treatments?per_page=100` and similar |
| C4 / E1 | Consent screen was mock and no consent template was seeded, so **no patient could book from the app** | Seeder publishes a patient consent form (plus the doctor agreements). The consent screen is wired to `GET` / `POST …/sign` | Backend `ae17a8f`, patient app `9ec2fa4` | Live: an admin published v2, the app showed it as needing a signature, signing returned `201`, and `consent_pending` became false |
| E2 | A rescheduled visit could never be started | Decision #8 (rule A): a doctor's reschedule goes straight back to `accepted`; a patient's waits in the dentist's Pending list for Accept new time / Decline (decline cancels). Only `accepted` visits can be moved | Backend `8d9b4b9`, dentist app `c59e68a`, patient app `924abf8` | Live in all three apps, full status history checked |
| H1 | Admin password reset sent `token`; the backend wants `code` | OTP code entry | Dashboard `b2882f6` | Live: code → `200` → login |
| H2 | Two copies of the admin plan had diverged | Root `docs/` is the only copy; per-repo files are pointers | Root docs repo | — |
| H3 | Bookings couldn't be made for family members | "Who is this visit for?" in Before we book; checks, consent and booking use the chosen profile's `patient_profile_id` | Patient app `badb8a9` | Live: booked for child "Mia Patient" and signed her consent; the dentist sees "Mia Patient" |
| E4 | No way to add a phone number while booking | "No number on file" → Add → number field → Send code → code entry. The backend now validates the phone format (`5ee82df`) | Patient app `badb8a9` | Live: added and verified a phone inside the step. The earlier "code boxes don't accept input" note was a test-tool limitation; key presses work |
| E11 | The provider starting fee had no currency ("$80" hardcoded) | `ProviderResource.currency`; the dentist step and the dentist profile use `formatMoney()`. The last hardcoded `$` amounts (payment step, invoice detail, visits list) were also replaced. The branch step's fixed "From $95 / $165" copy remains | Backend `efb65c2`, patient app `c22338a` (*branches `fix/e11-e12`, not merged*) | Live API: providers return `currency: USD`. Tests cover the parsing and the fallback |
| E12 | A parent's insurance could be attached to a child's visit | The insurance step shows the shared "Who is this visit for?" picker and lists only that person's policies. Switching the person drops the chosen policy. The backend rejects a policy that isn't the booked person's own | Backend `efb65c2`, patient app `c22338a` (*branches `fix/e11-e12`, not merged*) | Live API: booking Mia with the parent's policy → `422 insurance_id`. Three new backend tests. In the app: each person sees only their own policies; switching to Mia in Before we book shows "Choose a plan for Mia", which returns to the insurance step |
| M8 | Patient smoke test failed | Test now registers dependencies | Patient app `82a0509` | Suite passes |
| C2 | Medical history was hardcoded ("Penicillin" for everyone), so dentists never got real allergies | Medical tab wired to `GET/PUT /patient/medical-history`, with an edit sheet | Patient app `64a2539` | Live: the patient saved Latex / Metformin 500mg, and the dentist's record showed the same |
| E3 | The dashboard showed AED while the patient app showed $ | Decision #9: the backend sends `currency` (USD). Dashboard `money` pipe; patient app `formatMoney()` | Backend `7860aee`, dashboard `3df5016`, patient app `3d2a4b1` | Live: dashboard invoices "$250.00 / $100.00" |
| E5 | The dentist and admin saw "Patient" / "#1" | Decision #10: profile names pre-filled from the account name and backfilled; `full_name` used in every app; booking requires a name | Backend `422ceb4` / `7860aee`, dashboard `3df5016`, dentist app `b14a731`, patient app `3d2a4b1` | Live: "Sample Patient" in the dentist Pending list, the Patients tab and the dashboard Appointments list |
| E6 | No invoice existed until the patient paid | Decision #11: invoice created at booking (direct requests) or when a dentist claims an open request; an unpaid invoice is cancelled with its visit | Backend `422ceb4` | Live: new booking #3's $250 invoice existed immediately; tests cover the claim, cancel and partly-paid cases |

---

## 4. Verified consistent ✅

- **Endpoints:** every client call maps to an existing backend route and method.
- **Enum wire values:** every value matches `app/Enums`. Dentist inbox notification types match the backend's notification classes.
- **Tooth numbering:** Universal 1–32 everywhere.
- **Dashboard permission slugs:** every slug the dashboard checks exists in the backend seeder.
- **Password rules:** minimum 8 characters, with 6-digit OTP reset codes, in all three clients.
- **Cancel and reschedule:** a reason is required and sent. The "up to 3" recommendations limit matches.
- **Transport:** all clients send Bearer auth and `Accept-Language`, and clear the session on `401`.
- **Money:** amounts are decimal strings on the wire. Currency *display* differs (E3).
- **Live (2026-10-18):**
  - Login in all three apps.
  - Visit time agrees in all three apps, in the visit's own zone.
  - Consent signing and versioning.
  - Rule A reschedule flow, in both directions.
  - Clinical note → report reaches the patient.
  - Status notifications are created.
  - Invoices appear in the dashboard.
  - Admin OTP reset.

---

## 5. Decisions

**Locked** (recorded in `IMPLEMENTATION_PLAN.md` §2 or `docs/README.md`):
- **Single source of truth for plans:** the root `docs/` (2026-10-17).
- **#7 Timezones:** one timezone per city, with exact instants on the wire (2026-10-17).
- **#8 Rescheduled visits:** a doctor's reschedule settles back to accepted; a patient's needs the doctor's acceptance (2026-10-18).
- **#9 Currency:** one platform currency (USD), sent by the backend as `currency` (2026-10-18).
- **#10 Patient display name:** the profile's first/last name, pre-filled from the account name; `full_name` everywhere; required to book (2026-10-18).
- **#11 Invoice timing:** at booking for direct requests, on claim for open requests; cancelled with an unpaid visit (2026-10-18).

**Still needed:**
1. **H6:** admin access to the shared PHI routes. Add permissions, or document the exception.
2. **M1 / M2:** Arabic on mobile, and doctor photos (product decisions).
3. **H4:** set up one Firebase project for both apps.
4. **Product note:** the backend allows one active consent form per template type, while the old mock had four patient forms. Decide whether one combined form is enough.

### Suggested order of work
1. **Backend Phase 10,** including H7, M4 and H6.
2. **H4 / H5:** patient Phase 9 (notifications and push) and Phase 2b (insurance and payment methods). Dentist Phase 11.
3. **Merge** `fix/e11-e12` (backend and patient app) and run the suites on `master`.
4. **Re-run the live end-to-end scenario** (§7) after each group.

---

### Merge check (2026-10-18, C2 and decisions #9–#11)
`feat/decisions-9-11` was fast-forwarded into `master` in all four repos (it includes `fix/medical-history` in the patient app). Commits: backend `7860aee`, dashboard `3df5016`, patient app `3d2a4b1`, dentist app `b14a731`.
- Backend: 491 Pest tests pass; PHPStan and Pint are clean.
- Dashboard: `ng build` is clean and 189/189 tests pass.
- Patient app: 36/36 tests pass, 0 analyzer warnings or errors.
- Dentist app: 100/100 tests pass, 0 analyzer warnings or errors.

### Merge check (2026-10-18, H3 and E4)
Fast-forwarded: TEETHi-Patient-App `master` → `badb8a9` (`feat/booking-gaps`); TEETHi-Backend `master` → `5ee82df` (`fix/phone-format`).
- Backend: 497 Pest tests pass; PHPStan and Pint are clean.
- Patient app: 41/41 tests pass, 0 analyzer warnings or errors.
- The dashboard and the dentist app are unchanged since the last merge check (189/189 and 100/100).

## 6. Docs consolidated in this folder

See `docs/README.md` for the index and the rules for keeping docs here.

---

## 7. Live end-to-end runs (2026-10-18)

**Setup:**
- Backend `php artisan serve` on :8000 against the local MySQL `teethi` database.
- Patient app (:8917) and dentist app (:8765) as Flutter web debug builds; dashboard (`ng serve`, :4200).
- The machine's zone is Syria (UTC+3); every dentist is in Los Angeles (UTC−7).
- Seeded accounts: `patient@`, `dr.michael@` and `admin@teethi.test`. Run configs are in `.claude/launch.json` at the root.

**Run 1, the full visit lifecycle:**
1. The patient books Mon 19 Oct 09:00. This needed a consent template published from the dashboard and a sign through the API; that blocker was E1.
2. The dentist accepts, then reschedules to 14:00 through the open-slot picker.
3. The visit couldn't be started (E2) until it was re-accepted through the API.
4. The dentist documents and completes the visit.
5. The patient sees the visit and the report.
6. The dashboard checks pass: Appointments, Cities, Invoices, and an OTP password reset.

**Run 2, after the E1/E2 fixes:**
1. Consent v2 was published and then re-signed in the patient app.
2. Visit #2 was booked for Tue 20 Oct 10:00.
3. The dentist accepted it.
4. The patient moved it to 12:00. The dentist's Pending tab showed "New time" with the reason; the patient saw "New time · awaiting dentist".
5. The dentist chose Accept new time.
6. The dentist's own move to 15:00 stayed Accepted, and the patient saw "Confirmed" after a reload.

**Run 5, E11 and E12 (2026-10-18):**
1. `GET /patient/providers` returned `currency: USD` beside `starting_fee`.
2. Policy #1 (Delta Dental) was added to the seeded patient and policy #2 (Cigna Kids) to Mia (#10).
3. Booking Mia with policy #1 through the API returned `422` on `insurance_id`.
4. In the app's routine flow:
   - The insurance step listed only the chosen person's policy.
   - Switching to Mia in Before we book showed "Insurance · Choose a plan for Mia". **Choose** led back to the insurance step.
   - With Mia's plan chosen, every check passed.
   - No booking was placed.

**Run 4, H3 and E4 (2026-10-18):**
1. The seeded patient's phone was cleared, and child "Mia Patient" (no date of birth) was added.
2. In the app's booking flow, Mia was chosen. Her date of birth was added in the step, a phone was added and verified, and her consent was signed.
3. Booking #4 was created for her. The dentist's Pending list shows "Mia Patient", and invoice #4 exists.
4. The phone was saved doubled by the test tool's typing, which led to the new backend format rule. The seeded patient's phone was then reset to `+13105550142`.

**Run 3, after C2 and decisions #9–#11:**
1. The patient filled in their medical history, and the dentist saw it.
2. Booking #3 (Wed 21 Oct 10:00) returned `currency: USD` and `full_name`, and its invoice existed immediately.
3. The dashboard showed $ amounts and "Sample Patient" everywhere.

**Test data left in the local database:**
- Appointment #1 (completed, Invoice #1 pending pay-later), appointment #2 (accepted, Tue 20 Oct 15:00, no invoice because it was booked before decision #11), and appointment #3 (pending, Wed 21 Oct 10:00, Invoice #2 $250 pending).
- Insurance policies #1 (Delta Dental, seeded patient) and #2 (Cigna Kids, Mia).
- Dependent "Mia Patient" (child, profile #10, date of birth 1996-10-08 from the test picker) with consent v2 signed, and her booking #4 (pending, Thu 22 Oct 11:00, invoice $100). The visit address street was doubled by the test tool.
- The seeded patient's medical history: allergies "Latex", current medications "Metformin 500mg".
- Migrations `2026_10_18_000002_backfill_patient_profile_names` applied; the seeded patient's profile is now "Sample" / "Patient".
- Consent templates: patient consent v1 (superseded) and v2 (active), signed by the seeded patient; plus the seeded doctor business agreement and nondiscrimination notice.
- The seeded patient has a date of birth (1996-10-18) and a verified phone (+13105550142).
- The admin password was reset to the same seed value.

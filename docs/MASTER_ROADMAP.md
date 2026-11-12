# TEETHi — Master Roadmap (pending work, decisions, phases)

**Written:** 2026-11-11, from a full read of every file in `docs/` plus a check of each repo's git state.
**Purpose:** one place that lists everything still open across the four projects, sorted into categories and then into phases you can work through in order. The detailed specs stay in the source plans; each item here points to its source.

**How to use it**
- Work the phases in §5 top to bottom. Items inside a phase can run in parallel unless marked otherwise.
- When an item lands, tick it here **and** in the source plan (the source plan keeps the "Verified" note, per `docs/README.md` rule 2).
- Repo tags: **BE** backend, **DB** admin dashboard, **PA** patient app, **DA** dentist app, **DOCS** root docs repo, **OPS** accounts/hosting/devices (owner action, no code).
- Source abbreviations: **IMP** `plans/IMPLEMENTATION_PLAN.md`, **PAY** `plans/PAYMENTS_ROLLOUT_PLAN.md`, **TELE** `plans/TELEDENTISTRY_PLAN.md`, **PAP** `plans/PATIENT_APP_FRONTEND_PLAN.md`, **DAP** `plans/DENTIST_APP_FRONTEND_PLAN.md`, **ADP** `plans/ADMIN_DASHBOARD_FRONTEND_PLAN.md`, **CR** `CONSISTENCY_REPORT.md`, **E2E** `E2E_REPORT_2026-10-23.md`, **SEC** `PAYMENTS_SECURITY_FINDINGS.md`.

---

## 1. State of the project (2026-11-11)

### 1.1 What's done
- **Core visit lifecycle works end to end in all four apps:** sign-up and verification, dentist onboarding and vetting (bookable gate), booking (home, emergency open pool, teledentistry), consent, insurance, accept / reschedule (rule A) / start / document / complete, reports, prescriptions, recommendations, reviews, invoices, messaging, push (web verified), account deletion, app settings (maintenance / forced update).
- **Backend:** plan phases 0–15, 17, 18, the photo and account-deletion addenda, Stripe Payment Sheet (Phase 11), teledentistry T1–T2, payments P1–P3, and security fixes S1–S7 are merged. Only **Phase 16 (Google/Apple sign-in)** is unbuilt.
- **Dashboard:** phases 0–11 (with Phase 0 leftovers), timezone, currency, app settings, teledentistry T6, payments P1–P3 and S5/S7 merged. Optional Phase 12 (messaging viewer) not built.
- **Patient app:** phases 0–10 (Phase 2b and Phase 10's device Stripe pass open), teledentistry T3–T4, payments P2–P3, Arabic.
- **Dentist app:** phases 0–11, teledentistry T5, payments P3, Arabic. Phase 12 (social sign-in) not built.
- **E2E run (2026-10-23):** all of R8-01 to R8-26 are fixed except R8-25 (dentist payouts, which is payments P7) and two R8-01 client leftovers.

### 1.2 Repos right now

| Repo | `master` | Tests at last merge | Remote | Ahead of remote | Working tree |
|---|---|---|---|---|---|
| Root docs | `7c99eb5` | — | GitHub `nbarazi/TEETHi` | **72 commits not pushed** | `.claude/launch.json` modified (+63 lines) |
| TEETHi-Backend | `0867191` | 853+ Pest, PHPStan max, Pint | GitLab `teethi1/teethi-backend` | **39** | clean; stale prunable worktree `claude/mystifying-rhodes-2e8412`; worktree `D:/Projects/TEETHi-worktrees/backend-p1` |
| TEETHi-Frontend | `7ca9a54` | 215 Vitest, `ng build` clean | GitLab `teethi1/teethi-frontend` | **12** | untracked `CLAUDE.md`; worktree `dashboard-p1` |
| TEETHi-Patient-App | `79906e7` | 214 Flutter | GitLab `teethi1/teethi-patient-app` | **30** | **10 files with uncommitted work** (inline error banners → `ToastOnError`, plus regenerated desktop plugin registrants); crash logs in root; worktree `patient-p2` |
| TEETHI-Dentist-App | `0d85281` | 251 Flutter | GitLab `teethi1/teethi-dentist-app` | **27** | clean; worktree `dentist-p3` |

### 1.3 Biggest risks
1. **Nothing is pushed.** Remotes now exist, but every repo is 12–72 commits ahead. One disk failure loses weeks of work.
2. **Money paths have never run against Stripe end to end.** P1–P3 and the webhook were tested only with the fake gateway (plus a 2026-10-22 sandbox smoke run). No Stripe CLI webhook run, no phone run.
3. **Nothing has run on iOS**, and Android only for the first teledentistry call. Push, Stripe sheet, account deletion and app settings have never run on a phone.
4. **Business and legal gates:** Agora BAA, California lawyer review, D2/D3/D5/D7/D8 are still open, and they block go-live and payments P5–P7.

---

## 2. Decisions needed

Nothing in a phase that depends on one of these should start until the answer is written into its source plan.

### 2.1 Business and legal (owner)

| ID | Decision | Blocks | Source |
|---|---|---|---|
| D1-L | California healthcare lawyer review before go-live (launch state is CA "for now") | Payments P1 go-live, teledentistry launch | PAY §2 |
| D4-L | Lawyer confirms a non-dentist company can be the seller of dental care in CA (TEETHi is seller "for now") | P1 go-live, P8 | PAY §2 |
| D2 | Revenue model: commission per visit, flat fee per booking, or dentist subscription (many states restrict fee-splitting) | P7 | PAY §2 |
| D3 | Dentists paid per visit, or salary / fixed fee (salary shrinks P7 to an earnings view and drops P8) | P7, P8 | PAY §2 |
| D5 | Who absorbs Stripe fees and lost disputes; the refund hold period before earnings become payable | P7 | PAY §2 |
| D7 | Late-cancellation and no-show policy: window, fee, who can waive | P6 | PAY §2 |
| D8 | Keep pay later (option A: needs a saved card; option B: remove)? Allow cash at the door? | P5 | PAY P5 |
| D6-R | Revisit before P4: price can rise during the visit with **no cap and no patient approval**. P4 turns that straight into card charges (SEC S3 note) | P4 | PAY §2, SEC S3 |
| BAA-1 | **Agora BAA signed** (blocks any real patient call) | Teledentistry launch | TELE T0 |
| BAA-2 | Agora region pinning (`areaCode`), decide together with the BAA | Teledentistry launch | TELE T8 |
| BAA-3 | Confirm BAAs (or a no-PHI guarantee) for SendGrid and Twilio. The receipt email no longer carries PHI (S1), but confirm the agreement anyway | Go-live | SEC S1 |
| LEG-1 | Teledentistry consent text from legal (a placeholder is seeded) | Teledentistry launch | TELE T0 |
| LEG-2 | Record retention: how long a deleted account's records are kept before purge | Go-live | IMP account-deletion addendum |
| LEG-3 | Clinical note revision history: "last write wins + audit log" enough, or a full revisions table? | — | IMP §7 Q2 |
| LEG-4 | Sales tax on dental services in CA (accountant) | Go-live | PAY §4 |

### 2.2 Product

| ID | Decision | Blocks | Source |
|---|---|---|---|
| PR-1 | Licensed states are self-declared by dentists with no admin check. Add verification, or accept? | Teledentistry launch | TELE T8 |
| PR-2 | Call screen: block screenshots / app-switcher preview (`FLAG_SECURE`)? Ongoing-call notification? | Teledentistry polish | TELE T8 |
| PR-3 | Tele call joins: write them into the PHI access log too (plan §5.3 says so), or is `tele_session_participants` enough? | — | TELE T8 |
| PR-4 | One combined patient consent form, or several (backend allows one active form per template type) | — | CR §5 |
| PR-5 | Should an incomplete dentist profile block Requests/Schedule, and should the app route to a "finish your profile" state? (Bookability already hides them from patients) | DA Phase 1 leftover | DAP §16 #8, Phase 1 |
| PR-6 | Admin messaging viewer (dashboard Phase 12): only if a support/dispute workflow needs it | DB Phase 12 | ADP Phase 12 |
| PR-7 | Who sees prices: should dentists get treatment prices from `GET /doctor/services` / `GET /treatments`? Should `view_invoices` staff without `view_patients` see treatment names on invoice lines? | SEC S5, S13 | SEC |
| PR-8 | Sign-up still reveals that an email is registered (different message). Fully hiding it needs "verify email before the account exists". Redesign or accept? | — | E2E Low |
| PR-9 | Open-pool (emergency) requests reach dentists regardless of their services. Filter by expertise? | — | E2E R8-07 |
| PR-10 | Show a "Deleted" badge in the dashboard (needs `account_deleted_at` exposed) | — | ADP deleted accounts |
| PR-11 | Teledentistry incoming-call ring: CallKit / ConnectionService with VoIP pushes (T10 "later", separate approval) | Later | TELE T10 |
| PR-12 | Older open questions: one shared treatment catalog (Q1), city as browse filter only vs jurisdiction (Q3), admin token abilities/scopes (Q4) | — | IMP §7 |

### 2.3 Engineering approvals
- `firebase/php-jwt` as an explicit backend dependency (Phase 16). IMP Phase 16.
- Flutter packages `google_sign_in` 7.x, `sign_in_with_apple`, `crypto` (both apps). PAP Phase 11.
- Optional `timezone` package for real zone abbreviations (PDT) in the patient app. PAP timezone section.

---

## 3. Owner actions (accounts, hosting, devices)

These need the owner, not code. Several gate phases in §5.

**Now**
- [x] **OPS** Push all five repos to their remotes (§1.2). `master` only, all five pushed 2026-11-11.
- [ ] **OPS** When staff roles are created in production, grant `manage_appointments` / `view_messages` to those that need them (Super Admins already have them). CR §5.

**Before payments go-live (P1)**
- [ ] **OPS** Verified live Stripe account for the launch entity; restricted live key (`rk_live_`) with only the gateway's permissions. PAY P1.
- [ ] **OPS** Live webhook endpoint for `StripeWebhookService::EVENTS`, including `charge.dispute.created` / `charge.dispute.closed`; `STRIPE_SECRET_KEY`, `STRIPE_PUBLISHABLE_KEY`, `STRIPE_WEBHOOK_SECRET`. PAY P1.
- [ ] **OPS** Apple Pay: merchant ID, payment-processing certificate in Stripe, `STRIPE_MERCHANT_ID` dart-define, iOS capability. PAY P1, IMP Phase 11.
- [ ] **OPS** Google Pay production access. PAY P1.
- [ ] **OPS** Install the Stripe CLI on this machine (needed for the webhook test in Phase 1).

**Before any production launch**
- [ ] **OPS** API on HTTPS at a real domain (needed by the Stripe webhook, Agora NCS, Apple sign-in return URL).
- [ ] **OPS** Production env (CR §5): `APP_DEBUG=false`, `OTP_EXPOSE_IN_RESPONSE=false`, `CORS_ALLOWED_ORIGINS`, `SANCTUM_TOKEN_EXPIRATION`, `MAIL_MAILER` real transport, `STRIPE_SECRET_KEY`. Run `php artisan migrate` on every deploy (README rule 5).
- [ ] **OPS** Twilio: upgrade from trial (custom SMS bodies fail with 572006); SendGrid sender verified. IMP Phase 17.
- [ ] **OPS** FCM in production: attached service account or workload identity federation, `FIREBASE_PROJECT_ID`, **a supervised queue worker** (OTP, push and receipts are queued). IMP Phase 15.
- [ ] **OPS** Agora: turn on NCS for `https://<api-host>/api/v1/agora/webhook`, events 102–108, put the secret in `AGORA_NCS_SECRET`. TELE T0/T2.
- [ ] **OPS** A Mac with Xcode and an APNs key: first iOS build of both apps (Podfile gets `PERMISSION_CAMERA=1`, `PERMISSION_MICROPHONE=1`), push on iOS, Apple Pay, Sign in with Apple. TELE T4/T5, CR §5.
- [ ] **OPS** Release signing keys for Android (and their SHA-1s for Google sign-in).

**Before Google/Apple sign-in (Phase 16)**
- [ ] **OPS** Google Cloud `teethi-508df`: OAuth consent screen, one Web-application client, two iOS clients, two Android clients (debug SHA-1 now, release later). IMP Phase 16.
- [ ] **OPS** Apple Developer: Team ID, Sign in with Apple on both App IDs, two Services IDs with the HTTPS return URLs, a `.p8` key and Key ID (outside every repo), private email relay registration with SPF/DKIM. IMP Phase 16.

---

## 4. All open work by category

### 4.1 Repo and docs hygiene
- [x] **PA** Decide on the 10 uncommitted files on `master` (inline error banners replaced by `ToastOnError`, regenerated Linux/macOS/Windows plugin registrants): finish and commit on a branch, or discard.
- [x] **DOCS** Commit or revert the root `.claude/launch.json` change. Decide about the untracked `TEETHi-Frontend/CLAUDE.md`.
- [x] **ALL** Delete merged feature branches; `git worktree prune` the backend's `claude/mystifying-rhodes-2e8412`; remove the four `D:/Projects/TEETHi-worktrees/*` worktrees whose branches are merged. CR M12.
- [x] **PA** Delete `hs_err_pid*.log` and `flutter_01.log` from the repo root (and gitignore them). CR M12.
- [ ] **DB** Remove unused CoreUI demo views. CR M12.
- [x] **DOCS** Fix stale docs:
  - `README.md` "Not yet done" says no repo has a remote. They all do now; it should say "not pushed".
  - `CONSISTENCY_REPORT.md` was last updated 2026-10-19. Its status table and open list are out of date: **M1 (no Arabic on mobile) is done** (both apps have `app_ar.arb` and a language switch); **M13 is fixed** (R8-24); H5 is partly fixed; test counts are old. Refresh it or mark it historical and point here.
  - `architecture/DENTIST_APP_ARCHITECTURE.md` still says every feature but auth is mock (CR M9).
  - `TELEDENTISTRY_PLAN.md` header says "nothing built yet".
  - `ADMIN_DASHBOARD_FRONTEND_PLAN.md` "Blocked" section still says backend Phases 10 and 11 aren't built.
  - Phase 18 (app settings) notes in IMP, PAP, DAP and ADP say "not committed". Check they're merged and update.
  - `PATIENT_APP_FRONTEND_PLAN.md` §16 "Verified: _(fill in once wired)_".
  - `DENTIST_APP_FRONTEND_PLAN.md` Phase 5 "confirm a saved schedule is bookable" was effectively covered by the E2E run (Layla's slots were booked). Tick it with a note.
- [x] **BE** `composer test` stops at `rector --dry-run` (flags ~169 files on `master`). Either run Rector and commit, or drop it from the script. IMP Phase 13, TELE T1.

### 4.2 Verification debt (built, never run for real)
- [x] **BE+PA** Stripe webhook end to end with the Stripe CLI, test mode (done 2026-11-12, E2E_REPORT_2026-11-12): pay, cancel the sheet, refund, redelivered event, out-of-order event, dispute created/closed. PAY P1, IMP Phase 11.
- [ ] **PA** Android and iOS pass against the Stripe sandbox: success, decline, 3-D Secure (`4000 0025 0000 3155`), a saved card reused, a card removed, the invoice updating after the webhook. PAP Phase 10.
- [x] **ALL** (done 2026-11-12 on web and API; real mail and the app's receipt View still open) Payments P1–P3 "done when" checks in test mode: dispute flag in the dashboard; receipts by email (real mail transport) and in the app; a dentist adds and removes a treatment at completion and the patient, the admin and the invoice all agree. Dashboard P1–P3 and S5/S7 screens were never opened in a browser. PAY P1–P3.
- [x] **PA+DA** (done 2026-11-12 except NCS, which waits on T0) Teledentistry T8 leftovers: complete visit 23 after the call and check the call log duration; leave and rejoin; switch camera; an audio call; a call over 15 minutes (token renewal) and an expired token dropping a participant after completion; NCS integer fields (needs NCS on). TELE T8.
- [ ] **PA+DA** Push on Android and iOS devices (only web was verified; the on-screen pop-up was never seen). CR H4.
- [ ] **PA+DA** Account deletion, app settings gate and the polish-phase changes on a device. PAP/DAP.
- [ ] **ALL** Re-run the full live E2E scenario (CR §7) after the `fix/r8-medium` merge (never re-run live) and again after each phase below. Several Low fixes were only covered by tests.
- [ ] **PA** Map pins: one blank pin at the Santa Monica fallback hides whether doctor photos show on pins. PAP dentist photos.

### 4.3 Backend backlog (no decision needed)
Payments hardening (SEC):
- [ ] **S9** Allow `FakePaymentGateway` only in `local` and `testing`, not "anything except production". Small; do first.
- [ ] **S8** Lock and re-read the invoice in `pay()` before creating the PaymentIntent; better, auto-refund anything paid above the current balance.
- [ ] **S12** One transaction for confirm treatments + outcome + completion with the appointment locked; notify after commit.
- [ ] **S10** Receipt links: ~5-minute lifetime, `throttle:60,1`, `trustProxies` for the host.
- [ ] **S11** Backfill: add an adjustment line when lines don't sum to the stored total; test it.
- [ ] **S13** Update `appointments.treatment_id` when the list changes; generic message instead of `payments.invoice_closed` to dentists. (Prices to doctors: PR-7.)
- [ ] Mirror refunds made in Stripe's dashboard (`charge.refunded`, idempotent). Today staff must record them by hand. SEC S7.
- [ ] Scheduled cleanup of card PaymentIntents left `initiated` with no later attempt. IMP Phase 11 follow-up.

Timezone follow-ups (IMP Phase 13):
- [ ] Admin `filter[scheduled_from/to]` compare UTC dates, not local.
- [ ] `AppointmentReminderNotification` payload lacks `timezone` / `scheduled_at_local`.
- [ ] Today's open slots include slots already past (booking one fails `after:now`).
- [ ] Spring-forward date: a window over the skipped hour yields a duplicate slot (only matters at 2 am).

Other:
- [ ] Server-side role check at login (the app says which app it is), so a wrong-app login never gets a token. E2E R8-09.
- [ ] Close the Agora channel from the server when a visit ends (needs Agora's REST customer key). TELE T2.
- [ ] Dentist-side API gaps: `GET`/`DELETE /doctor/availability/exceptions`; bulk enable/disable a visit type's windows; a way to clear a tooth-chart entry; `distance_km` on open requests. DAP §16.
- [ ] Admin-side API gaps: an action on city requests (and wire `pending → notified` when the city is enabled); `filter[treatment_category_id]` on `GET /admin/treatments`; `AppointmentResource.invoice` is never populated; expose `account_deleted_at` (PR-10). ADP §7.

### 4.4 Admin dashboard backlog
- [ ] Phase 0 leftovers: global `429` surface (string already exists), `GET /enums` cache feeding every label, typed envelope union. CR M5, M3; ADP Phase 0.
- [ ] Appointment detail view (backend already serves `GET /appointments/{id}` to admins) and a real Home page instead of the placeholder. CR M6.
- [ ] R8-19 leftovers: doctor detail shows documents, appointments, services and availability; Invoices list gets patient and dentist columns; Appointments shows all treatments, not the first.
- [ ] "Cities" column labelled "State/Emirate". CR E10.
- [ ] "Deleted" badge (after PR-10).

### 4.5 Patient app backlog
- [ ] **Phase 2b / H5:** insurance management screen (list, add, edit, delete, card photos) outside the booking flow; a payment methods screen (`GET/POST/DELETE /patient/payment-methods`, SetupIntent sheet). CR H5, PAP Phase 2.
- [ ] **M7:** remove leftover mock state. `lib/state/app_state.dart` and `lib/data/mock_data.dart` are still referenced by 27 files (booking draft, payment options, parts of profile).
- [ ] **R8-01 leftover:** handle the new `404` on a dentist's profile, reviews and slots (dentist no longer listed).
- [ ] **E8:** relative labels ("Tomorrow") use the device date, not the visit's zone.
- [ ] **E9:** booking requires device geolocation to confirm the address, with no fallback (no geocoder exists).
- [ ] **E10:** "Use profile" in the visit-address step does nothing when the profile has no address; the consent list briefly shows the previous list.
- [ ] Teledentistry: the Consent forms banner counts an unsigned teledentistry consent as blocking all visits; Join call only on Visits (not Home's next-visit card); Android foreground service for a call in the background; keep the screen awake. TELE T3/T4.
- [ ] "Already reviewed" signal for completed visits (hidden only per session today). PAP Phase 6.
- [ ] Finish the swap to `GET /enums` labels where screens still hardcode them. PAP Phase 0, CR M3.
- [ ] Lint debt (17 infos: `withOpacity` etc.). CR M11.

### 4.6 Dentist app backlog
- [ ] **R8-01 leftover:** show `bookable` / `bookability_gaps` from `GET /doctor/dashboard` so a dentist knows why patients can't see them.
- [ ] Teledentistry: Android foreground service, keep the screen awake, and the schedule note should actually check that Virtual Consultation is in the dentist's services. TELE T5.
- [ ] Change-password screen (the view model already has it; no screen offers it). DAP Phase 12 lists it, but it doesn't depend on social sign-in.
- [ ] Phase 1 "finish your profile" routing (after PR-5).
- [ ] Smaller gaps: open message attachments and uploaded credentials (`url_launcher` + fresh signed URL); Home bell dot after reading notifications; reset per-account view models on a `401` sign-out; visit workflow forgets what was saved when reopened. DAP Phases 2, 3, 8, 10.
- [ ] Finish the swap to `GET /enums` labels; lint debt (46 infos). DAP Phase 0, CR M11.

### 4.7 Feature roadmap (new builds)
| Feature | Gated by | Source |
|---|---|---|
| **Payments P4:** card on file, 24-hour hold, charge on completion | D6 decided; D6-R revisit; Stripe test-mode run first | PAY P4 |
| **Payments P5:** pay later limited or removed, optional cash | D8 | PAY P5 |
| **Payments P6:** cancellation and no-show fees, partial refunds, waive fee | D7, P4 | PAY P6 |
| **Payments P7:** dentist earnings ledger, statements, bank-transfer payouts (closes E2E R8-25) | D2, D3, D5 | PAY P7 |
| **Payments P8:** Stripe Connect payouts | D4-L, P7, when manual payouts hurt | PAY P8 |
| **Sign in with Google and Apple:** backend Phase 16, patient Phase 11, dentist Phase 12 (plus Apple token revocation in account deletion) | §3 Google/Apple setup, HTTPS, Android/iOS devices | IMP Phase 16, PAP Phase 11, DAP Phase 12 |
| Dashboard Phase 12: read-only messaging viewer | PR-6 | ADP Phase 12 |
| Teledentistry CallKit / VoIP ring | PR-11 | TELE T10 |
| Claim-ready receipts (CDT codes, NPI, tax ID); insurance; BNPL / financing | Later, out of scope today | PAY §4 |

---

## 5. Phases

Each phase lists what to do, what it's waiting on, and when it's done. Phases 2 and 3 can run in parallel; Phase 6 runs alongside everything as owner actions land.

### Phase A — Secure and tidy (now, about a day)
**Goal:** nothing can be lost, and the docs tell the truth.
- [x] Push all repos (§3 "Now"). `master` only; merged feature branches deleted locally (decided 2026-11-11). All five pushed 2026-11-11 (GitLab after fixing the sign-in): backend `ff9e731`, dashboard `6799b50`, patient app `20aa3e8`, dentist app `0d85281`.
- [x] Settle the patient app's uncommitted work (merged as `20aa3e8`), the root `launch.json` (committed: E2E and port-8001 run configs) and the frontend `CLAUDE.md` (committed: `@AGENTS.md` pointer) (§4.1).
- [x] Delete merged branches and stale worktrees; remove crash logs (§4.1).
- [x] Fix the stale docs listed in §4.1; `CONSISTENCY_REPORT.md` is marked historical, with M1, M9 and M13 moved to resolved.
- [x] Rector: applied once and committed (TEETHi-Backend `ff9e731`). Three rules are skipped in `rector.php` because they broke code; `composer test` passes end to end (874 tests).
- [x] Grant `manage_appointments` / `view_messages` to the right roles. Nothing to grant locally: the only role is Super Admin, which already has everything. Moved to go-live (§3): grant them when staff roles are created.

**Done when:** every repo is clean, `origin/master` matches local `master` everywhere, and the docs match the code.

**Done 2026-11-11.**

### Phase B — Verify what's merged (1–2 weeks)
**Goal:** prove the money and call paths work for real before building on them.
**Waits on:** Stripe CLI installed; an Android phone (have: SM-A566B and SM-X210); a Mac for the iOS half (can trail).
- [x] Stripe CLI webhook run in test mode (§4.2). Done 2026-11-12, all scenarios passed: [E2E_REPORT_2026-11-12.md](E2E_REPORT_2026-11-12.md) §2.
- [x] Patient app Stripe pass on Android (iOS when a Mac is available). Android done 2026-11-12, all checks passed; found B1-05 (cancelled invoice payable) and B1-06 (sheet Confirm under the nav bar, fixed on `fix/sheet-nav-bar`). iOS still open.
- [x] Payments P1–P3 "done when" checks, dashboard screens opened in a browser. Done 2026-11-12 through the API and the dashboard (mail to the log, not SendGrid); phone parts stay with the item above. Found B1-01 (Medium) to B1-04.
- [x] Teledentistry T8 leftovers on the two Android devices; then tick T8. Done 2026-11-12, all passed except the NCS check (waits on T0). Found B1-07 (Low).
- [ ] Push on Android. Skipped 2026-11-12: the local gcloud application-default sign-in has expired, so the backend skips every push. Re-run `gcloud auth application-default login` first (owner action).
- [ ] Full live E2E scenario on web plus the Android checks above; log new findings with a new prefix in a new E2E report.

**Done when:** PAY P1's "done when" holds in test mode, TELE T8 is ticked, and the new E2E report has no High findings open.

### Phase C — Hardening backlog (1–2 weeks, parallel with D)
**Goal:** close the known low-severity money and correctness gaps before P4 adds more money movement.
- [ ] S9, S8, S12, S10, S11, S13 (§4.3). Run `security-reviewer` before merging.
- [ ] `charge.refunded` mirroring and stale PaymentIntent cleanup. B1-01 fixed 2026-11-12 (TEETHi-Backend `4f8e9d1`): reconcile and visit cancellation now cancel open intents. Still open: mirroring refunds made in Stripe's dashboard, and intents abandoned without any later action (they expire in Stripe but stay `initiated` here).
- [x] **B1-05 (Medium):** refuse payments on cancelled, refunded and paid invoices (backend `409`), and show the app's Pay button only for pending and partially paid invoices. Fixed 2026-11-12 and merged to `master` (pushed): TEETHi-Backend `f553122`, TEETHi-Patient-App `e20fcac` (with the B1-06 sheet fix `e96d4b9`).
- [ ] B1-02 to B1-04 (Lows from the 2026-11-12 run; B1-02 needs a product call). B1-07 fixed 2026-11-12 (TEETHi-Backend `2d23774`).
- [ ] Timezone follow-ups (§4.3).
- [ ] Server-side login role check.
- [ ] Update `PAYMENTS_SECURITY_FINDINGS.md` statuses.

**Done when:** SEC has no Open rows except those waiting on PR-7, and the backend suite, PHPStan and Pint pass.

### Phase D — Client gaps (2–3 weeks, parallel with C)
**Goal:** no mock screens or known UI dead ends left in any client.
- [ ] Patient: Phase 2b (insurance and payment methods screens), M7 mock-state removal, R8-01 `404`, E8–E10, tele leftovers, review signal (§4.5).
- [ ] Dentist: bookability gaps on Home, tele leftovers, change-password screen, smaller gaps (§4.6).
- [ ] Dashboard: Phase 0 leftovers (429, `/enums`), appointment detail and Home page, R8-19 leftovers, label fix (§4.4).
- [ ] Backend API gaps the clients need (§4.3 "Other"): availability exceptions list/delete first.
- [ ] Run `contract-reviewer` and `/sync-api-contract` after the backend pieces.

**Done when:** H5, M3, M5, M6, M7, E8–E10 are closed in `CONSISTENCY_REPORT.md` and every client suite passes.

### Phase E — Payments P4: card on file, hold, charge on completion (3–4 weeks)
**Waits on:** Phase B (Stripe proven), Phase C (S8/S12 fixed), and D6-R reconsidered.
- [ ] Build P4 across all four repos per PAY P4.
- [ ] Confirm whether wallet cards can be saved for off-session charges; if not, pay-after-visit is card-only.
- [ ] `security-reviewer` and `contract-reviewer` before merge.

**Done when:** PAY P4's "done when" passes in Stripe test mode.

### Phase F — Remaining payments phases (as decisions land)
- [ ] **P5** after D8.
- [ ] **P6** after D7 (needs P4).
- [ ] **P7** after D2, D3, D5 (can start in parallel with P4 once decided; only needs P1). Closes R8-25.
- [ ] **P8** later, after D4-L and P7, when manual payouts become a burden.

### Phase G — Sign in with Google and Apple (2–3 weeks)
**Waits on:** §3 Google/Apple setup, HTTPS domain or tunnel, dependency approvals (§2.3), an Android device; a Mac for iOS.
- [ ] Backend Phase 16, including `AppleTokenClient::revoke()` in account deletion.
- [ ] Patient Phase 11 and dentist Phase 12 (shared pieces built the same way in both).
- [ ] Verify live on Android and iOS per each plan's "Verify live" list.

**Done when:** all "Verify live" checks in PAP Phase 11 and DAP Phase 12 pass.

### Phase H — Go-live readiness
**Waits on:** the legal and BAA items in §2.1 and the production owner actions in §3.
- [ ] Legal: D1-L, D4-L, LEG-1, LEG-2, LEG-4; BAAs (BAA-1 to BAA-3).
- [ ] Product: PR-1, PR-2, PR-3 settled for teledentistry.
- [ ] Production environment, HTTPS, queue worker, FCM, Twilio, SendGrid, Stripe live, Agora NCS (§3).
- [ ] First iOS builds of both apps; release builds on Android and iOS.
- [ ] A final `security-reviewer` pass over the whole backend; a final E2E on release builds with live Stripe (a real small charge and refund).
- [ ] Store listings, privacy policy text in app settings, App Store / Play submission.

**Done when:** PAY P1 is ticked, TELE T0 is ticked, and both apps are approved in the stores.

### Phase I — Later
- Dashboard Phase 12 messaging viewer (PR-6), CallKit ring (PR-11), Stripe Connect (P8), claim-ready receipts and insurance, BNPL, the remaining open questions (LEG-3, PR-4, PR-8, PR-9, PR-12).

---

## 6. Keeping this file current
- Tick items here and in the source plan in the same root commit, naming the sub-repo commit hash (CLAUDE.md).
- When a decision in §2 is made, record it in the source plan's decision table first, then strike it here with the date.
- When a phase finishes, add a one-line "Done (date, commits)" under it.

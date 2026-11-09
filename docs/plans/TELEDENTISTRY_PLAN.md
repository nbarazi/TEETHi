# TEETHi — Teledentistry Plan (video and audio consultations)

Status: **decisions approved 2026-11-09**, nothing built yet. Phase T0 (Agora BAA) gates any real patient call.

Covers all four repos: the backend, the patient app, the dentist app and the admin dashboard. The patient-side screens come from the Claude Design project "TEETHi Patient app" (`PatientAppV2.jsx`, Teledentistry branch). §3 lists where the approved decisions change that design. The dentist app reuses its existing patient/request page designs and the same call screen as the patient app (decision Q5).

---

## 1. Context

Teledentistry is a third kind of visit, next to the at-home routine visit and the at-home emergency visit. The patient books a consultation with a California-licensed dentist and meets them over a video or audio call **inside the TEETHi apps**. Typical reasons are advice, second opinions, follow-ups and prescriptions. If treatment is needed, the dentist books an in-person visit.

The calls carry PHI, so every vendor that touches the media or its metadata must sign a **BAA**.

## 2. Build vs buy

| | Build on WebRTC ourselves | Self-host an open-source media server | **Managed provider with a BAA** ✅ |
|---|---|---|---|
| We build | Signalling, TURN/STUN, reconnects, call UI plumbing, monitoring | Hosting and operations | App screens plus a token endpoint |
| HIPAA | Everything is on us | Cloud BAA; patching, logs and uptime are on us | The vendor signs a BAA and runs the media |
| Time to first call | ~3–4 months | ~6–8 weeks | **~3–5 weeks** |

**Approved: Agora**, behind a backend `VideoCallProvider` interface with a `FakeVideoCallProvider` (the same pattern as `PaymentGatewayService`, Phase 11). Tests and local development never need Agora, and the vendor can be swapped later without touching the apps' booking code.

Agora specifics:
- **Apps:** Agora's official Flutter SDK, `agora_rtc_engine`.
- **Tokens:** the backend issues an RTC token per user per channel with Agora's token builder (AccessToken2). The App Certificate stays server-side.
- **Encryption:** turn on Agora's built-in media encryption (`AES-128-GCM2`). The backend creates a key and salt for each session and returns them with the token.
- **Webhook:** Agora's Notification Center Service (NCS) reports channel and participant events. The backend checks each event's signature.
- **BAA:** confirm Agora signs a BAA for this account and plan (T0). If it can't, stop and go back to the fallback (LiveKit or Zoom Video SDK) through the same interface.

## 3. Screens

### 3.1 Patient app (from the design, with the approved changes)

| # | Screen | Built as | Changes from the design |
|---|---|---|---|
| D1 | **Book → "What kind of visit?"** — third card **Teledentistry**, "From $45" | `BranchStep` | Copy: "A virtual consultation by video or audio call in the app" (no "phone"). |
| D2 | **Teledentistry step**: call type, reason, description, when | New `ConsultStep` | Call type: **Video call** / **Audio call** ("Audio only, in the app"). Reason chips: Tooth pain, Second opinion, Follow-up, Prescription, Cosmetic advice, Other. "Describe it briefly". **When: only the week strip and time slots.** "Next available" and "Remind me" are removed (T3, T4). |
| D3 | **Choose a dentist** | Existing `DentistStep` | Filtered to dentists with teledentistry hours who are licensed in the patient's state (T11). |
| D4 | **Insurance details** | Existing `InsuranceStep` | **Added to the route** (T6): the same step and rules as a routine home visit. |
| D5 | **Before we book** | Existing confirm step | Badge **Video** / **Audio**. Rows: Treatment "Virtual consultation · {reason}", When, **Call type** ("Video call in the app" / "Audio call in the app"), Insurance, Estimated total, Plan may cover. **No phone number** (Q6). The teledentistry consent joins the consent check (T12). |
| D6 | **Payment** | Existing `PaymentStep` | **Same step as the other visit types**: Stripe Payment Sheet (card, Apple Pay) or pay later, with the same invoice rules. |
| D7 | **Confirmation** | Existing | One message for both modes: "Join the call from Visits when it is time — we will notify you." |
| D8 | **Visits → Upcoming card**: badge **Teledentistry**, "Video call in the app" / "Audio call in the app" | `VisitCard` tele variant | Buttons: **Cancel**, **Reschedule**, **Join call** (Q4). Cancel and Reschedule use the same reason sheet and rules as home visits. |
| D9 | **Call screen** (full screen, dark): switch camera, "TEETHi Consult", timer, Leave; dentist tile with self-view picture-in-picture (tap to swap) | New `VideoCallScreen` | Dock: **Mute**, **Stop video**, **People**. **Chat and Share removed** (T7, T8). The Leave sheet says notes and any prescription will appear in Records. Audio mode shows avatars instead of video and hides Stop video and Switch camera. |
| D10 | **After the visit**: visit summary, prescriptions in Records | Existing post-visit flow | Unchanged. |

Route: `virtual = ['consult','dentist','insurance','confirm','payment','confirmation']`, the same chrome as the other routes (5 numbered steps).

**As built (T3):** `['consult','dentist','visit','insurance','confirm','payment','confirmation']` (6 numbered steps). Open slots come from one dentist's tele hours (`GET /patient/availability` needs `doctor_profile_id`), so the day and time are picked after the dentist, as in the routine route. The tele `visit` step is only the week strip and slots, with no address. The consult step adds "Where will you be during the call?" (a state picker, prefilled from the profile address), which T11 needs.

### 3.2 Dentist app (no new design; reuse existing patterns)
- Pages follow the existing request detail and patient pages.
- The call screen is the **same design as the patient's** D9, with the dentist as host.

### 3.3 Design file follow-ups
Update the Claude Design project to match §3.1: remove "Next available", "Remind me", Chat, Share and the phone number. Add the insurance step and the Cancel button on tele visit cards. Rename Phone to Audio.

## 4. Approved decisions

| # | Decision | Approved |
|---|---|---|
| T1 | Video vendor | **Agora behind `VideoCallProvider`** (see §2). |
| T2 | What "phone call" means | **An in-app audio-only call**: the same channel with the camera off. No calling outside the app (Q1, Q6). |
| T3 | "Next available" (on demand) | **Removed.** Scheduled slots only. |
| T4 | "Remind me" | **Removed.** |
| T5 | Price | **A catalog treatment "Virtual consultation"** (`$45`, admin-editable). Invoice at booking per decision #11. |
| T6 | Insurance | **The same insurance step as a regular visit** (Q7: like home visits for now). |
| T7 | In-call chat | **None.** The visit's existing messaging stays available outside the call. |
| T8 | Sharing during the call | **None** (no records, photos or screen sharing). |
| T9 | Recording | **None.** |
| T10 | Incoming-call ring | **v1: a high-priority generic push** ("Your consultation is ready") that opens the visit. **Later: CallKit / ConnectionService** with APNs VoIP pushes. |
| T11 | Licensure | **Dentists list the states they are licensed in. Tele booking shows only dentists licensed in the patient's state. The patient confirms their current state at Join.** |
| T12 | Teledentistry consent | **A new consent document** in the existing legal documents flow, required before the first tele booking. |
| Q4 | Cancel | **Patients can cancel a tele visit with the same reasons and rules as a home visit.** |
| Q5 | Dentist screens | **Reuse the dentist app's existing patient/request page design. The call screen matches the patient app's.** |
| — | Payment | **The same payment step as the other visit types** (Stripe sheet or pay later). |

## 5. Backend (TEETHi-Backend)

### 5.1 Data model
- `VisitType::Tele = 'tele'`. No address: `visit_address_id` stays null, and booking validation skips the address rules for it.
- New fields on `appointments`:
  - `call_mode` enum(`video`,`audio`)
  - `consult_topic` (an enum of the six reasons)
  - `consult_description` (**encrypted**)
- `doctor_availability_slots.visit_type` accepts `tele`, so a dentist publishes tele hours separately from home-visit hours.
- Licensed states for each doctor: a `doctor_licensed_states` table (doctor, state code).
- The patient's state for the call: `appointments.patient_state`, set at booking and checked against the dentist's licences. Join re-confirms it (T2).
- The consult description reuses the existing encrypted `problem_description` (no new column).
- `treatments.is_virtual`: the consultation is a catalog treatment, and tele visits take only virtual treatments.
- **`tele_sessions`**:
  - `appointment_id` (unique), `provider`
  - `channel_name` (random, **no PHI**)
  - `encryption_key` and `encryption_salt` (**encrypted**)
  - `started_at`, `ended_at`, `ended_reason`, `duration_seconds`
- **`tele_session_participants`**: `tele_session_id`, `user_id`, `role` (host/patient), `joined_at`, `left_at`. Filled from Agora NCS events.
- `LegalDocumentType` gains a teledentistry consent type (T12).
- Catalog seed: the "Virtual consultation" treatment (T5).

### 5.2 Endpoints
- **Availability, booking, insurance, payment, cancel and reschedule:** the existing endpoints, accepting `visit_type=tele`, `call_mode`, `consult_topic` and `consult_description`. `BookingService::missingRequirements()` adds the tele consent. Invoices and the Stripe payment are unchanged.
- **`POST /patient/appointments/{id}/call-token`** and **`POST /doctor/appointments/{id}/call-token`** return `{ app_id, channel_name, uid, token, expires_at, encryption: { mode, key, salt } }`. Rules:
  - The caller must be this visit's patient account or doctor.
  - Only for an `accepted` / `in_progress` tele visit, inside the start window (`START_WINDOW_MINUTES`) and until a grace period after the slot.
  - The token lives ~15 minutes and is bound to the channel and uid.
  - The patient's request carries `current_state` (T11). It is refused if the doctor isn't licensed there.
  - The doctor's first token moves the visit to `in_progress` (the existing "Start visit"), creates the session and pushes the patient (T10).
- **`POST /api/v1/agora/webhook`** (NCS):
  - Public and unthrottled. Rejects a bad signature, and handles each event once (like `stripe/webhook`).
  - Writes participant joins and leaves, and the session end.
- **Ending a visit** stays the doctor's existing **Complete visit** with the outcome, prescriptions and recommendations. After that, no new tokens are issued.

### 5.3 HIPAA controls
- Agora BAA signed **before any real patient call** (T0).
- Media encryption on, with a key for each session, never logged.
- No names, treatments or reasons in channel names, uids or pushes (decision #14).
- Each token issue and each join writes `PhiAccessLog`.
- No recording, chat or sharing (T7–T9).
- Run the `security-reviewer` agent before merge.

### 5.4 Tests
Pest coverage for:
- Token authorization: wrong user, wrong status, outside the window, an unlicensed state, admin refused.
- The NCS webhook: signature check and once-only handling.
- Booking without an address, with insurance and payment as for home visits.
- Cancel and reschedule reasons.
- Hiding dentists not licensed in the patient's state.
- `FakeVideoCallProvider` in every feature test.

Refresh `docs/api/openapi.json` and run `/sync-api-contract`.

## 6. Patient app (TEETHi-Patient-App)
- **Booking screens:** branch card, `ConsultStep`, and the tele variants of Dentist / Insurance / Before we book / Payment / Confirmation / Visit card (D1–D8).
- **Call screen (D9):** built on `agora_rtc_engine`.
  - Ask for mic permission, plus camera in video mode.
  - Confirm the patient's current state before joining (T11).
  - Join with the backend's token and encryption settings.
  - Controls: mute, stop video, switch camera, swap tiles, People, Leave.
  - Timer from the session's `started_at`.
  - Also needs a "Waiting for your dentist" state and a "Reconnecting" state.
- **Join call:** enabled only inside the window, using the server's time from the appointment.
- **Refresh rules:** the "consultation ready" push and the end of the call refresh only the visit's view model. No polling (CLAUDE.md).
- **Native setup:**
  - iOS: `NSCameraUsageDescription`, `NSMicrophoneUsageDescription`, background audio mode.
  - Android: `CAMERA`, `RECORD_AUDIO`, and `FOREGROUND_SERVICE_MICROPHONE` / `_CAMERA` for a call that continues in the background.
- **Web:** show "Video and audio consultations work in the TEETHi mobile app", as the Stripe sheet does.

## 7. Dentist app (TEETHI-Dentist-App)
- **Availability:** a **Teledentistry** slot type next to at-home and emergency. A licensed-states field in the profile.
- **Lists:** requests and today's list show a Teledentistry badge with call mode, reason and description. They have no address or travel information. The detail pages follow the existing patient/request page design.
- **Start call** on an accepted tele visit inside the window: fetch the token and open the call screen. It is the same design as the patient's, with the dentist as host.
- **After End call:** the existing Complete visit flow (outcome, pending, medication, recommendations).
- **Contract tests** in `test/contract` for the new fields.

## 8. Admin dashboard (TEETHi-Frontend)
- Appointments table: a `tele` visit-type filter and badge, and a call column (duration and who joined, from `tele_sessions`).
- Treatment catalog: the "Virtual consultation" treatment is admin-editable (exists already).
- Patient detail → Appointments tab shows the call log. Permission: `view_appointments` (decision #12).

## 9. Phases

| Phase | Scope | Rough size |
|---|---|---|
| **T0** | Agora account, **BAA signed**, App Certificate, NCS set up, design file updated per §3.3 | 1 week (mostly waiting) |
| **T1** | Backend: visit type, fields, tele slots, licensed states, consent type, catalog item, booking/insurance/payment/cancel for tele, tests | 1 week |
| **T2** | Backend: `VideoCallProvider` (Agora plus fake), call-token endpoints, encryption keys, sessions, NCS webhook, audit, tests | 1 week |
| **T3** | Patient app: booking flow D1–D8 | 1 week |
| **T4** | Patient app: call screen D9, native permissions | 1 week |
| **T5** | Dentist app: tele availability, licensed states, badges, call screen as host | 1–1.5 weeks |
| **T6** | Admin: filter, badge, call log | 2–3 days |
| **T8** | Security review, live run on two real phones, Verified notes here | 3–4 days |
| *Later* | CallKit / VoIP ring (T10) | separate approval |

T3 can start once T1's contract is fixed, in parallel with T2. In total, about **4–5 weeks** to a pilot.

## 10. Progress

- [ ] T0 Agora account and BAA — in progress (2026-11-09)
  - [x] Agora project in secure mode. `AGORA_APP_ID` and `AGORA_APP_CERTIFICATE` are in TEETHi-Backend `.env` (gitignored): both are 32-character hex and different from each other. They aren't proven against Agora yet; the first token that joins a channel in T2 does that.
  - [x] Design file updated per §3.3 (Claude Design "TEETHi Patient app", version 4):
    - Branch copy and the consult step say "video or audio call in the app". Audio replaces Phone.
    - "Next available" and "Remind me" are removed; only the day and time picker is left.
    - The tele route now has the insurance step.
    - Before we book and Confirmation have no phone number.
    - Tele visit cards have Cancel, Reschedule and Join call.
    - The call screen has no Chat or Share.
  - [ ] BAA signed with Agora (business; blocks real patient calls, not development).
  - [ ] NCS (webhook): turn it on in the Agora console once T2's webhook URL exists, then add `AGORA_NCS_SECRET` to `.env`.
  - [ ] Teledentistry consent text from legal (a placeholder is seeded).
  - [ ] Launch states confirmed (seed assumes California).
- [x] T1 Backend booking — built 2026-11-09 on TEETHi-Backend `feat/teledentistry-booking` (`4446128`), not merged yet.
  - `VisitType::Tele` (`tele`). A tele booking has no address (`address` is prohibited) and needs `call_mode` (`video`/`audio`), `consult_topic` (the six reasons) and `patient_state` (USPS code: the state the patient will be in). The description goes in the existing encrypted `problem_description`. Home-visit bookings reject the three tele fields. `AppointmentResource` returns all three, and `visit_address: null` for tele.
  - Tele visits are **direct only** (the open pool matches dentists by distance to an address).
  - The consultation is a catalog treatment: new `treatments.is_virtual` flag (admin create/update and `TreatmentResource`). A tele booking takes only virtual treatments, a home visit none. Seeded "Virtual Consultation", $45, Preventative Dentistry. Invoice at booking and payment are unchanged (decision #11, Phase 11).
  - **How a dentist offers teledentistry:** `tele` availability windows, plus the Virtual Consultation in their services (the existing R8-07 check), plus licensed states. Slots, booking and reschedule checks run through `AvailabilityService` unchanged.
  - Licensed states: `doctor_licensed_states` table. `PATCH /doctor/profile` takes `licensed_states` (the full list; `[]` clears it), and `DoctorProfileResource`/`ProviderResource` return it. A direct tele booking is refused unless the dentist is licensed in `patient_state`.
  - Directory: `GET /patient/providers?visit_type=tele&licensed_state=CA`.
  - Consent (T12): `DocumentTemplateType`/`LegalDocumentType::TeledentistryConsent`. It is listed and signed through the patient consent-form endpoints, and is a missing requirement only for tele bookings (`booking.teledentistry_consent_not_confirmed`). A placeholder template is seeded until legal supplies the text.
  - Cancel and reschedule: unchanged endpoints and reasons (Q4).
  - Seeded dentists: tele hours Mon–Fri 09–17, the Virtual Consultation, and a licence in their city's state.
  - Tests: `TeledentistryBookingTest` (21). Full suite: 773 Pest tests pass, PHPStan max and Pint clean. Rector `--dry-run` already flags 169 files on `master`, so that check was left as it was. OpenAPI refreshed.
  - **For the clients:**
    - The patient app must send `visit_type=tele` and no address.
    - The dentist app's schedule `VisitType` and the dashboard's `VisitType` union don't know `tele` yet (T3, T5, T6). Until then, a tele visit shows as a home visit with a blank address, and the dentist schedule skips tele windows. Nothing crashes.
    - The dashboard's treatment form needs the `is_virtual` toggle (T6).
- [x] T2 Backend call sessions — built 2026-11-10 on TEETHi-Backend `feat/teledentistry-calls` (`81f9c7b`), stacked on `feat/teledentistry-booking`. Not merged yet.
  - `VideoCallProvider`: `AgoraVideoCallProvider` when `AGORA_APP_ID` and `AGORA_APP_CERTIFICATE` are set, otherwise `FakeVideoCallProvider` (the same pattern as the payment gateway; `phpunit.xml` blanks the keys).
  - `AgoraTokenBuilder` is a port of Agora's AccessToken2 / `RtcTokenBuilder2::buildTokenWithUid` with the publisher role. Agora doesn't publish it to Packagist, so no new dependency.
  - **Live check:** a token minted from the real keys joined an Agora channel from the Web SDK. A tampered token and a token-less join were both refused, so the project is in secure mode.
  - Tables: `tele_sessions` (one per visit, random `tele_…` channel, encrypted media key and salt for `aes_128_gcm2`, `started_at`, `ended_at`, `ended_reason`, `duration_seconds`), `tele_session_participants` (each join/leave), and `video_call_events` (NCS events handled once).
  - `POST /doctor/appointments/{id}/call-token`:
    - Only for the assigned dentist, from 15 minutes before to 60 minutes after an accepted visit, or at any time once it is in progress (`409` otherwise).
    - The first call starts the visit (`in_progress`, in the status history), sets `started_at` and sends the patient `TeleCallReadyNotification` (`tele_call_ready`, generic text) instead of the generic "Visit started".
  - `POST /patient/appointments/{id}/call-token` with `current_state`:
    - For the managing account (including a parent for a dependent), in the same window.
    - `409` unless the dentist is licensed in `current_state`.
    - Joining before the dentist is allowed: `started_at` is null while the patient waits.
  - Response: `provider`, `app_id`, `channel_name`, `uid` (the account id), `token`, `expires_at` (15 minutes, `AGORA_TOKEN_TTL_SECONDS`), `role` (`host`/`patient`), `call_mode`, `encryption` {`mode`, `key`, `salt`}, `started_at`. Apps call again before `expires_at` and renew the SDK token; the channel and encryption stay the same for the visit.
  - Every token issue writes `PhiAccessLog`.
  - Completing or cancelling the visit ends the session (`ended_reason` `completed`/`cancelled`), and no new tokens are issued. Tokens already issued run out within the TTL.
  - `POST /api/v1/agora/webhook`: verifies `Agora-Signature-V2` (HMAC-SHA256 with `AGORA_NCS_SECRET`, `400` otherwise) and handles each `noticeId` once. Join events 103/105/107 and leave events 104/106/108 write participants for the visit's dentist and patient; other uids are ignored. 102 ends the session (`channel_closed`).
  - Tests: `TeleCallTest` (24), `Unit/AgoraTokenBuilderTest` (3), and the two session key columns added to `EncryptedColumnsTest`. 801 Pest tests pass; PHPStan max and Pint clean. OpenAPI refreshed.
  - **Still to do outside the code:** turn on NCS in the Agora console for `https://<api host>/api/v1/agora/webhook` (needs a public host or a tunnel), subscribe it to events 102–108, and put its secret in `AGORA_NCS_SECRET`. Until then, sessions record start and end from the visit, but not who joined when.
  - Not done here: closing the Agora channel from the server when a visit ends (needs Agora's REST customer key). Tokens expiring within 15 minutes covers it for now.
- [x] T3 Patient booking flow — built 2026-11-10 on TEETHi-Patient-App `feat/teledentistry-booking` (`dddd29d`), not merged yet. Needs the backend branches above.
  - Book tab: third card **Teledentistry**, "From $45 · video or audio" from the catalog's virtual treatment. Copy says routine and emergency are in person, teledentistry is a call in the app.
  - Route as built: consult → dentist → day and time → insurance → Before we book → payment → confirmation (see §3.1 for why the time follows the dentist).
  - **Consult step:** Video call / Audio call, the six reason chips, "Describe it briefly" (sent as the encrypted `problem_description`), and the state the patient will be in (51 USPS codes, prefilled from the profile address).
  - **Dentist step:** `GET /patient/providers?visit_type=tele&licensed_state=XX&treatment_ids[]=…`. No map, distance or in-person fee. Empty state: "No dentists licensed in {state} offer teledentistry yet." Changing the state after picking a dentist drops that dentist.
  - **Before we book:** Video/Audio badge, "Virtual consultation · {reason}", "Call type: Audio call in the app", no address and no phone number. Booking sends `visit_type=tele`, `call_mode`, `consult_topic`, `patient_state` and no `address`. Insurance and payment are the same steps as routine.
  - **Consent fix:** consent forms now carry `type`. The teledentistry consent blocks only a tele booking. Before this, since T1 seeded it, an unsigned tele consent also blocked routine bookings in the app.
  - **Catalog fix:** the virtual consultation is left out of the routine and emergency pickers and the routine "From" price. Backend (`3ab9a7a` on `feat/teledentistry-calls`): a dentist's `starting_fee` leaves it out too; before, every tele dentist showed "From $45".
  - **Confirmation:** "Join the call from Visits when it is time. We will notify you."
  - **Visits and Home:** a tele visit shows "Audio/Video call in the app" and a Teledentistry badge instead of "At your address". Cancel and Reschedule use the existing rules and reason sheet (Q4). The `tele_call_ready` push opens and refreshes Visits.
  - English and Arabic strings.
  - Tests: `test/features/teledentistry_booking_test.dart` (16). 180 Flutter tests pass; `flutter analyze` shows only the 17 old deprecation notes.
  - **Verified 2026-11-10** in the web preview against the dev backend: booked an audio second-opinion consultation with a California dentist (pay later). The backend stored `visit_type=tele`, `call_mode=audio`, `consult_topic=second_opinion`, `patient_state=CA`, no address. The tele consent blocked the booking until signed; Visits and Home showed it as an audio call.
  - **Left for T4:** the Join call button and the call screen.
  - **Small follow-up:** the Consent forms screen's banner still counts an unsigned teledentistry consent as needed "before a visit can be confirmed", which isn't true for in-person visits.
- [x] T4 Patient call screen — built 2026-11-10 on TEETHi-Patient-App `feat/teledentistry-call` (`98529a9`, stacked on `feat/teledentistry-booking`), not merged yet. Needs the backend branches above.
  - **Backend addition** (TEETHi-Backend `f75bd61` on `feat/teledentistry-calls`): `AppointmentResource` returns `call_opens_at` and `call_closes_at` for tele visits (15 minutes before to 60 after; null otherwise), from the same `Appointment::callWindow()` that `isCallOpen()` uses. Two new tests in `TeleCallTest`; 804 Pest tests pass, PHPStan clean, OpenAPI refreshed.
  - **Visits card:** a tele visit that is accepted or in progress has **Join call** (mic or camera icon) on its own full-width row above Cancel and Reschedule. Three buttons in one row cut "Reschedule" off on a phone, so this differs from the design's single row. Join is enabled inside the window, or any time once the dentist has started. Before the window it says "You can join 15 minutes before the visit." The card redraws once when the window opens or closes (a local timer at that moment; nothing is fetched).
  - **Join sheet:** "Where are you right now?" with the state picker, prefilled with the state given at booking (T11). Sends `current_state` to `POST /patient/appointments/{id}/call-token`; a `409` (not licensed there, not open) shows the backend's reason on the call screen.
  - **Call screen (D9):** dark, full screen. Switch camera, "TEETHi Consult" with the timer, Leave. The dentist's tile with the self view picture-in-picture (tap to swap), the visit title pill, and a dock with Mute, Stop video and People. People and Leave sheets as designed ("Your dentist's notes and any prescription will appear in Records"). No chat, sharing or recording. Audio calls show avatars, and have no Stop video or Switch camera.
  - **States:** Connecting, "Waiting for your dentist", "Your dentist left the call", Reconnecting, and a failure screen (microphone refused → Open Settings; connection lost → Try again with a fresh engine; backend refused → its message; backend on the fake provider → "Calls aren't set up on this server yet"). A refused camera joins a video call with the camera off. Android back asks to leave.
  - **Timer** from the backend's `started_at`, or from when the dentist joins if they started after the patient's token was issued.
  - **Engine:** `CallEngine` interface (`lib/core/calls`) with `AgoraCallEngine` (`agora_rtc_engine` 6.6.4, `permission_handler` 13.0.2). It turns on `aes128Gcm2` encryption with the visit's key and salt before joining, uses the communication profile, and keeps the SDK log at errors only. The token is renewed when Agora reports it is about to expire (`onTokenPrivilegeWillExpire`), not on a timer.
  - **Refresh:** leaving refreshes only the Visits view model (CLAUDE.md rule).
  - **Logging:** the app's debug request log now redacts the `encryption` block, as it already did the token.
  - **Android build:** the app compiles against SDK 37 (`permission_handler_android` needs it; minSdk and targetSdk unchanged), and the root `build.gradle` sets `ext.compileSdkVersion = 37` because Agora's module otherwise builds against 31 and fails the AAR metadata check. `flutter build apk --debug` succeeds.
  - **Native:** Android `RECORD_AUDIO`, `CAMERA`, `MODIFY_AUDIO_SETTINGS`, `BLUETOOTH_CONNECT`. iOS `NSMicrophoneUsageDescription`, a camera text that mentions consultations, and the `audio` background mode.
  - **Web:** Join call opens a sheet saying calls work in the mobile app.
  - English and Arabic strings.
  - Tests: `test/features/teledentistry_call_test.dart` (22: the window, the token request, the view model against a fake engine, and the screens). 202 Flutter tests pass; `flutter analyze` shows only the 17 old deprecation notes.
  - **Verified 2026-11-10** in the web preview against the dev backend: test visit 23 (moved to accepted) showed Join call enabled inside the window and disabled with the hint outside it; on web, Join opened the "mobile app" sheet. **Not yet run on a phone:** no device was connected, so the Agora join, encryption, permissions and the call screen on a device are only covered by the fake-engine tests. That run needs the dentist side (T5) or a second client, and belongs to T8.
  - **Left open:**
    - iOS: `permission_handler` needs `PERMISSION_CAMERA=1` and `PERMISSION_MICROPHONE=1` in the Podfile's `GCC_PREPROCESSOR_DEFINITIONS`. The repo has no `ios/Podfile` yet (it is created on the first Mac build), so add them then; without them iOS reports both as refused.
    - A call that keeps going with the app in the background on Android needs a foreground service (`FOREGROUND_SERVICE_MICROPHONE` / `_CAMERA`); not built. The screen also doesn't keep the display awake.
    - Join call is on the Visits card only; Home's next-visit card links to Visits as before.
- [x] T5 Dentist app — built 2026-11-10 on TEETHI-Dentist-App `feat/teledentistry` (`8243242`, from `master`), not merged yet. Needs the backend branches above; no backend changes were needed.
  - Built in a separate git worktree (`.worktrees/dentist-tele`, ignored by the root repo), because the main dentist checkout has uncommitted work in the same files (profile, schedule, home, strings).
  - **Visits:** tele visits map `call_mode`, `consult_topic`, `patient_state`, `call_opens_at`/`call_closes_at`. Request cards, the request detail and Home show "Video/Audio call in the app" with a Teledentistry badge, the reason and "Patient in CA" instead of an address or distance. This closes T1's note that the dentist app showed tele visits as home visits with a blank address. Reschedule offers tele slots.
  - **Requests > Accepted:** an accepted tele visit has **Start call** (mic or camera icon) instead of Start visit. It is enabled inside the server's window, with "You can start the call 15 minutes before the visit" before it and "The time to start this call has passed" after it; the card redraws once when the window opens or closes (a local timer, nothing fetched). An in-progress tele visit has **Rejoin call** and Continue visit. The documentation screen also has Rejoin call until the visit is completed.
  - **Call screen as host (Q5):** the same D9 design and `CallEngine`/`AgoraCallEngine` as the patient app (encryption with the visit's key and salt, token renewal on Agora's warning). `POST /doctor/appointments/{id}/call-token` starts the visit on the first call. States: waiting for the patient (who got the "call ready" push), patient left (can rejoin), reconnecting, failure screen. Leaving says the visit isn't over, then opens the existing Complete visit flow (§7); from Rejoin it returns to the documentation. Leaving refreshes only the Requests view model.
  - **Web:** Start call and Rejoin call only show "Video and audio consultations work in the TEETHi mobile app", before any token request (which would start the visit).
  - **Schedule:** a third row, **Teledentistry** ("Video and audio calls in the app. Needs Virtual Consultation in your services."), editing `tele` windows like the others. While tele hours are on and no licensed states are set, it warns and links to the states screen.
  - **Profile > Credentials:** a Teledentistry group with the licensed states, and a **Licensed states** screen (search, one tick per state, `PATCH /doctor/profile { licensed_states }`).
  - **Logging:** the debug request log redacts the `encryption` block.
  - **Native:** the same as the patient app: Android `RECORD_AUDIO`, `CAMERA`, `MODIFY_AUDIO_SETTINGS`, `BLUETOOTH_CONNECT`, compileSdk 37 (root `extra["compileSdkVersion"]` for Agora's module); iOS microphone and camera texts and the `audio` background mode. `flutter build apk --debug` succeeds.
  - English and Arabic strings.
  - Tests: `test/features/teledentistry_test.dart` (24: mapping, the window, the token request, the host view model against a fake engine, the call screen, request detail and Accepted cards, the web guard). The live contract test checks the tele fields, tele hours and slots, and licensed states read-only (asking for a token would start a visit). 242 Flutter tests pass, and the live contract tests pass for two seeded dentists. `flutter analyze` adds only three `prefer_initializing_formals` notes, the same style as every other view model.
  - **Verified 2026-11-10** in the web preview against the dev backend, as the dentist of test visit 23: Home and Requests showed it as an audio call with the reason and state; Start call was enabled inside the window and, on web, showed the mobile-app message without a token request (the visit stayed accepted); Schedule showed the seeded tele hours; Licensed states saved CA + NV to the backend (then set back to CA). **Not run on a phone:** the call itself, between both apps on two devices, is T8.
  - **Left open:** the same as T4 for the dentist app (iOS Podfile `PERMISSION_CAMERA`/`PERMISSION_MICROPHONE` once there is a Podfile, an Android foreground service for calls in the background, keeping the screen awake). The schedule note says tele hours need the Virtual Consultation service, but doesn't check it.
- [x] T6 Admin dashboard — built 2026-11-10 on TEETHi-Frontend `feat/teledentistry` (`73ef12b`, from `master`), not merged yet. Needs TEETHi-Backend `d3b20e2` on `feat/teledentistry-calls`.
  - **Backend addition** (`d3b20e2`): `GET /admin/appointments` takes `filter[visit_type]` (`at_home`, `emergency`, `tele`). Admin appointment rows (the directory and `GET /admin/patients/{id}/appointments`) carry `call` from the new `TeleSessionResource`: `started_at`, `ended_at`, `ended_reason`, `duration_seconds` and `participants` (`role`, `joined_at`, `left_at` for each join). It never includes the channel name or the media key and salt. Other endpoints don't load the session, so `call` is absent there. Three new tests in `AppointmentManagementTest`; 807 Pest tests pass, PHPStan and Pint clean, OpenAPI refreshed.
  - **Appointments:** a visit-type filter and a Type column (a Teledentistry badge with Video call / Audio call under it). A **Call** column: "Not started", "In progress", or the duration with why it ended (visit completed, visit cancelled, everyone left), then each dentist and patient join and leave in the admin's time. "Joins not recorded" until NCS is set up (T0), since joins come only from Agora's webhook.
  - **Patient detail → Appointments:** the same Type and Call columns. Permission stays `view_patients` for that tab and `view_appointments` for the directory (decision #12); no new permission.
  - **Treatment catalog:** a Teledentistry (`is_virtual`) checkbox with a hint, and a Teledentistry flag in the list. This closes T1's note that the form lacked the toggle.
  - English and Arabic strings.
  - Tests: `visit-cells.spec.ts` (4) and new cases in the appointments list and service specs; 212 dashboard tests pass and `ng build` is clean.
  - **Verified 2026-11-10** in the dev preview as the seeded admin: the Teledentistry filter listed only test visit 23, shown as "Teledentistry / Audio call" with "Not started" in the Call column; the Virtual Consultation showed the Teledentistry flag, and its edit form opened with the box ticked (closed without saving). The badge first used dark text at about 3.9:1 contrast and now uses white (about 4.7:1, WCAG AA). A filled call log (duration, joins) was checked only in tests, since no call has run yet (T8).
- [ ] T8 Security review and live run

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
- [ ] T3 Patient booking flow
- [ ] T4 Patient call screen
- [ ] T5 Dentist app
- [ ] T6 Admin dashboard
- [ ] T8 Security review and live run

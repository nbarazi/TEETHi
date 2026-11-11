# TEETHi Dentist App — Architecture

Companion Flutter app to `TEETHi-Patient-App`, built from the Claude Design
mockup `DentistApp.dc.html`. Mirrors the patient app's stack and layering —
`provider` for state, `dio` for networking, `get_it` for DI, plain
imperative `Navigator` (no go_router), feature-first Clean Architecture —
so the two codebases stay structurally interchangeable. See
`PATIENT_APP_ARCHITECTURE.md` (next to this file) for the fuller rationale behind these
conventions; this file only tracks what's specific to the dentist app.

## Layering

```
lib/
  core/        network (ApiClient), config, storage (token), error (Result/Failure), di, enums (GET /enums),
               utils, push (FCM, push_handler.dart), app_settings (maintenance / update gate),
               calls (CallEngine + AgoraCallEngine), location
  theme/       AppColors / AppSpace+AppRadius / AppText / AppTheme — copied verbatim from the patient app
  widgets/     shared design-system widgets — copied verbatim except app_tab_bar.dart and tab_entrance.dart,
               which were generalized off the patient app's AppState-specific versions
  shell/       DentistShellState (tab index) + DentistShell (bottom nav + NavBar chrome), the RootShell equivalent
  l10n/        English and Arabic (gen-l10n), with a locale controller
  features/
    auth/ home/ requests/ patients/ patient_detail/ schedule/ dentist_profile/
    expertise/ documents/ messages/ visit_workflow/ calls/
    common/              cross-feature domain entities (Patient, Visit, ClinicalNote, ToothChart, TreatmentPlanItem)
```

Every feature follows: `domain/repositories` (abstract interface) →
`data/repositories/<feature>_repository_impl.dart` (calls the shared
`ApiClient`, with JSON mapping in `data/mappers/`) →
`presentation/providers/<feature>_view_model.dart` (`ChangeNotifier`) →
`presentation/screens`. Each feature's `injector.dart` registers its pieces
with `get_it`; tests swap in fakes there.

## Feature status

Every feature is wired to TEETHi-Backend. The mock repositories and
`MockDentistData` were removed during Phases 1–10 (2026-09-20 to 09-21).
The phase-by-phase record, with what each phase verified, is in
`docs/plans/DENTIST_APP_FRONTEND_PLAN.md` §15; open work is in
`docs/MASTER_ROADMAP.md`.

| Feature | Backend endpoints (main) |
|---|---|
| auth | `/register`, `/login`, `/me`, email/phone verify, password reset, `DELETE /account`; onboarding writes `PATCH /doctor/profile`, `POST /doctor/specializations` |
| home | `GET /doctor/dashboard` |
| requests | `/doctor/requests/open`, `/pending`, `/accepted`, accept/decline/start, shared reschedule/cancel, `/doctor/requests/{id}/medical-history` |
| patients, patient_detail | `GET /doctor/patients`, `/doctor/patients/{id}`, private notes |
| schedule | `/doctor/availability` (incl. `tele`), `/doctor/availability/open-slots`, exceptions |
| dentist_profile | `/doctor/profile` (incl. `licensed_states`), photo, address, specializations |
| expertise | `/treatments`, `/doctor/services` |
| documents | `/doctor/agreements`, `/licenses`, `/liability-insurance`, `/nondiscrimination` |
| messages | `/conversations`, `/notifications` |
| visit_workflow | clinical notes, treatment plan + tooth chart, prescriptions, recommendations, visit outcome (with final `treatments`) |
| calls | `POST /doctor/appointments/{id}/call-token` (Agora, teledentistry) |

## Navigation

No go_router. `main.dart`'s `_AppGate` watches `AuthViewModel.phase`
(`login`/`register`/`onboarding`/`verifyEmail`/`verifyPhone`/`app`) and mounts either an
auth screen or `DentistShell`. `DentistShell` owns the 5-tab bottom nav
(Home / Schedule / Requests / Patients / Profile — `DentistShellState`) via
a cross-fading `_FadeTabSwitcher`; anything else (Patient Detail, Documents,
Messages, Expertise) is a `Navigator.push(AppPageRoute(...))`.

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
  core/        network (ApiClient), config, storage (token), error (Result/Failure), di, enums, utils
  theme/       AppColors / AppSpace+AppRadius / AppText / AppTheme — copied verbatim from the patient app
  widgets/     shared design-system widgets — copied verbatim except app_tab_bar.dart and tab_entrance.dart,
               which were generalized off the patient app's AppState-specific versions
  shell/       DentistShellState (tab index) + DentistShell (bottom nav + NavBar chrome), the RootShell equivalent
  features/
    auth/               real backend — the only feature wired to TEETHi-Backend
    common/              cross-feature domain entities (Patient, Visit, ClinicalNote, ToothChart,
                          TreatmentPlanItem) + MockDentistData, the shared dummy-data pool every
                          other feature's mock repository reads from
    home/ requests/ patients/ patient_detail/ schedule/
    dentist_profile/ expertise/ documents/ messages/
```

Every feature other than `auth` follows: `domain/repositories` (abstract
interface) → `data/repositories/mock_<feature>_repository.dart` (returns
canned `Result.success(...)` data from `common/data/mock_dentist_data.dart`
after a short delay) → `presentation/providers/<feature>_view_model.dart`
(`ChangeNotifier`) → `presentation/screens`. Swapping in a real backend
later is a one-file change per feature: implement `<Feature>Repository`
against a real `<Feature>RemoteDataSource` and point that feature's
`injector.dart` at it instead of the mock.

## Feature status

| Feature | Backend | Notes |
|---|---|---|
| auth | **Real** — TEETHi-Backend `/register` (`role: doctor`), `/login`, `/me`, email/phone verify, password reset | Register collects account details only. A doctor-only `AuthPhase.onboarding` step follows it (registration no., date of birth, city, specialties) and writes `PATCH /doctor/profile` + `POST /doctor/specializations` via `DoctorOnboardingRepository` — the city/specialty catalogs need the token register issues, hence a separate step |
| home, requests, patients, patient_detail, schedule, dentist_profile, expertise, documents, messages | Dummy | `Mock<Feature>Repository` per feature, per the table above |

## Navigation

No go_router. `main.dart`'s `_AppGate` watches `AuthViewModel.phase`
(`login`/`register`/`verifyEmail`/`verifyPhone`/`app`) and mounts either an
auth screen or `DentistShell`. `DentistShell` owns the 5-tab bottom nav
(Home / Schedule / Requests / Patients / Profile — `DentistShellState`) via
a cross-fading `_FadeTabSwitcher`; anything else (Patient Detail, Documents,
Messages, Expertise) is a `Navigator.push(AppPageRoute(...))`.

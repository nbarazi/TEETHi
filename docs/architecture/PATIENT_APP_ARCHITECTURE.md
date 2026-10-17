# Clean Architecture migration — done together with the backend wiring

This app is moving from one flat `lib/screens` + single `AppState`
`ChangeNotifier` (all mock data) to a feature-first Clean Architecture, at the
same time it gets wired to the real backend. The two are the same piece of
work: `docs/plans/PATIENT_APP_FRONTEND_PLAN.md` (in the root docs repo — the
source of truth for the API contract, business rules, and phase order) builds
the networking layer from scratch starting at its Phase 0, so that's the
natural point to also introduce proper layering instead of bolting HTTP calls
onto the existing mock structure.

**Decisions made (2026-09-13):**
- **State management**: keep `provider` (`ChangeNotifier`-based view models per
  feature) — already a dependency, no rewrite of the widget tree's
  `context.watch`/`context.select` usage.
- **DI**: `get_it` as a plain service locator for data sources, repositories,
  and use cases.
- **Error handling**: a hand-rolled `Result<T>` / `Failure` pair
  (`lib/core/error/`) instead of a functional package — every repository
  method returns `Future<Result<T>>`, callers `switch`/`fold` on it instead of
  try/catch. `Failure` subtypes map directly onto the four API envelope shapes
  in the wiring plan's §1 (`AppFailure`, `ValidationFailure`,
  `UnauthorizedFailure`, `RateLimitFailure`, `NetworkFailure`, `UnknownFailure`).

## Folder convention

```
lib/
  core/                     # cross-feature, no business logic of its own
    config/api_config.dart      # base URL per build target
    di/injector.dart            # get_it setup — configureDependencies()
    error/failure.dart          # Failure subtypes
    error/result.dart           # Result<T> = Success<T> | ResultFailure<T>
    network/api_client.dart     # Dio + interceptor (Authorization, Accept-Language) + envelope parsing
    storage/token_storage.dart  # secure bearer-token persistence
  features/
    <feature>/
      data/
        datasources/    # e.g. auth_remote_data_source.dart — raw ApiClient calls, returns DTOs
        models/         # DTOs with fromJson/toJson — replaces lib/models/models.dart per-feature
        repositories/   # implements the domain repository interface, returns Result<T>
      domain/
        entities/       # plain Dart classes the UI/use-cases work with, no JSON knowledge
        repositories/   # abstract interfaces (what domain/use-cases depend on)
        usecases/       # one class per action (e.g. LoginUseCase), the only thing presentation calls
      presentation/
        providers/      # ChangeNotifier view models, injected via `locator`
        screens/         # moved from lib/screens/* as each feature migrates
        widgets/          # feature-local widgets only; cross-feature UI stays in lib/widgets, lib/theme
  screens/, widgets/, theme/, utils/   # existing shared UI — left in place; only the
                                        # feature-specific screens move out as each phase lands
  models/models.dart, data/mock_data.dart   # deleted incrementally, per §13's mapping table,
                                              # as each feature's real data layer replaces it
  state/app_state.dart                       # deleted incrementally — each feature's own
                                              # provider replaces the slice of AppState it owns
```

`lib/theme`, `lib/widgets` (the generic ones — buttons, cards, nav bar, glass,
etc.) and `lib/utils` are genuinely cross-feature UI, not architecture layers;
they stay where they are rather than moving into `core/` or being duplicated
per feature.

## How this maps onto the wiring plan's phases

The phase list below is `docs/plans/PATIENT_APP_FRONTEND_PLAN.md` §14, reframed so
each phase lands inside the new folder structure instead of editing the
existing flat files in place. Scope and order are unchanged from that
document — only *where* the code goes changes.

| Wiring plan phase | Clean Architecture framing |
|---|---|
| Phase 0 — API foundation | **Split in two**: (1) `lib/core/*` scaffolding — done in this pass (`ApiClient`, `TokenStorage`, `Result`/`Failure`, `injector.dart`); (2) `GET /enums` caching becomes its own tiny `lib/core/enums/` cache, added once a feature first needs an enum label |
| Phase 1 — Auth & session | Becomes the **first vertical slice**: `lib/features/auth/{data,domain,presentation}`, replacing `lib/screens/auth_screens.dart` + the auth-related slice of `AppState`. This is the reference implementation every later feature copies the pattern from — build it fully (including `TokenStorage` wiring for "keep me signed in") before starting Phase 2 |
| Phase 2 — Profile & family | Done: `lib/features/profile/` — replaces `PatientProfile`/`FamilyMember` in `models.dart` and the corresponding `AppState` methods. **Split off as "Phase 2b"**: Insurance and Payment methods are separate REST resources, not sub-fields of `PatientProfileResource` — give each its own `lib/features/insurance/` and `lib/features/payment_methods/` slice rather than cramming both into `profile/` |
| Phase 3 — Home dashboard | `lib/features/dashboard/` — one `GET /patient/dashboard` call per §16 |
| Phase 4 — Doctor directory | `lib/features/doctors/` — replaces the mock `Doctor` class |
| Phase 5 — Booking flow | `lib/features/booking/` — the existing `lib/screens/booking/*_step.dart` files move into `presentation/screens/` one at a time as each step is wired |
| Phase 6 — Appointment management | Folds into `lib/features/booking/` (shares `AppointmentResource`/`treatment_ids` etc.) or a sibling `lib/features/visits/` if the two flows diverge enough to warrant separating — decide when Phase 6 starts |
| Phase 7 — Records tab | `lib/features/records/` |
| Phase 8 — Messaging | `lib/features/messaging/` |
| Phase 9 — Notifications & push | `lib/features/notifications/` |
| Phase 10 — Polish & hardening | Add: delete `lib/models/models.dart`, `lib/data/mock_data.dart`, and `lib/state/app_state.dart` entirely once every feature above has migrated off them (they should be unused dead code by this point, not just "polished") |

## What "done" looks like per feature

Each feature phase is done only when:
1. `lib/features/<feature>/data|domain|presentation` all exist and compile.
2. The corresponding screen(s) moved out of `lib/screens/` into
   `presentation/screens/` (not duplicated — the old file is deleted).
3. The corresponding mock model/AppState slice named in §13's mapping table is
   deleted, not left dangling as unused dead code.
4. The feature's own `register<Feature>Dependencies(locator)` is added to
   `lib/core/di/injector.dart`'s `configureDependencies()`.

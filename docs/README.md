# TEETHi — Shared Docs (source of truth)

This folder is the **single source of truth** for every document shared across the TEETHi projects: the implementation plans, the API contract and the cross-project conventions. This was decided on 2026-10-17 (consistency report §6, decision 2, option a).

The workspace root `D:\Projects\TEETHi Project` is its own git repo (`main`). It tracks only `docs/`, plus its `.gitignore`, which excludes the four project folders. Each project is a separate git repo:

| Project | Folder | Stack |
|---|---|---|
| Backend API | `TEETHi-Backend/` | Laravel 13, Sanctum, Pest |
| Admin dashboard | `TEETHi-Frontend/` | Angular 21 (CoreUI) |
| Patient app | `TEETHi-Patient-App/` | Flutter, provider + dio + get_it |
| Dentist app | `TEETHI-Dentist-App/` | Flutter, same stack as the patient app |

## Contents

| File | What it is |
|---|---|
| [CONSISTENCY_REPORT.md](CONSISTENCY_REPORT.md) | Status of each project and the issues between them (2026-10-17) |
| [plans/IMPLEMENTATION_PLAN.md](plans/IMPLEMENTATION_PLAN.md) | Backend plan: data model, API surface, locked decisions, phase tracker |
| [plans/PATIENT_APP_FRONTEND_PLAN.md](plans/PATIENT_APP_FRONTEND_PLAN.md) | Patient app API contract and phase tracker |
| [plans/DENTIST_APP_FRONTEND_PLAN.md](plans/DENTIST_APP_FRONTEND_PLAN.md) | Dentist app API contract and phase tracker |
| [plans/ADMIN_DASHBOARD_FRONTEND_PLAN.md](plans/ADMIN_DASHBOARD_FRONTEND_PLAN.md) | Admin dashboard API contract and phase tracker. Merged from the two copies that had diverged. |
| [api/openapi.json](api/openapi.json) | Generated OpenAPI 3.1 spec, the machine-readable contract for all clients |
| [architecture/PATIENT_APP_ARCHITECTURE.md](architecture/PATIENT_APP_ARCHITECTURE.md) | Flutter layering conventions shared by both apps |
| [architecture/DENTIST_APP_ARCHITECTURE.md](architecture/DENTIST_APP_ARCHITECTURE.md) | Dentist-app specifics. Its feature-status table is out of date; see the report, M9. |

## Rules

1. **Edit docs only here.** The old locations still exist, but each one now holds only a short pointer back to this folder:
   - `TEETHi-Backend/docs/*.md`
   - `TEETHi-Frontend/docs/ADMIN_DASHBOARD_FRONTEND_PLAN.md`
   - `TEETHi-*-App/docs/ARCHITECTURE.md`

   `TEETHi-Backend/AGENTS.md` and `TEETHi-Frontend/AGENTS.md` point here as well.
2. **Track progress here.** When a project commit completes a plan item, tick it and add the "Verified" note here. Commit that in this root repo, naming the project and its commit hash in the message, e.g. `Patient Phase 0: pagination helper (TEETHi-Patient-App 82a0509)`.
3. **A contract change touches both sides.** If a backend change alters an endpoint, update every client plan that documents that endpoint in the same root commit. The admin password-reset breakage (report H1) happened because only one copy was updated.
4. **`api/openapi.json` is generated, not hand-edited.** Backend code is its real source. Refresh it after backend API changes. This needs the backend's database to be running.
   ```bash
   cd TEETHi-Backend && php artisan scramble:export --path=../docs/api/openapi.json
   ```
   `TEETHi-Backend/api.json`, the backend's own export target in `config/scramble.php`, still exists as a build artifact. Treat the copy here as the shared one.

## Not yet done
- No repo here has a remote yet: not this root repo, and not any of the four project repos. Everything exists only on this machine. Add private remotes and push, including one for this docs repo (for example `teethi-docs`).

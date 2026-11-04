# TEETHi workspace

This folder holds four independent git repos plus a docs-only root repo. Run git inside each sub-repo for code. The root repo tracks only `docs/`, `CLAUDE.md`, `.mcp.json` and `.claude/`.

| Repo | Stack | Notes |
|------|-------|-------|
| `TEETHi-Backend/` | Laravel 13 API (Sanctum, Pest 4, Larastan, Pint, Rector, Scramble) | Conventions in its `AGENTS.md` |
| `TEETHi-Frontend/` | Angular 21 admin dashboard (Vitest) | Conventions in its `AGENTS.md` |
| `TEETHi-Patient-App/` | Flutter patient app | |
| `TEETHI-Dentist-App/` | Flutter dentist app | Has `test/contract` API contract tests |

All clients call `http://127.0.0.1:8000/api/v1`. Dev servers are defined in `.claude/launch.json`.

## Docs are canonical here

- `docs/plans/` is the only copy of the plans. Tick items and add Verified notes there, never in per-repo copies. Commit them in the root repo with the sub-repo commit hash in the message, e.g. `Record X as merged (TEETHi-Backend 91a92ce)`.
- Check `docs/CONSISTENCY_REPORT.md` for known cross-project issues before starting work.
- `docs/api/openapi.json` is generated. Refresh it from `TEETHi-Backend/` with `php artisan scramble:export --path=../docs/api/openapi.json`.

## Tooling gotchas

- `php` on PATH is a 0-byte stub. Use `C:/Users/ASUS/.config/herd-lite/bin/php.exe` for artisan, Pest and Composer (`php.exe C:/ProgramData/ComposerSetup/bin/composer.phar <cmd>`).
- The `laravel-boost` MCP server (`.mcp.json`) exposes the backend's schema, routes, logs, tinker and Laravel docs search.

## App behaviour rules

- No polling or blanket reloads to keep screens fresh. Refresh on a foreground push (each Flutter app's `core/push/push_handler.dart`) or after the user's own successful action, refreshing only the affected view model.

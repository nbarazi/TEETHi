# Teethi Admin Dashboard — Angular Frontend Implementation Plan

> **Consolidated copy (2026-10-17).** This file merges the two diverged copies: the progress/"Verified" notes from `TEETHi-Frontend/docs/` plus the two backend contract updates that only existed in `TEETHi-Backend/docs/` (§2 OTP-code password reset; §4.9 managing-account email/phone fields). The dashboard's reset-password fix (`token` → `code`) is merged into TEETHi-Frontend `master` (`b2882f6`); see `docs/CONSISTENCY_REPORT.md` H1.

This document tells an Angular engineer/agent **what exists in the Teethi backend today, exactly how to call it, and in what order to build the admin dashboard screens against it.** It intentionally contains no Angular/TypeScript code or framework guidance — that's an implementation detail left to whoever builds it. What follows is the API contract, the business rules behind it, and a phase-tracked scope.

**Source of truth for the backend itself**: `docs/plans/IMPLEMENTATION_PLAN.md`, next to this file in the root docs repo. This document covers the backend phases that are **actually finished and merged** as of this writing (Phases 0–9). Backend Phase 6 (clinical documentation — tooth charts, clinical notes, treatment plans, prescriptions) is almost entirely **not admin-facing** by design (HIPAA minimum-necessary scoping keeps it doctor/patient-only) — the one exception, `prescription_templates` admin CRUD, is covered below (§4.6). Backend Phase 7 (payments/invoices), Phase 8 (messaging/notifications), and Phase 9 (admin dashboard aggregation & reporting) shipped on 2026-09-13/14/15 and are covered below in §4.7–§4.13 — this is where the admin dashboard's most obviously "admin" screens live: a searchable directory of patients, doctors, and appointments, invoice reconciliation, city management, and the PHI access audit log. Backend Phases 10–11 (HIPAA hardening close-out, real payment gateway integration) are **not built yet** — do not build UI against endpoints that don't exist. Phase 11 in particular does not change any API shape the dashboard sees (it only swaps the payment gateway behind `reconcile`), so nothing here is expected to move because of it; Phase 10 may add a "log out all devices" endpoint and tighten error envelopes (§1.1), which this doc should be revisited for once it ships.

**How to use this file**: work phase by phase, top to bottom. Check items off (`- [x]`) as they land, and add a short "Verified" note and any design decisions under each phase, the same way `docs/plans/IMPLEMENTATION_PLAN.md` tracks the backend. If a backend detail below turns out to be wrong or has changed, fix this doc in the root docs repo as part of the same change that discovers it.

---

## 1. API fundamentals

- **Base path**: `http://localhost/api/v1/...` in local dev (`APP_URL` is `http://localhost`; there is no separate public-facing host configured yet). All admin routes are under `/api/v1/admin/...`; a few routes the admin shares with the other two apps live at the top level (`/api/v1/...`) — called out explicitly below wherever relevant.
- **Auth transport**: Laravel Sanctum **personal access tokens**, not cookie/session-based SPA auth. There is no CSRF cookie step, no `withCredentials`, no `/sanctum/csrf-cookie` call. Every authenticated request sends `Authorization: Bearer <token>`. Tokens **do not expire** (`config/sanctum.php` has `'expiration' => null`) — a token is valid until the user logs out (`POST /logout`, deletes just that token) or changes/resets their password (deletes **all** of that user's tokens, forcing re-login everywhere).
- **Locale**: send an `Accept-Language: en` or `Accept-Language: ar` header on every request to control the language of `message` strings and any `label`-bearing fields the backend returns. Unrecognized values silently fall back to the app's default locale. Fetch enum display labels from `GET /enums` (§5) rather than hardcoding them — it returns the exact catalog for whichever locale you send.
- **Rate limits** (a `429` is returned past these): `POST /login` — 5/min per IP. Every other authenticated route the admin dashboard uses — 120/min per authenticated user id.

### 1.1 Response envelopes — there are four distinct shapes

The backend has one exception-handling gap worth knowing up front: **only success responses and a few hand-written error responses go through the app's custom `{success, message, data}` envelope.** Validation errors and "not authenticated at all" errors currently fall through to **Laravel's own default JSON error shape**, not the custom one. An HTTP client / error interceptor must handle all four:

**a) Success (200/201)**
```json
{ "success": true, "message": "Human-readable, localized string", "data": { /* varies by endpoint, or null */ } }
```

**b) Success, paginated list (200)** — every list/index endpoint uses this. `data` is always the array of items; `meta`/`links` are Laravel's standard paginator shape:
```json
{
  "success": true,
  "message": "...",
  "data": [ /* array of resource objects */ ],
  "meta": {
    "current_page": 1, "from": 1, "to": 15, "per_page": 15, "last_page": 4, "total": 52,
    "path": "http://localhost/api/v1/admin/treatments",
    "links": [ { "url": null, "label": "&laquo; Previous", "active": false }, "..." ]
  },
  "links": { "first": "...?page=1", "last": "...?page=4", "prev": null, "next": "...?page=2" }
}
```
Every list endpoint accepts `?per_page=N` (default 15, clamped to a max of 100) and `?page=N`.

**c) Hand-written application error** (403 "forbidden", 404 "not found", 409 "conflict"/invalid state transition, and a handful of 400/422s written explicitly in a controller):
```json
{ "success": false, "message": "Human-readable, localized string" }
```
Some of these add an `errors` object (e.g. the booking "missing requirements" gate in the patient app — not currently relevant to the admin dashboard, but the shape can recur):
```json
{ "success": false, "message": "...", "errors": { "some_key": ["..."] } }
```

**d) Laravel's default shapes (not the custom envelope) — these WILL appear and must be handled separately:**
- **422 validation failure** (any `FormRequest` rejecting the payload):
  ```json
  { "message": "The given data was invalid.", "errors": { "field_name": ["This field is required."] } }
  ```
- **401 — no/invalid/expired bearer token** (Sanctum's own `auth:sanctum` guard failing, before any app code runs):
  ```json
  { "message": "Unauthenticated." }
  ```
- **400 — an unknown `filter[...]` or `sort=` parameter on a Phase 7/9 list endpoint** (the `spatie/laravel-query-builder` endpoints: `GET /admin/invoices` §4.7, `/admin/appointments` §4.8, `/admin/patients` §4.9, `/admin/doctors` §4.10). Sending a filter/sort key the endpoint doesn't allow is rejected, it isn't silently ignored:
  ```json
  { "message": "Requested filter(s) `foo` are not allowed. Allowed filter(s) are `status, patient_profile_id`." }
  ```
  Only send filter/sort keys this doc lists per endpoint. The older admin list endpoints (§4.1–§4.6) don't use this library and ignore unknown query params instead.
- Three role/permission/active-account middleware checks (below) DO use the custom envelope even though they're also "auth-adjacent" — they run after `auth:sanctum` succeeds, as app-level middleware:
  - Not the right role (not `role=admin`): `403 { "success": false, "message": "You do not have permission to access this resource." }`
  - Missing dashboard permission: `403 { "success": false, "message": "You do not have permission to perform this action." }`
  - Account deactivated (`users.status = inactive`): `403 { "success": false, "message": "Your account is inactive. Please contact support." }`

**No content (204)**: every `DELETE` and a couple of other endpoints return a bare `204` with an empty body — don't try to parse JSON from these.

---

## 2. Authentication & session

| Endpoint | Purpose | Auth required |
|---|---|---|
| `POST /login` | `{ email, password }` → `{ user: UserResource, token: string }` on success | No |
| `POST /forgot-password` | `{ email }` → sends a 6-digit reset code by email (OTP-based as of 2026-09-13 — not a mail link) | No |
| `POST /reset-password` | `{ email, code, password, password_confirmation }` | No |
| `GET /me` | Returns the current `UserResource` | Yes |
| `POST /logout` | Deletes the current token only | Yes |
| `POST /change-password` | `{ current_password, new_password, new_password_confirmation }`; deletes **all** tokens for the user (forces re-login) | Yes |

Notes:
- **There is no admin self-registration.** `POST /register` exists but its `role` field only accepts `patient` or `doctor` — an admin/staff account can only be created by another admin via `POST /admin/staff` (§4.2), or via the database seeder (`admin@teethi.test`, seeded as the standing Super Admin). Do not build a "sign up" screen for the dashboard.
- `login` returns `403 { success: false, message: "..." }` (the hand-written envelope) if the account's `status` is `inactive`, and the hand-written `401` unauthorized envelope for wrong credentials (this one specific case does use the custom envelope, since it's written explicitly in `AuthController`, not thrown by the guard).
- **`UserResource` (what the top-level `/me` and `/login` return) does not include the staff member's dashboard roles or permissions.** That's expected, not a gap — it's the endpoint shared by all three apps. The admin-specific `GET /admin/me` (§3.3) is the one to call right after login for that.

`UserResource` shape:
```json
{
  "id": 1, "name": "Admin User", "email": "admin@teethi.test",
  "role": "admin", "status": "active", "profile_image": null,
  "email_verified_at": "2026-09-01T00:00:00+00:00",
  "created_at": "...", "updated_at": "..."
}
```

---

## 3. The RBAC model (read this before building anything past login)

Two layers stack on top of each other for every `/admin/*` route, and the frontend needs to reason about both:

1. **`role: admin`** — the coarse platform gate. Every `/admin/*` route requires the logged-in user's `role` to be `admin`. This is fixed at account-creation time and never changes via the dashboard.
2. **Dashboard permission** (a route-level `permission:{slug}` check) — a specific admin account can be `role=admin` and still get a `403` from a specific screen if none of their assigned dashboard **roles** grant that permission. This is the actual "what can this staff member do" layer, and it's what the admin dashboard's own Roles/Permissions/Staff screens (§4) manage.

**`roles` vs `permissions`, precisely:**
- A **permission** is a fixed, developer-defined string slug (e.g. `manage_roles`, `view_roles`, `manage_staff`, `manage_treatment_catalog`, `manage_document_templates`, `verify_legal_documents`). There is **no "create a permission" UI** — the full catalog only grows as new backend phases add their own `/admin/*` endpoints and register a new slug. `GET /admin/permissions` (§4.1) is the read-only way to list the current catalog for building a role-editor's checkbox list.
- A **role** (e.g. "Billing", "Support") is an admin-defined bundle of permissions. A staff account can hold **multiple** roles; their effective permission set is the union of every permission across every role they hold.
- **`roles.is_super_admin`** is a boolean flag on a role, not a permission. Any staff account holding a role with this flag bypasses **every** `permission:` check in the system, unconditionally. The seeded `admin@teethi.test` account holds the standing "Super Admin" role.
- The **Super Admin role itself cannot be edited or deleted by anyone**, including another super-admin (`RolePolicy::delete`/`update`) — it's a structural fixture, not an ordinary row. The dashboard's role-edit/delete UI should expect a `403` on this specific role and should probably just disable those actions in the UI once it sees `is_super_admin: true` on a role, rather than let the user hit the error.
- **Escalation guards the frontend should expect (as validation errors, not silent failures)**: a non-super-admin `manage_roles` holder gets a `422` if they try to create/edit a role with `is_super_admin: true`, or if they try to sync roles to a staff member that includes a super-admin role. Only an actual super-admin may do either.

### 3.1 Current permission catalog (as of backend Phase 9)

| Slug | Grants | Introduced in |
|---|---|---|
| `view_roles` | Read-only access to `GET /admin/permissions` only (**not** the roles list itself — see §4.1) | Phase 2 |
| `manage_roles` | Full CRUD on roles + syncing a role's permissions, **and** viewing/listing roles (there is no separate "view roles" access — reading and writing roles require the same permission) | Phase 2 |
| `manage_staff` | Create staff accounts, view a staff account, assign roles to a staff account | Phase 2 |
| `manage_treatment_catalog` | Full CRUD on treatment categories and treatments | Phase 3 |
| `manage_document_templates` | List + publish consent-form/agreement document templates | Phase 5 |
| `verify_legal_documents` | List, verify, or reject doctor-submitted legal documents (licenses, liability insurance) | Phase 5 |
| `manage_prescription_templates` | Full CRUD on reusable prescription templates | Phase 6 |
| `view_invoices` | View the invoice list (`GET /admin/invoices`) and individual invoices | Phase 7 |
| `manage_invoices` | Reconcile an invoice (`POST /admin/invoices/{invoice}/reconcile`) — record a collected pay-later amount, refund, or correct status | Phase 7 |
| `view_appointments` | View the platform-wide appointment list (`GET /admin/appointments`), filterable by date range/status/doctor/patient | Phase 9 |
| `view_patients` | View the patient directory and each patient's tabs (profile, appointments, medical history, prescriptions, medical reports, invoices) | Phase 9 |
| `manage_patients` | Activate/deactivate a patient's account (`PATCH /admin/patients/{patient}/status`) | Phase 9 |
| `view_doctors` | View the doctor directory (`GET /admin/doctors`, `GET /admin/doctors/{doctor}`) | Phase 9 |
| `manage_doctors` | Activate/deactivate a doctor's account (`PATCH /admin/doctors/{doctor}/status`) | Phase 9 |
| `manage_cities` | Manage the city catalog (`GET`/`PATCH /admin/cities`) and review city requests (`GET /admin/city-requests`) | Phase 9 |
| `view_phi_access_logs` | Review and export the PHI access audit log (`GET /admin/phi-access-logs`, `.../export`) | Phase 9 |

Fetch this list live from `GET /admin/permissions` rather than hardcoding it — it grows every time a later backend phase ships. Note the actual Phase 9 slugs differ slightly from what an earlier draft of this plan predicted: there is **no** `manage_appointments` (the admin appointment view is read-only — status changes still go through the doctor/patient workflow endpoints), and the patient-directory read permission is `view_patients` (one slug, covering every tab), not a per-tab set.

### 3.2 What a `403` from a permission check looks like

```json
{ "success": false, "message": "You do not have permission to perform this action." }
```
There is no machine-readable "which permission was missing" field — the dashboard can only know from context (which button/screen the user was on) which permission to point them at.

### 3.3 `GET /admin/me` — the account's own roles/permissions

This is the endpoint the dashboard's navigation/menu should be built on. Unlike every other `/admin/*` route, it has **no `permission:` gate** at all — only `role:admin` — specifically so a staff account holding zero dashboard permissions can still discover that fact, rather than being locked out of the one call that would tell them so.

```
GET /admin/me
```
```json
{
  "success": true,
  "message": "...",
  "data": {
    "id": 5, "name": "Jane Staff", "email": "jane@teethi.test", "status": "active",
    "is_super_admin": false,
    "permissions": ["manage_treatment_catalog", "manage_staff"],
    "roles": [ /* RoleResource[] — same shape as §4.1, each with its own nested permissions; mainly useful for a "your role(s)" display in account settings, not for menu-building */ ]
  }
}
```

**Read `is_super_admin` first, unconditionally, before ever looking at `permissions`.** The seeded "Super Admin" account has `is_super_admin: true` and an **empty** `permissions` array — the standing Super Admin role carries no individual permission rows of its own; it bypasses every `permission:` check purely via the flag. Treating `permissions.includes(...)` as the only signal will incorrectly hide every screen from the actual super-admin account. The correct check for "can this account do X" is always:

```
is_super_admin === true  OR  permissions.includes("the_specific_slug")
```

Call this once after login (and cache the result for the session — it only changes when someone else edits this account's role assignments, which doesn't happen mid-session), and drive every nav-item/route-guard/action-button visibility check off it. There's no need for the defensive "let the API call fail and catch the 403" fallback this plan originally called for — build the menu directly from this endpoint's `is_super_admin`/`permissions` pair.

---

## 4. Admin API reference, by feature area

All endpoints below are under `/api/v1/admin/...` unless marked "shared". All require `Authorization: Bearer <token>` for a `role: admin` account, plus the `permission:` noted per group.

### 4.1 Roles & Permissions

| Method & path | Purpose | Body | Permission |
|---|---|---|---|
| `GET /admin/roles` | Paginated list, each with its permissions eager-loaded | — | `manage_roles` |
| `POST /admin/roles` | Create a role | `{ name, slug, description?, is_super_admin? }` | `manage_roles` |
| `GET /admin/roles/{role}` | Single role + its permissions | — | `manage_roles` |
| `PATCH /admin/roles/{role}` | Update a role's own fields (not its permission list — see next row) | same fields as create, all optional | `manage_roles` |
| `DELETE /admin/roles/{role}` | Delete a role (blocked with `403` if `is_super_admin`) | — | `manage_roles` |
| `POST /admin/roles/{role}/permissions` | **Replace** (sync, not append) the role's permission set | `{ permission_ids: number[] }` — send the **full** desired list every time, including an empty array to clear it | `manage_roles` |
| `GET /admin/permissions` | Read-only catalog for building the role-editor's checkbox list | — | `view_roles` |

`slug` must be unique, `alpha_dash` (letters/numbers/dashes/underscores only) — validate this client-side too before submitting, since the server 422 is generic ("has already been taken" / "format invalid").

`RoleResource`:
```json
{
  "id": 2, "name": "Billing", "slug": "billing", "description": "Invoicing staff",
  "is_super_admin": false,
  "permissions": [ { "id": 4, "name": "manage_treatment_catalog", "description": "..." } ],
  "created_at": "...", "updated_at": "..."
}
```
`PermissionResource`: `{ "id": 1, "name": "manage_roles", "description": "..." }`.

### 4.2 Staff accounts — `permission: manage_staff`

For the acting account's *own* roles/permissions (no `manage_staff` needed), use `GET /admin/me` instead — see §3.3. Everything below is for managing *other* staff accounts.

| Method & path | Purpose | Body |
|---|---|---|
| `GET /admin/staff` | Paginated list of `role=admin` accounts, with roles eager-loaded | — |
| `POST /admin/staff` | Provision a new dashboard account. Email is marked verified immediately (no OTP flow — this isn't self-registration) and `status` is set `active` | `{ name, email, password, password_confirmation }` |
| `GET /admin/staff/{staff}` | Single staff account, with roles **and their permissions** eager-loaded | — |
| `PATCH /admin/staff/{staff}` | Update `name`/`email`/`status` (partial — send only what changed) | `{ name?, email?, status? }` |
| `POST /admin/staff/{staff}/roles` | **Replace** (sync) the staff member's role assignments | `{ role_ids: number[] }` — full desired list, empty array clears all |

- `PATCH` covers both "edit profile" and "deactivate/reactivate" in one endpoint (`status: "active"` / `"inactive"`, §5) — there's no separate `.../status` sub-route for staff, unlike the doctor/patient status toggles sketched for a later backend phase.
- **There is no `password` field on this endpoint.** Resetting another staff member's password is intentionally left to the existing `POST /forgot-password` flow (addressed to *that* person's email), not something this screen can do directly — don't build a "set new password" field here.
- **A staff account can never deactivate itself.** Submitting `status: "inactive"` where `{staff}` is the acting account's own id returns a `422` (`{"errors": {"status": ["You cannot deactivate your own account."]}}` — the Laravel-default validation shape, §1.1d). This is a real, expected rejection the UI should surface clearly (e.g. disable the "Deactivate" action on your own row in the staff list, rather than let the user hit the error) — not a bug to work around. Deactivating *any other* account, including another super-admin, works normally.

`StaffResource`:
```json
{
  "id": 5, "name": "Jane Staff", "email": "jane@teethi.test", "status": "active",
  "roles": [ /* RoleResource[], each with its own nested permissions */ ],
  "created_at": "..."
}
```

Escalation rule to surface in the UI (as a disabled checkbox or an inline note, not just a caught error): assigning a role where `is_super_admin: true` to a staff member via this endpoint returns `422` unless the *acting* user is themselves a super-admin.

### 4.3 Treatment catalog — `permission: manage_treatment_catalog`

Two related resources, standard REST CRUD on both, no soft delete (a hard `DELETE` — confirm before calling).

| Method & path | Purpose | Body |
|---|---|---|
| `GET /admin/treatment-categories` | Paginated | — |
| `POST /admin/treatment-categories` | Create | `{ name, group, sort_order?, is_active? }` |
| `GET /admin/treatment-categories/{treatmentCategory}` | Single | — |
| `PATCH /admin/treatment-categories/{treatmentCategory}` | Update | any subset of the above |
| `DELETE /admin/treatment-categories/{treatmentCategory}` | Delete → `204` | — |
| `GET /admin/treatments` | Paginated, each with its category eager-loaded | — |
| `POST /admin/treatments` | Create | `{ treatment_category_id, name, description?, standard_amount, is_quick_select?, requires_tooth_chart?, is_active? }` |
| `GET /admin/treatments/{treatment}` | Single | — |
| `PATCH /admin/treatments/{treatment}` | Update | any subset of the above |
| `DELETE /admin/treatments/{treatment}` | Delete → `204` | — |

`group` is one of the two `TreatmentCategoryGroup` enum values (§5). `standard_amount` is a decimal (AED), sent/received as a plain number/string, max `99999999.99`. `created_by` on a treatment is set server-side from the acting admin — never send it.

`TreatmentCategoryResource`: `{ "id": 1, "name": "Preventative Dentistry", "group": "preventative", "sort_order": 1, "is_active": true }`

`TreatmentResource`:
```json
{
  "id": 10, "treatment_category_id": 1,
  "category": { /* TreatmentCategoryResource, only present when the list/show eager-loads it (it always does here) */ },
  "name": "Teeth Cleaning", "description": "...", "standard_amount": "250.00",
  "is_quick_select": true, "requires_tooth_chart": false, "is_active": true,
  "created_at": "...", "updated_at": "..."
}
```

### 4.4 Document templates — `permission: manage_document_templates`

**A template is never edited in place.** There is no `PATCH`/`PUT`/`DELETE` for this resource at all — "editing" a template's wording is done by `POST`ing a brand-new version. The backend auto-assigns the next `version` number for that `type` and, when the new one is created with `is_active: true` (the default), automatically deactivates whichever version of that same `type` was previously active. **The UI should present this as "Publish a new version" or "Edit" (which under the hood posts a new version pre-filled with the old one's content), never as an in-place form save.**

| Method & path | Purpose | Body |
|---|---|---|
| `GET /admin/document-templates` | Paginated, optionally `?type=consent_form` (or `business_agreement` / `nondiscrimination_notice`) | — |
| `POST /admin/document-templates` | Publish a new version | `{ type, audience, title, body, is_active?, effective_at? }` |

- `is_active` defaults to `true` server-side if omitted. Set it to `false` explicitly to prepare a draft version without deactivating the current live one (useful for a "save draft" action distinct from "publish").
- `effective_at` defaults to "now" if omitted.
- There's no delete — a superseded/inactive version stays in the list forever (for audit purposes) and should probably render as read-only/greyed-out once a newer version exists for its type.

`DocumentTemplateResource` (as returned from this admin endpoint — `signed`/`signed_at` are always `null` here, since "signed by whom" only makes sense from the patient/doctor sign-flow endpoints, not the admin catalog view):
```json
{
  "id": 3, "type": "consent_form", "audience": "both", "title": "Patient Consent Form",
  "body": "Full legal text...", "version": 2, "is_active": true,
  "effective_at": "2026-09-11T00:00:00+00:00",
  "signed": null, "signed_at": null,
  "created_at": "..."
}
```

### 4.5 Legal documents (credential review queue) — `permission: verify_legal_documents`

This is the review queue for doctor-submitted `dental_license` and `liability_insurance` uploads (and, incidentally, lists every signed consent form/agreement too, since they share the same table — filter by `?type=` to separate them in the UI).

| Method & path | Purpose | Body |
|---|---|---|
| `GET /admin/legal-documents` | Paginated. Filter with `?status=submitted` (the actual review queue), or `?type=dental_license` / `liability_insurance` / `consent_form` / `business_agreement` / `nondiscrimination_notice` | — |
| `PATCH /admin/legal-documents/{legalDocument}/verify` | Approve a `submitted` document | — (no body) |
| `PATCH /admin/legal-documents/{legalDocument}/reject` | Reject a `submitted` document | `{ reason }` (required, max 500 chars) |

- Both `verify` and `reject` return **`409`** (the hand-written envelope, message key resolves to "This document cannot be moved to that status from its current status") if the document isn't currently `status: submitted` — e.g. it was already verified/rejected, or it's a `signed` consent form (which never goes through this workflow at all, only credential uploads do). **The UI should only show Verify/Reject buttons for rows where `status === "submitted"`**, not rely on the 409 as the primary guard.
- There is also a **shared, non-admin-prefixed** endpoint any authenticated user (including admin) can use to fetch one specific document's full detail before deciding: `GET /api/v1/legal-documents/{id}` (note: no `/admin` prefix). Useful for an "open this submission" detail view/modal ahead of the verify/reject decision.

`LegalDocumentResource`:
```json
{
  "id": 12, "type": "dental_license", "status": "submitted",
  "document_template_id": null, "on_behalf_of_patient_profile_id": null,
  "metadata": { "license_number": "LIC-1001", "issuing_authority": "Dubai Health Authority", "expiration_date": "2027-01-01" },
  "file_url": "http://localhost/storage/...(short-lived signed URL, ~30 min TTL, regenerated on every fetch — never cache/store this URL, always re-fetch the resource to view/download the file)",
  "submitted_at": "...", "verified_at": null, "rejection_reason": null, "expires_at": null,
  "version": 1, "supersedes_id": null, "created_at": "..."
}
```
`metadata`'s shape depends on `type`: for `dental_license` it's `{license_number, issuing_authority, expiration_date}`; for `liability_insurance` it's `{insurer_name, policy_number, coverage_amount, expiration_date}`; it's `null` for the three template-based types (`consent_form`, `business_agreement`, `nondiscrimination_notice`), which carry a `signature_attestation` instead (not currently exposed on this resource — only relevant to the patient/doctor sign-flow endpoints, not to admin review).

### 4.6 Prescription templates — `permission: manage_prescription_templates`

The one admin-facing piece of backend Phase 6 (clinical documentation) — everything else in that phase (clinical notes, tooth charts, treatment plans, prescriptions themselves) is doctor/patient-only by HIPAA minimum-necessary design and has no admin endpoint at all. A template here is reusable boilerplate *text*, not tied to any patient — ordinary catalog CRUD, the same shape as §4.3's treatment catalog.

| Method & path | Purpose | Body |
|---|---|---|
| `GET /admin/prescription-templates` | Paginated | — |
| `POST /admin/prescription-templates` | Create | `{ name, body, is_active? }` |
| `GET /admin/prescription-templates/{prescriptionTemplate}` | Single | — |
| `PATCH /admin/prescription-templates/{prescriptionTemplate}` | Update | any subset of the above |
| `DELETE /admin/prescription-templates/{prescriptionTemplate}` | Delete → `204` | — |

`created_by` is set server-side from the acting admin — never send it.

`PrescriptionTemplateResource`: `{ "id": 1, "name": "Amoxicillin 500mg", "body": "Take 1 tablet 3 times daily for 7 days.", "is_active": true, "created_at": "..." }`

---

### 4.7 Invoices (backend Phase 7) — `permission: view_invoices` / `manage_invoices`

Backend Phase 7 built the payment/invoicing domain. Most of it is patient-facing (saved cards, paying for an appointment); the **admin** surface is exactly two endpoints — a filterable invoice list and a reconciliation action. There is no "create an invoice" endpoint anywhere: an invoice is generated lazily by the backend the first time a patient initiates payment on an appointment (including choosing "pay later"), derived from that appointment's attached treatments.

| Method & path | Purpose | Body | Permission |
|---|---|---|---|
| `GET /admin/invoices` | Paginated list, each with its `payments` eager-loaded. Filter: `?filter[status]=`, `?filter[patient_profile_id]=`, `?filter[doctor_profile_id]=`. Sort: `?sort=standard_amount` / `-standard_amount` / `created_at` / `-created_at` (default `-created_at`) | — | `view_invoices` |
| `POST /admin/invoices/{invoice}/reconcile` | Record an out-of-band correction to an invoice | `{ status, amount_collected? }` | `manage_invoices` |

- **`GET /admin/invoices` uses `spatie/laravel-query-builder`** — filters are `?filter[key]=value`, sorts are `?sort=key` (prefix `-` for descending). This is the query-string convention for the four Phase 7/9 directory list endpoints (§4.7 invoices, §4.8 appointments, §4.9 patients, §4.10 doctors — but *not* §4.11 cities or §4.12 PHI logs, which don't use the library). An unknown `filter[]`/`sort` key returns `400` (§1.1d), not a silent ignore.
- **`reconcile` body**: `status` is required, one of the `invoice_status` enum values (§5 — `pending`, `partially_paid`, `paid`, `refunded`, `cancelled`). `amount_collected` is optional (`numeric`, `min: 0.01`) — send it when recording a real cash/transfer collection against a `pay_later` invoice, which creates a `succeeded` payment row for that amount and moves the invoice toward `paid`. Setting `status: "refunded"` triggers a genuine gateway refund of the invoice's most recent successful card/Apple-Pay payment (only that one payment's amount, not the whole balance) and appends a `refunded` payment row. Every other status value is applied directly — this is an admin correction tool, not a state machine, so there's no "invalid transition" `409` here.
- **`reconcile` returns the updated `InvoiceResource`** (with `payments` reloaded), not a `204` — show the recomputed `remaining_amount`/`status`/new payment row back to the user.
- **There is no per-appointment "get this appointment's invoice" endpoint.** `AppointmentResource` defines an `invoice` field, but no current endpoint eager-loads that relation, so it never actually appears in a response today (it's `whenLoaded`-guarded and would be masked from a doctor viewer regardless). For an "invoice for appointment X" panel, call `GET /admin/invoices?filter[patient_profile_id]=...` and match on `appointment_id`, or link out to the patient's invoices tab (§4.9). If a future backend change starts loading it, this field will begin appearing for admin viewers automatically.

`InvoiceResource`:
```json
{
  "id": 7, "appointment_id": 42,
  "treatment_description": "Teeth Cleaning, Composite Filling",
  "standard_amount": "450.00", "advance_paid": "200.00", "remaining_amount": "250.00",
  "status": "partially_paid",
  "due_at": "2026-09-20T00:00:00+00:00",
  "payments": [ /* PaymentResource[], only when eager-loaded (the admin endpoints always load it) */ ],
  "created_at": "..."
}
```
`PaymentResource`:
```json
{
  "id": 15, "invoice_id": 7, "amount": "200.00",
  "method": "credit_card",          // payment_method_type enum (§5): apple_pay | credit_card | pay_later
  "provider_reference": "fake_ch_9c1e...", "status": "succeeded",   // payment_status enum (§5)
  "paid_at": "2026-09-13T10:04:00+00:00", "created_at": "..."
}
```
- **Money is always a decimal string** (`"450.00"`), not a JS number — parse it deliberately, don't `+` it.
- **Invoices/payments are not PHI-access-logged** — §4.12's PHI category list is explicit and doesn't include billing records, so viewing an invoice writes no `phi_access_logs` row (unlike the patient clinical tabs in §4.9).

### 4.8 Appointments directory (backend Phase 9) — `permission: view_appointments`

A single **read-only** platform-wide appointment list. The admin dashboard cannot change an appointment's state (accept/decline/reschedule/cancel/complete all stay in the doctor and patient workflow endpoints) — this is a monitoring/search view only.

| Method & path | Purpose | Permission |
|---|---|---|
| `GET /admin/appointments` | Paginated list, each with `treatment`, `patient_profile`, `doctor_profile` eager-loaded | `view_appointments` |

Query params (`spatie/laravel-query-builder`, same convention as §4.7):
- `?filter[status]=` — one `appointment_status` value (§5: `open`, `pending`, `accepted`, `in_progress`, `completed`, `rescheduled`, `cancelled`, `declined`)
- `?filter[doctor_profile_id]=` , `?filter[patient_profile_id]=` — exact id match
- `?filter[scheduled_from]=2026-01-01` , `?filter[scheduled_to]=2026-02-01` — **inclusive** `scheduled_at` range bounds; accepts any string `Carbon` can parse (`YYYY-MM-DD` or a full ISO timestamp). Send either, both, or neither.
- `?sort=` — `scheduled_at`, `requested_at`, or `created_at` (each with an optional `-` prefix). Default sort is `-requested_at` (newest requests first).

Returns `AppointmentResource` (the same appointment shape used across all three apps). This list endpoint only eager-loads `treatment`/`patientProfile`/`doctorProfile`, so `insurance`/`visit_address` come back absent (not `null`) here — fetch the shared `GET /api/v1/appointments/{appointment}` (an admin token is authorized for any appointment) for the fuller detail view, which additionally loads `visitAddress` and `insurance` (an admin viewer is not subject to the doctor-viewer masking of `insurance`). `doctor_profile` is `null` on an `open`-status (open-pool) appointment that no doctor has claimed yet.

### 4.9 Patients directory (backend Phase 9) — `permission: view_patients` / `manage_patients`

"One endpoint per tab, not one giant payload" — a patient detail screen with tabs (Profile / Appointments / Medical history / Prescriptions / Medical reports / Invoices) fetches only the tab currently open. All seven read endpoints require `view_patients`; the status toggle requires `manage_patients`.

| Method & path | Purpose | Body |
|---|---|---|
| `GET /admin/patients` | Paginated directory. Filter: `?filter[first_name]=` / `?filter[last_name]=` (both **partial**, case-insensitive substring), `?filter[city_id]=` (exact), `?filter[is_account_holder]=1` / `=0`. Sort: `?sort=first_name` / `last_name` / `created_at` (± prefix; default `-created_at`). Each row eager-loads `city` and `manager` (so `account_status` is populated) | — |
| `GET /admin/patients/{patient}` | One patient — eager-loads `city`, `address`, `manager`. **Writes a `phi_access_logs` row** (`action: viewed`, `resource_type: App\Models\PatientProfile`) | — |
| `GET /admin/patients/{patient}/appointments` | That patient's appointments (paginated, `treatment` + `doctor_profile` loaded, newest `requested_at` first) — `AppointmentResource[]` | — |
| `GET /admin/patients/{patient}/medical-history` | The patient-reported `MedicalHistoryResource`, or `data: null` if they've never filled one in. **PHI-logged** (`resource_type: App\Models\MedicalHistory`) | — |
| `GET /admin/patients/{patient}/prescriptions` | Paginated `PrescriptionResource[]`, newest `prescribed_at` first. **PHI-logged** (`resource_type: App\Models\Prescription`) | — |
| `GET /admin/patients/{patient}/medical-reports` | Paginated `ClinicalNoteResource[]` — the doctor-authored clinical notes across **every** appointment this patient has had (this is what "medical reports" means here, as distinct from the patient-reported history above). **PHI-logged** (`resource_type: App\Models\ClinicalNote`) | — |
| `GET /admin/patients/{patient}/invoices` | Paginated `InvoiceResource[]` (with `payments`), newest first. Not PHI-logged (billing, §4.7) | — |
| `PATCH /admin/patients/{patient}/status` | Activate/deactivate the patient's login account | `{ status }` — `"active"` or `"inactive"` (`user_status` enum) |

- **`{patient}` is a `PatientProfile` id, not a `User` id.** A family account has one `PatientProfile` per person it manages; each is addressed by its own profile id.
- **`PATCH /admin/patients/{patient}/status` only works on an account-holder profile.** A dependent profile (`is_account_holder: false`) has no login of its own, so toggling its status returns **`422`** — `{ "message": "...", "errors"? ... }` with a body whose message resolves to *"This patient has no account to activate or deactivate."* (`lang/*/admin.php` key `no_account_to_toggle`). The UI should hide/disable the activate-deactivate control for any row where `is_account_holder` is `false`, and only surface it for account holders. On success it returns the updated `PatientProfileResource` (with `account_status` reflecting the new value).
- **`account_status`, `email`, `phone_number`, `is_email_verified`, `is_mobile_verified`** on `PatientProfileResource` all come from the *managing account* (`users.status`/`email`/`phone_number`/`email_verified_at`/`phone_verified_at`), read through the `manager` relation — a dependent has no login of their own, so "this patient's account" always refers to the account holder who manages them. These fields are only present on responses from this directory (the endpoints here eager-load `manager`); every other place `PatientProfileResource` appears omits them.
- **Every "show"/tab endpoint except appointments and invoices writes a PHI access-log row** as a side effect of the admin viewing it — this is deliberate HIPAA instrumentation, and those rows are exactly what §4.12's audit-log viewer surfaces. No client action needed; just be aware the read is recorded.

`PatientProfileResource` (directory/detail shape — clinically-sensitive fields like `date_of_birth`, `profile_note` are present because this is an authorized `view_patients` admin context):
```json
{
  "id": 88, "is_account_holder": true, "relationship_to_account_holder": null,
  "title": null, "first_name": "Amina", "middle_name": null, "last_name": "K.", "suffix": null,
  "date_of_birth": "1990-04-12", "profile_note": null, "languages": ["en", "ar"],
  "email": "amina@example.com", "phone_number": "+9715...",
  "is_email_verified": true, "is_mobile_verified": false,
  "city": { /* CityResource, when loaded */ }, "address": { /* AddressResource, on the detail endpoint */ },
  "account_status": "active",
  "profile_completed_at": "...", "missing_fields": [], "created_at": "...", "updated_at": "..."
}
```
`email`/`phone_number`/`is_email_verified`/`is_mobile_verified` were added to the backend resource on 2026-09-13 (both `GET /admin/patients` and `GET /admin/patients/{patient}` now return them) — surfaced on the patients list (email column) and the patient detail Profile tab (both fields plus a verified/unverified badge each).
`ClinicalNoteResource` (the medical-reports tab):
```json
{
  "id": 5, "appointment_id": 42, "doctor_profile_id": 9,
  "chief_complaint": "...", "clinical_finding": "...", "recommendation": "...",
  "quick_sketch_transcript": null, "additional_notes": null,
  "created_at": "...", "updated_at": "..."
}
```
`MedicalHistoryResource`: `{ "id": 3, "patient_profile_id": 88, "conditions": "...", "allergies": "...", "current_medications": "...", "notes": null, "updated_by": 12, "updated_at": "..." }`
`PrescriptionResource`: `{ "id": 7, "patient_profile_id": 88, "doctor_profile_id": 9, "appointment_id": 42, "prescription_template_id": null, "medication_name": "Amoxicillin", "dosage": "500mg twice daily", "instructions": "...", "prescribed_at": "..." }`

### 4.10 Doctors directory (backend Phase 9) — `permission: view_doctors` / `manage_doctors`

Simpler than the patient directory — no tabs (the original scope never described a tabbed doctor detail view), just list / show / status toggle.

| Method & path | Purpose | Body |
|---|---|---|
| `GET /admin/doctors` | Paginated directory. Filter: `?filter[first_name]=` / `?filter[last_name]=` (partial), `?filter[city_id]=` (exact). Sort: `?sort=first_name` / `last_name` / `average_rating` / `created_at` (± prefix; default `-created_at`). Each row eager-loads `city`, `specializations`, and (implicitly, for `account_status`) `user` | — |
| `GET /admin/doctors/{doctor}` | One doctor — eager-loads `city`, `address`, `specializations`, `user`. Not PHI-logged (a doctor's own professional profile isn't patient PHI) | — |
| `PATCH /admin/doctors/{doctor}/status` | Activate/deactivate the doctor's login account | `{ status }` — `"active"` / `"inactive"` |

- `{doctor}` is a `DoctorProfile` id. A doctor profile always maps 1:1 to exactly one `User`, so the status toggle has **no** "nothing to toggle" `422` case the way patients do — it always succeeds for a valid id.
- **`account_status`** on `DoctorProfileResource` is that doctor's own `users.status`, via the `user` relation. Present only on these directory endpoints (which eager-load `user`); omitted everywhere else `DoctorProfileResource` is used (e.g. the patient-app doctor search).
- On success the status toggle returns the updated `DoctorProfileResource`.

`DoctorProfileResource` (directory shape — see the patient/doctor frontend contract for the full field list; the admin-relevant addition is `account_status`):
```json
{
  "id": 9, "first_name": "Sara", "last_name": "M.", "mobile_no": "+9715...",
  "health_authority_registration_no": "DHA-12345",
  "city": { /* CityResource */ }, "address": { /* AddressResource, detail endpoint only */ },
  "account_status": "active",
  "years_experience": 12, "average_rating": 4.6,
  "specializations": [ /* SpecializationResource[] */ ],
  "profile_completed_at": "...", "missing_fields": [], "created_at": "...", "updated_at": "..."
}
```

### 4.11 Cities & city requests (backend Phase 9) — `permission: manage_cities`

The city catalog is what gates which emirates/areas the platform serves (`is_available`). Doctors can submit a request for an unlisted city from the doctor app; the admin dashboard reviews those requests (read-only) and toggles catalog availability.

| Method & path | Purpose | Body |
|---|---|---|
| `GET /admin/cities` | Paginated, ordered by `name`. `CityResource[]` | — |
| `PATCH /admin/cities/{city}` | Update a city | `{ name?, state?, is_available? }` — all optional, send only what changed |
| `GET /admin/city-requests` | Paginated list of doctor-submitted city requests, newest first. `CityRequestResource[]` | — |

- **`GET /admin/cities` here is the admin-gated one** (`permission: manage_cities`). There's also the unauthenticated-shared `GET /api/v1/cities` (§ no prefix) that every app uses to populate city dropdowns — same `CityResource`, but that one only returns `is_available: true` cities and needs no permission. Use the `/admin/` one for the management screen so deactivated cities are visible/editable.
- **City requests are read-only from the dashboard.** There is no endpoint to approve/reject/annotate a `CityRequest` row — its `status` (`pending` → `notified`) is expected to flip as a side effect of the requested city actually being added/enabled via `PATCH /admin/cities/{city}`, not via a direct action on the request. Render the list as an informational queue: "these areas have been asked for."
- `CityResource` has no `created_at`/`updated_at` — it's `{ id, name, state, is_available }` only.

`CityResource`: `{ "id": 3, "name": "Dubai", "state": "Dubai", "is_available": true }`
`CityRequestResource`: `{ "id": 8, "city_name": "Hatta", "status": "pending", "notified_at": null, "created_at": "..." }`

### 4.12 PHI access audit log (backend Phase 9) — `permission: view_phi_access_logs`

An append-only trail of every read/write/export touching patient PHI, across all three apps — this is the endpoint that surfaces the `phi_access_logs` rows written by the doctor clinical endpoints, the patient-directory tabs in §4.9, the messaging endpoints (§4.13), and more.

| Method & path | Purpose |
|---|---|
| `GET /admin/phi-access-logs` | Paginated log, newest first. `PhiAccessLogResource[]` |
| `GET /admin/phi-access-logs/export` | The same rows (respecting the same filters), **unpaginated**, capped at 5000, as a plain `data: [...]` array — for "download / review a window" |

- **Filters are plain query params here, not the `spatie` `filter[]` convention** (this endpoint predates that library's introduction into the codebase in a different form): `?action=viewed` (a `phi_access_action` value — §5: `viewed`, `created`, `updated`, `deleted`, `exported`, `downloaded`), `?resource_type=App\Models\PatientProfile` (send the exact PHP class string as it appears in `resource_type`), `?actor_id=5` (the staff/user id who performed the access). Combine freely. Unknown params are ignored, not `400`.
- **Both endpoints are rate-limited more tightly than the rest of the dashboard**: `10/min` per user (the `throttle:admin-export` limiter), not the usual `120/min`. A bulk export screen should not auto-refresh.
- **Viewing the audit log is itself audited.** Every call to `index` and `export` writes its *own* `phi_access_logs` row (`resource_type: App\Models\PhiAccessLog`, `action: viewed` / `exported`, with a `context` of `{"scope":"index"}` or `{"scope":"export","count":N}`). So the audit trail always contains a record of who reviewed it and when — expect your own reads to appear in the very next page of results, and don't treat those rows as noise to filter out by default.
- `export` returns a bare `{ success, message, data: [...] }` (no `meta`/`links`) — it is explicitly not a paginated envelope. If the cap of 5000 is hit, older rows are simply absent; narrow with the filters and re-export.

`PhiAccessLogResource`:
```json
{
  "id": 4021, "actor_id": 5, "subject_user_id": 88,
  "resource_type": "App\\Models\\PatientProfile", "resource_id": 88,
  "action": "viewed", "ip_address": "127.0.0.1",
  "context": { "scope": "index" },     // field NAMES only, never PHI values — safe to render as-is
  "created_at": "2026-09-15T09:12:44+00:00"
}
```
There is no `updated_at` (the log is insert-only) and rows can never be edited or deleted through any endpoint.

### 4.13 Messaging visibility (backend Phase 8) — no permission, `role: admin` is enough

Backend Phase 8 built patient↔doctor messaging (one thread per patient-doctor pair) plus an in-app notification feed. Almost none of it is an admin dashboard feature — but the **conversation and message list endpoints are shared across all three apps and an admin token can read them**, so an admin can inspect a thread if a screen needs it (e.g. a support/dispute view). These live at the top level (no `/admin` prefix) and, unusually, have **no `permission:` gate** — the `role: admin` platform gate alone is the whole check.

| Method & path | Admin behavior |
|---|---|
| `GET /api/v1/conversations` | An admin sees **every** conversation on the platform (paginated), `patient_profile` + `doctor_profile` eager-loaded. `unread_count` is relative to the caller, so for an admin it's effectively meaningless (they're never a participant) |
| `GET /api/v1/conversations/{conversation}/messages` | An admin can read any thread's messages (paginated, `?since=<ISO timestamp>` for polling). **Read-only for admin**: an admin viewing does *not* mark messages read (no read-receipt side effect), and… |
| `POST /api/v1/conversations/{conversation}/messages` | …**an admin cannot post.** `MessagePolicy::create` allows only the two actual participants — an admin `POST` here returns `403`. Don't build a compose box for the admin view. |
| `GET /api/v1/notifications`, `PATCH /api/v1/notifications/{id}/read` | Scoped to the caller's *own* notifications only — an admin just sees their own admin notifications, there's no "all users' notifications" view. Not useful for a dashboard beyond the acting user's own bell menu. |

- **Reading a thread's messages as an admin writes a `phi_access_log` row** (`resource_type: App\Models\Message`, `action: viewed`) — message bodies are PHI. Same instrumentation as §4.9's patient tabs.
- `MessageResource.attachment_url` is a short-lived (~30 min) signed URL onto a private disk, regenerated on every fetch — never cache it, re-fetch the message to view/download.
- Whether the dashboard should expose messaging at all is a product call; the capability exists if needed, but there is no dedicated `/admin/conversations` screen contract beyond "the shared endpoints happen to accept an admin token."

`ConversationResource`: `{ "id": 12, "patient_profile": {…}, "doctor_profile": {…}, "appointment_id": 42, "last_message_at": "...", "unread_count": 0, "created_at": "..." }`
`MessageResource`: `{ "id": 300, "conversation_id": 12, "sender_id": 88, "body": "…", "attachment_url": null, "read_at": null, "created_at": "..." }`
`NotificationResource`: `{ "id": "uuid", "data": { /* generic payload, no clinical detail */ }, "read_at": null, "created_at": "..." }`

---

## 5. `GET /enums` — the enum label catalog

```
GET /enums
```
```json
{
  "success": true, "message": "...",
  "data": {
    "user_role": { "patient": "Patient", "doctor": "Doctor", "admin": "Admin" },
    "user_status": { "active": "Active", "inactive": "Inactive" },
    "treatment_category_group": { "preventative": "Preventative Dentistry", "cosmetic": "Cosmetic Dentistry" },
    "document_template_type": { "consent_form": "Consent Form", "business_agreement": "Business Agreement", "nondiscrimination_notice": "Nondiscrimination Notice" },
    "document_audience": { "patient": "Patient", "doctor": "Doctor", "both": "Both" },
    "legal_document_type": { "consent_form": "...", "business_agreement": "...", "nondiscrimination_notice": "...", "dental_license": "Dental License", "liability_insurance": "Liability Insurance" },
    "legal_document_status": { "pending": "Pending", "signed": "Signed", "submitted": "Submitted", "verified": "Verified", "rejected": "Rejected", "expired": "Expired", "superseded": "Superseded" },
    "invoice_status": { "pending": "Pending", "partially_paid": "Partially Paid", "paid": "Paid", "refunded": "Refunded", "cancelled": "Cancelled" },
    "payment_method_type": { "apple_pay": "Apple Pay", "credit_card": "Credit Card", "pay_later": "Pay Later" },
    "payment_status": { "initiated": "Initiated", "succeeded": "Succeeded", "failed": "Failed", "refunded": "Refunded" },
    "appointment_status": { "open": "Open", "pending": "Pending", "accepted": "Accepted", "in_progress": "In Progress", "completed": "Completed", "rescheduled": "Rescheduled", "cancelled": "Cancelled", "declined": "Declined" },
    "visit_type": { "at_home": "In-Person at Home Visit", "emergency": "Dental Emergency" },
    "appointment_request_mode": { "direct": "Direct", "open_pool": "Open Pool" },
    "city_request_status": { "pending": "Pending", "notified": "Notified" },
    "phi_access_action": { "viewed": "Viewed", "created": "Created", "updated": "Updated", "deleted": "Deleted", "exported": "Exported", "downloaded": "Downloaded" },
    "...": "plus every other enum in the app (more patient/doctor-domain ones) — not just the ones the admin dashboard currently uses"
  }
}
```

- No `role:`/`permission:` gate — any authenticated user of any of the three apps can call it, same as `/cities`, `/specializations`, `/treatments`.
- Not paginated — it's the whole fixed catalog in one response, every time.
- **Respects `Accept-Language`** exactly like every other endpoint's `message` — send `ar` to get the Arabic labels instead. Fetch it once per session per locale and cache it; it only changes when a future deploy adds/renames an enum case, not per-request.
- Each top-level key is a `{ value: label }` map — directly usable as a dropdown's option list for the enum used in the corresponding resource field. The keys of the outer object match the group names used throughout this doc and `lang/{locale}/enums.php`. The ones the admin dashboard actually needs (per §4's resources): `user_role`, `user_status`, `treatment_category_group`, `document_template_type`, `document_audience`, `legal_document_type`, `legal_document_status` (§4.1–§4.6), and now `invoice_status`, `payment_method_type`, `payment_status` (§4.7), `appointment_status`, `visit_type`, `appointment_request_mode` (§4.8), `city_request_status` (§4.11), `phi_access_action` (§4.12). The response includes several more (patient/doctor-domain enums) that this dashboard has no current use for — ignore those rather than trying to render every group somewhere.
- For `legal_document_status`, only `submitted`/`verified`/`rejected`/`superseded` are reachable through any endpoint this dashboard calls today; `pending` and `expired` are schema-reserved for a later backend phase and won't appear in real data yet (§4.5).
- For `appointment_status`, all eight values can appear in real data in the §4.8 directory. `phi_access_action` in practice is almost always `viewed`, `created`, or `exported` in the §4.12 log; `updated`/`deleted`/`downloaded` are defined but rarely written by current endpoints.

---

## 6. Phased build plan

### Phase 0 — API foundation
- [x] Environment config for the API base URL (`/api/v1`) per build target (local/staging/prod)
- [x] A single place that attaches `Authorization: Bearer <token>` and `Accept-Language` to every request
- [~] Central handling for all four response shapes in §1.1 — success, paginated success, hand-written error, and Laravel's default 401/422 shapes — so every screen can rely on one consistent "did this fail, and why" model
- [x] Global handling for `401` (any source) → clear stored token, redirect to login
- [ ] Global handling for `429` → a generic "too many requests, try again shortly" surface
- [x] A pagination helper/pattern that reads `meta`/`links` consistently, since every list screen from Phase 2 onward needs it
- [ ] Fetch `GET /enums` (§5) once per session (and again if the user switches locale) and cache it; every later phase's dropdowns/labels read from this cache instead of hardcoding label text

**Verified (2026-09-11, done incidentally while building Phase 1):**
- API base URL is `environment.apiUrl` (`src/environments/environment*.ts`). Local dev points at `http://127.0.0.1:8000/api`; relative request paths pass `/v1/...`, so the effective base is `.../api/v1/...`, matching §1. The `apiUrl`/`documentUrl` split lets local/staging/prod differ without code changes.
- `src/app/core/auth/auth.interceptor.ts` is the single choke point: it prefixes the base URL, attaches `Authorization: Bearer` (when a token exists) and `Accept-Language` (from the active ngx-translate locale, default `en`), and turns any `401` into "clear session + redirect to `/login`" and any `403` into a redirect to `/unauthorized` — with `/login` exempted so the login screen renders its own `401`/`403` copy.
- `PaginatedResponse<T>` / `PaginationMeta` / `PaginationLinks` in `src/app/core/models/pagination.model.ts` already model the §1.1b shape.
- **Still open:** a typed union for the four §1.1 envelopes is only partially in place (`src/app/core/models/api-response.model.ts` covers success + hand-written error; components currently narrow Laravel's 401/422 by `HttpErrorResponse.status` inline). `429` has no dedicated surface yet (there's no global toast service). `GET /enums` caching is untouched. Do these as a dedicated Phase 0 pass before Phase 3's forms need enum labels.

### Phase 1 — Authentication & session
- [x] Login screen (`POST /login`); surface the `403 account_inactive` and `401 invalid_credentials` messages distinctly
- [x] Forgot/reset password screens (`POST /forgot-password`, `POST /reset-password`)
- [x] Change-password screen (`POST /change-password`) — warn the user this logs them out everywhere
- [x] Logout (`POST /logout`)
- [x] Route guard: block the whole dashboard shell unless `GET /me` succeeds and `role === "admin"`
- [x] No "sign up" screen — confirmed there isn't one for admin (§2)

**Verified (2026-09-11):**
- Built on the auth scaffolding inherited from the pre-TEETHi CoreUI template (`src/app/core/auth/*`, `src/app/views/pages/login/`) rather than from scratch — only the parts that disagreed with this plan were changed.
- **Login** (`views/pages/login/`): existing reactive-form screen kept; `#resolveError()` now branches on the error — `403` → `AUTH.ACCOUNT_INACTIVE`, `401` → `AUTH.INVALID_CREDENTIALS`, a client-side `NonAdminError` (see below) → `AUTH.NOT_ADMIN` — always preferring the server's localized `message` when present. On any failure it calls `clearSession(false)` so a half-authenticated token can't linger and bounce the user past `loginGuard`. The "Forgot password?" control is now a real `routerLink="/forgot-password"`.
- **Forgot / Reset password** (`views/pages/forgot-password/`, `views/pages/reset-password/`): new screens matching the login card styling. Forgot-password always shows the same neutral "if an account exists…" confirmation (no account-existence disclosure), surfacing only a `422` email-format error inline. Reset-password pre-fills `email` + `token` from the query string (`?email=&token=` — the shape a Laravel reset link uses), validates an 8-char minimum and a client-side confirm-match, renders `422` field errors per input, and on success routes to `/login?reset=1`.
- **Change password** (`views/pages/change-password/`): authenticated screen, a child of the shell layout (so `authGuard` covers it), reachable from the header user-menu ("Change password", replacing the dead "Lock account" link). Shows a standing warning that changing the password signs the user out everywhere; on success it displays a confirmation and then calls `clearSession()` after ~2s (the backend has already revoked every token). `422` errors render per field.
- **Admin-only shell** (`core/auth/auth.guard.ts`): `authGuard` now requires `isAuthenticated()` **and** `role === 'admin'`. If the stored profile already proves admin it allows synchronously; otherwise it calls `AuthService.getProfile()` and, on any failure (non-admin, expired token, network), drops the session and redirects to `/login`. `AuthService.getProfile()` fetches `GET /me`, rejects a non-admin session with the exported `NonAdminError`, saves the profile, then chains `PermissionService.fetchAdminMe()` (`GET /admin/me`, §3.3) — so one call warms both the identity and the capability cache. The bootstrap initializer (`auth.initializer.ts`) now swallows errors and lets `authGuard` be the single redirect authority.
- **No sign-up:** the inherited `register` component, its spec, and the dead duplicate `views/pages/routes.ts` were deleted; the `/register` route is gone from `app.routes.ts`.
- **RBAC realignment (also touches Phase 2/3):** `PermissionService` was calling a non-existent `GET /v1/my-permissions`. It now calls `GET /v1/admin/me` (`fetchAdminMe()`), stores `is_super_admin` + `permissions` + `roles` as signals (hydrated from a `localStorage` `admin_me` cache on construction so guards work on hard refresh), and `has()/hasAny()/hasAll()` implement the §3.3 rule `is_super_admin === true || permissions.includes(slug)`. The legacy `user.roles?.includes('admin')` fallback is gone.
- **Models:** `User` (`core/models/user.model.ts`) now matches `UserResource` (§2) — `role` (singular) + `status`, with the old `roles: string[]` demoted to a deprecated optional purely so the dormant `features/administration` module still type-checks. New `core/models/admin-me.model.ts` (`AdminMe`, `AdminRoleSummary`, `AdminPermission`) models `GET /admin/me`; new `core/models/api-response.model.ts` holds the success / hand-written-error envelope types. `core/models/role.model.ts` was left on its legacy shape (`is_admin`, `permissions: string[]`) — it and the `features/administration` module get reworked in Phase 3/4 against `/admin/roles` + `/admin/staff`.
- **i18n:** `AUTH.*` keys for every new string added to both `en.json` and `ar.json`; `NAV.REGISTER` removed, `NAV.FORGOT_PASSWORD` / `RESET_PASSWORD` / `CHANGE_PASSWORD` and `HEADER.CHANGE_PASSWORD` added.
- **Not done in Phase 1 (deferred with reason):** actual end-to-end auth against a live backend wasn't exercised — `127.0.0.1:8000` wasn't running — but `ng build` is clean, `ng test` is green (53/53; three pre-existing `provideTranslateService` spec failures fixed along the way), and the four screens plus the shell redirect were verified rendering/routing in the browser.

**Verified (2026-10-17) — password reset moved to the OTP contract (consistency report H1):** the reset screen above still sent `token` from a `?token=` link, so every reset got a `422` after the backend switched to 6-digit codes. `ResetPasswordRequest` and the reset form now carry `code` (client-side `^\d{6}$`, `autocomplete="one-time-code"`) instead of reading the URL. Forgot-password no longer shows its own confirmation. On success, and on a `404` for an unknown address (kept neutral, no account-existence disclosure), it routes to `/reset-password` with the email in router navigation state, not the query string, so the address stays out of the URL. A reload keeps it. The reset screen has a "request a new code" link, and `422` errors still render per field, including `code`. `AUTH.*` strings were updated in en/ar, and the unused `FORGOT_PASSWORD_SENT` and `RESET_TOKEN` were removed. `ng build` is clean and `ng test` passes 174/174, with 12 new forgot/reset specs covering the request body, code validation, 422 rendering and the email hand-off. Checked in the browser against `ng serve`. Not exercised against a live backend because `127.0.0.1:8000` was down.

### Phase 2 — Dashboard shell & navigation
- [x] After login (and on app bootstrap/refresh), call `GET /admin/me` (§3.3) once and cache `is_super_admin`/`permissions` for the session
- [x] Shell layout (header/sidebar) — drive every nav item's visibility off `is_super_admin === true || permissions.includes(slug)` (§3.3); a staff account with zero permissions still logs in and sees a shell, just with (almost) nothing in it
- [x] Route guards on every permission-gated section, redirecting to a clear "you don't have access to this" page rather than letting a guarded API call 403 first
- [x] Global 404/409/500 empty-state patterns, reused by every later screen (403 is now pre-empted by the nav/route guards above, but keep a generic handler for the rare case a permission is revoked mid-session)

**Verified (2026-09-11):**
- **`/admin/me` cache** — done in Phase 1 and unchanged: `AuthService.getProfile()` chains `PermissionService.fetchAdminMe()` (called from the login flow and the bootstrap initializer); `PermissionService` also hydrates `is_super_admin`/`permissions`/`roles` from a `localStorage` `admin_me` cache in its constructor, so guards have data immediately on a hard refresh.
- **Permission-driven nav** (`layout/default-layout/default-layout.component.ts`): `#buildNavItems()` → `filterNavItems()` → `PermissionService.has()` already implemented the §3.3 rule; the rebuild `effect()` now also tracks `isSuperAdmin()` and `loaded()` (not just `permissions()`), so the nav re-filters when the super-admin flag flips or `/admin/me` resolves — a super-admin has an empty `permissions` array, which the old effect would have missed. Nav items carry an optional `permission` slug; a parent group hides once all its children are filtered out. The sidebar currently has only the (ungated) Dashboard entry — feature entries are added per later phase.
- **`permissionGuard`** (`core/auth/permission.guard.ts`): reads `route.data['permission']`, allows only when `has(slug)`, else redirects to `/unauthorized`. Now async — if `/admin/me` hasn't resolved this session it `fetchAdminMe()`s first so the decision uses real data (a deep-link hard refresh no longer races). Wired onto feature routes as `canActivate: [permissionGuard], data: { permission: '<slug>' }`. Also: `authInterceptor` still catches a stray runtime `403` (permission revoked mid-session) and routes to `/unauthorized`.
- **Shared status/empty pattern** (`shared/components/status-message/`): one `<app-status-message>` component — `code` or `icon`, `title`, `description`, a built-in `[actionLabel]`/`actionLink` button or a projected `[actions]` slot, and a `page` attribute for the full-viewport centred variant vs. a compact inline block. The `404`, `500` and `403 (/unauthorized)` route pages were rebuilt on it and fully translated (the CoreUI demo "search" boxes are gone). The wildcard route `**` now renders the 404 page instead of silently redirecting to `/dashboard`. Later list/detail screens use the same component for their empty / no-access / load-error states.
- **Not done / deferred:** there is still no dedicated `429` surface (an `ERRORS.TOO_MANY_REQUESTS` string is in `en`/`ar` ready for one) and no typed 4-envelope model — both belong to the dedicated Phase 0 close-out pass. `409` has no standalone page because it's always contextual (§1.1c, an invalid state transition on a specific action) — feature screens surface it inline via `<app-status-message>` or a form error. The authenticated shell itself wasn't click-tested (needs a live backend), but `/login`, `/unauthorized`, `/404` (incl. the `**` fallback) and `/500` were verified rendering; `ng build` clean; `ng test` 56/56.

### Phase 3 — Roles & Permissions (`manage_roles` / `view_roles`)
- [x] Roles list (paginated), with each role's permission chips shown inline
- [x] Create/edit role form — disable the "super admin" toggle unless the acting account is itself a super-admin (client-side convenience; server still enforces it)
- [x] Permission-assignment UI reading its checkbox list from `GET /admin/permissions`, submitting the **full** selected set to `POST /admin/roles/{role}/permissions`
- [x] Delete role, disabled/hidden for any role where `is_super_admin === true`
- [x] Handle the `422` escalation-guard errors from §3 with a clear inline message, not a generic failure toast

**Verified (2026-09-11):**
- Built fresh under `src/app/features/roles/` against `/admin/roles` — the pre-TEETHi `features/administration/` module (Alwataniya `is_admin` / `{action}_{module}` permission grid) was **deleted** along with the legacy `core/models/role.model.ts`; a plan-shaped `Role`/`Permission`/`RoleWriteRequest` now lives in `core/models/role.model.ts` and `AdminMe.roles` reuses it.
- **`RolesService`** (`features/roles/roles.service.ts`) — `list` (page/per_page only; §4.1 note that this endpoint ignores unknown params), `get`, `create` (POST), `update` (PATCH), `remove` (DELETE → 204), `syncPermissions` (POST `{ permission_ids }`), `permissionCatalog` (`GET /admin/permissions`). All under `/v1/admin/...`.
- **List** (`roles-list.component`) — paginated table (name + description, slug as `<code>` with a `Super Admin` badge, up to 4 permission-name chips + "+N more", created date). Loading / empty / load-error states via `<app-status-message>`. Row Edit/Delete are **disabled** (not just hidden) when `is_super_admin` — the plan says disable rather than let the user hit the `403`. Delete goes through a confirm modal; success/'failure feedback via the new `ToastService`.
- **Form** (`role-form.component`, a modal opened by the list via `viewChild`) — name, slug (auto-derived from name on create until the slug field is touched; validated client-side against `^[A-Za-z0-9_-]+$` per §4.1's `alpha_dash`), description, and an `is_super_admin` checkbox that stays **disabled** unless `PermissionService.isSuperAdmin()`. Permissions are a live, filterable, select-all checkbox list from `GET /admin/permissions` (slug + description per row). Submit does the role write (`POST`/`PATCH`) then **always** a separate `syncPermissions` call with the full id set (§4.1). `422`s render inline: field errors on name/slug, and the `is_super_admin` escalation guard (§3) as an inline alert in the form — never a toast.
- **Routing / nav** — `/roles` is a shell child, `canActivate: [permissionGuard]`, `data: { permission: 'manage_roles' }` (the plan's §3.1: reading and writing roles both require `manage_roles`; `view_roles` only covers `GET /admin/permissions`). Sidebar gained an **Administration** group (hides itself when its children are all filtered) with a "Roles & Permissions" child gated on `manage_roles`.
- **New shared infra used here and by every later list/CRUD screen:** `ToastService` + a single `<app-toast-host>` mounted in `AppComponent` (replaces per-screen `<c-toaster>`); `<app-pagination-bar>` (reads Laravel `meta`, emits page / per-page, windows page numbers with first/last + ellipses).
- **Not verified against a live backend:** a Laravel API is reachable at `127.0.0.1:8000` (CORS preflight + `401` on `/api/v1/me` confirm the Phase 1 transport assumptions) but no test credentials were available, so the roles screens weren't click-tested end to end. Covered instead by unit tests: `ng build` clean; `ng test` 72/72 — including `RolesService` request shapes, list load + error state, the slug-derivation and super-admin-lock form logic, `PaginationBarComponent` windowing, and `ToastService`.

### Phase 4 — Staff management (`manage_staff`)
- [x] Staff list (paginated)
- [x] Create staff form (`name`, `email`, `password` + confirmation) — no email/OTP step, account is active immediately
- [x] Staff detail view showing assigned roles (and, transitively, their permissions)
- [x] Edit-profile form (`name`/`email`) and an Active/Inactive toggle, both via `PATCH /admin/staff/{staff}` (§4.2) — no password field, and disable/hide the deactivate toggle on the acting account's own row (§4.2's self-deactivation guard)
- [x] Role-assignment UI on the staff detail screen, submitting the full desired `role_ids` set

**Verified (2026-09-11):**
- `src/app/features/staff/` — `StaffService` (`list`/`get`/`create`(POST)/`update`(PATCH)/`syncRoles`(POST `{ role_ids }`), all `/v1/admin/staff...`); `staff-list.component` (paginated table, role chips, "New staff" opens `staff-form.component` — a create-only modal, no edit/delete here per §4.2); `staff-detail.component` at `/staff/:id`.
- **Detail screen** loads `GET /admin/staff/{id}` and `GET /admin/roles` (`per_page=100`, for the role picker) via `forkJoin`; shows a `notfound` empty state on a `404` distinct from a generic load error. Three independent save actions, each its own `PATCH`/`POST` call: (1) profile `name`/`email`; (2) status Activate/Deactivate — Deactivate goes through the new shared `<app-confirm-dialog>`, and is **disabled outright** (not just on click) when the viewed staff id equals `AuthService.currentUserId()`, per §4.2's self-deactivation guard — the hint text also explains why; (3) roles — a checkbox list of every role (pre-checked from the member's current roles) that submits the **full** `role_ids` set via `syncRoles`, with a read-only "effective permissions" chip list computed as the union of the checked roles' permissions (the plan's "assigned roles and, transitively, their permissions"). The `422` role-escalation guard (assigning an `is_super_admin` role as a non-super-admin) renders inline under the roles card, not as a toast.
- **New shared component**: `<app-confirm-dialog>` (`shared/components/confirm-dialog/`) — a generic confirm-before-acting modal (title/message/confirm+cancel labels/busy state), used here for deactivation and intended to replace the ad-hoc delete-confirm markup in later CRUD screens.
- Routed at `/staff` (list) and `/staff/:id` (detail), both under the shell, `permissionGuard` + `permission: manage_staff`. Sidebar's Administration group gained a "Staff" child.
- `ng build` clean; `ng test` 84/84 (added `StaffService` request-shape tests, list load/error, detail load + role pre-selection + effective-permissions + not-found state, form password-mismatch validation). Not click-tested against the live `127.0.0.1:8000` backend (no test credentials) — same caveat as Phase 3.

### Phase 5 — Treatment catalog (`manage_treatment_catalog`)
- [x] Treatment categories: list, create, edit, delete
- [x] Treatments: list (with category shown/filterable), create, edit, delete
- [x] Category picker on the treatment form sourced from the categories list
- [x] Confirm-before-delete on both (hard deletes, no undo)

**Verified (2026-09-11):**
- `src/app/features/treatment-catalog/` — one `TreatmentCatalogService` for both resources (`listCategories`/`createCategory`/`updateCategory`/`removeCategory`, `listTreatments`/`createTreatment`/`updateTreatment`/`removeTreatment`, all `/v1/admin/treatment-categories` / `/v1/admin/treatments`). Routed at `/treatment-catalog/categories` and `/treatment-catalog/treatments` (`/treatment-catalog` redirects to the latter); sidebar gained a "Treatment Catalog" group with both as children, gated on `manage_treatment_catalog`.
- **Categories** (`treatment-categories-list` + `category-form` modal) — table with group (translated), sort order, active badge; create/edit modal with a `group` `<select>` (`preventative`/`cosmetic` — the only two `treatment_category_group` values, hardcoded with i18n labels since `GET /enums` caching is still a deferred Phase 0 item, not yet a live source). Delete via the shared `<app-confirm-dialog>` (hard delete, no undo, per the plan).
- **Treatments** (`treatments-list` + `treatment-form` modal) — table shows the eager-loaded `category.name`, `standard_amount` rendered as-is (a decimal **string**, never coerced to a number), quick-select/tooth-chart flag badges, active badge. The category picker in the form is populated from a categories fetch the list screen already does (passed into the form's `open()`, no duplicate call). `standard_amount` is validated client-side against `^\d{1,8}(\.\d{1,2})?$` (the API's `99999999.99` max, 2 decimal places).
- **Category filter is client-side only, by design**: `GET /admin/treatments` is one of the pre-`spatie` admin list endpoints (§1.1d / §4.3) — it has no server-side filter and silently ignores unknown query params rather than 400ing, so a `?treatment_category_id=` would do nothing. The filter dropdown narrows only the currently-loaded page (default `per_page=50`, selectable up to the API's 100 max) with an explicit hint under it explaining that. This is a real, backend-imposed limitation, not an oversight — flagged in §7 below for the backend team.
- `ng build` clean; `ng test` 94/94 (added `TreatmentCatalogService` request-shape tests, categories list load/error, treatments list load + client-side category filtering). Not click-tested against the live backend (no test credentials) — same caveat as Phases 3–4.

### Phase 6 — Legal documents & document templates (`manage_document_templates` / `verify_legal_documents`)
- [x] Document templates list, filterable by `type`, showing version history per type (older versions read-only)
- [x] "Publish new version" form — pre-fill from the currently active version's `title`/`body` when used as an "edit," but always submits as a new `POST` (§4.4); a checkbox for "make this active immediately" mapped to `is_active`
- [x] Legal documents review queue, defaulted to `?status=submitted`, with a `type` filter (dropdown covering all five types, since only credential types realistically appear under the submitted default — see "Verified" note)
- [x] Document detail view/modal (via the shared `GET /legal-documents/{id}`) showing `metadata` fields relevant to its `type`, and the `file_url` (fetched fresh each time it's opened — never cached, §4.5)
- [x] Verify action, Reject action with a required reason field — both buttons visible only when `status === "submitted"`

**Verified (2026-09-11):**
- `src/app/features/document-templates/` — `DocumentTemplatesService` (`GET /admin/document-templates?type=&page=&per_page=`, `POST /admin/document-templates`); `document-templates-list.component` (type filter dropdown re-fetches server-side, superseded rows rendered greyed-out/read-only, "New version" opens a blank form pre-filled with the current type filter, an active row's pencil action opens the same form pre-filled from that version); `document-template-form.component` (modal — type/audience selects, title, body textarea, "Publish immediately" checkbox mapped to `is_active`; submit always `POST`s a new version, matching §4.4's no-in-place-edit contract). Routed at `/legal/document-templates`.
- `src/app/features/legal-documents/` — `LegalDocumentsService` (`GET /admin/legal-documents?status=&type=&page=&per_page=` defaulting to `status=submitted`; `PATCH .../verify`; `PATCH .../reject` with `{ reason }`; `detail()` calling the shared, non-admin-prefixed `GET /v1/legal-documents/{id}`); `legal-documents-list.component` (status + type filter dropdowns, Verify via `<app-confirm-dialog>`, Reject via a dedicated `legal-document-reject-form.component` modal requiring a reason, both actions rendered only on `status === 'submitted'` rows); `legal-document-detail.component` (read-only modal, re-fetches on every open per §4.5's file-URL caching warning, renders `dental_license`/`liability_insurance` metadata by type and an "Open file" link). Routed at `/legal/legal-documents`. Both routes nested under a new `/legal` parent (mirroring `/administration` and `/treatment-catalog`) with a "Legal Documents" sidebar group so CoreUI's auto-expand/collapse works (see AGENTS.md).
- **Design note on the type filter**: the plan called for "Licenses / Liability Insurance / everything else" tabs, but under the default `status=submitted` filter only `dental_license`/`liability_insurance` can realistically appear — §4.5 states signed consent forms/agreements never go through this workflow. Implemented as a single `<select>` covering all five `legal_document_type` values instead of fixed tabs, so broadening the status filter to `all` still lets an admin isolate a signed template type if one shows up.
- **Live-verified end-to-end** against the real backend at `127.0.0.1:8000` (a super-admin session was reachable this session): published a real "Test Consent Form" document template through the UI and confirmed it appeared in the list with the correct version/status/effective-date; the legal documents review queue loaded a real (empty) `status=submitted` result with no console errors. `ng build` clean; `ng test` includes new `DocumentTemplatesService`/`LegalDocumentsService` request-shape specs and list load/error-state specs.

### Phase 7 — Prescription templates (`manage_prescription_templates`)
- [x] Prescription templates: list, create, edit, delete (§4.6) — same shape as Phase 5's treatment catalog
- [x] Confirm-before-delete (hard delete, no undo)
- [x] Inactive templates stay visible (greyed out / filterable), not hidden — `is_active` only controls whether doctors see it when writing a prescription, not whether it still existed

**Verified (2026-09-11):**
- `src/app/features/prescription-templates/` — `PrescriptionTemplatesService` (`GET`/`POST`/`PATCH`/`DELETE /admin/prescription-templates`, same request shape as `TreatmentCatalogService`); `prescription-templates-list.component` + `prescription-template-form.component` (modal), following the Phase 5 treatment-catalog pattern exactly (frozen header/table, `<app-pagination-bar>`, `<app-confirm-dialog>` delete). Inactive rows render greyed-out (`text-body-secondary`) rather than being hidden or filtered out. Routed standalone at `/prescription-templates` (single screen, no nav group needed).
- **Live-verified end-to-end**: created a real "Amoxicillin 500mg" template through the UI (appeared in the list with an Active badge), then deleted it via the confirm dialog and saw the "Template deleted." toast and the list return to its empty state — a full create→list→delete round trip against the live backend with no console errors. `ng build` clean; `ng test` includes a new `PrescriptionTemplatesService` request-shape spec and list load/empty/error-state specs.

### Phase 8 — Invoices & reconciliation (`view_invoices` / `manage_invoices`) — backend Phase 7, §4.7
- [x] Invoice list — paginated, with `?filter[status]=` tabs/dropdown (values from `invoice_status`), plus optional `?filter[patient_profile_id]=` / `?filter[doctor_profile_id]=` when arriving from a patient/doctor detail screen; sortable by `standard_amount` / `created_at`
- [x] Invoice detail — show `standard_amount` / `advance_paid` / `remaining_amount` (all decimal strings — format as currency, don't do JS math on them) and the `payments` timeline (method, status, `paid_at`, `provider_reference`)
- [x] Reconcile action (`POST .../reconcile`) — a form with a `status` select and an optional `amount_collected`; the common case is "record a cash/transfer collection against a pay-later invoice" (`status: paid` + amount). Refund is `status: refunded` (no amount — the backend refunds the last successful gateway payment). Re-render the returned `InvoiceResource` in place afterward — no separate refetch needed
- [x] Gate the reconcile UI behind `manage_invoices`; `view_invoices` alone is list/detail only
- [~] For an "invoice for this appointment" panel on the appointment detail screen (Phase 9 below), query `GET /admin/invoices?filter[patient_profile_id]=...` and match `appointment_id` — deferred until Phase 9 builds the appointment detail screen this panel would live on; `InvoicesService.list()` already supports the needed filter params

**Verified (2026-09-11):**
- `src/app/features/invoices/` — `InvoicesService` (`GET /admin/invoices` using the `spatie/laravel-query-builder` convention — `filter[status]`, `sort=` — per §4.7/§1.1d; `POST /admin/invoices/{id}/reconcile`). `invoices-list.component` (status filter + sort dropdowns, both server-side; table shows treatment description, standard/remaining amounts as decimal strings, status badge). `invoice-detail.component` — **not** a routed page: §4.7 has no `GET /admin/invoices/{id}`, so this modal is always opened from a row the list already loaded, never a separate fetch, matching the real API surface. Shows the full money breakdown and payments timeline, plus a reconcile form gated behind `*hasPermission="'manage_invoices'"` (status select + optional amount-collected input); on success it re-renders the returned `InvoiceResource` in place and the list updates that row without a refetch. Routed standalone at `/invoices`.
- **Live-verified**: navigated to `/invoices` against the live backend and confirmed a real `GET /v1/admin/invoices?page=1&per_page=20&sort=-created_at` request returned `200 OK` (empty result — no seeded invoices in this dev database) with no console errors, confirming the query-builder param shape is accepted by the real endpoint. Reconcile itself wasn't exercised live (no seeded invoice to reconcile), but is covered by the `InvoicesService` request-shape spec. `ng build` clean; `ng test` includes new `InvoicesService` request-shape specs and list load/empty/error/reconcile-update specs.

### Phase 9 — Patients / Doctors / Appointments directories (`view_patients` / `manage_patients` / `view_doctors` / `manage_doctors` / `view_appointments`) — backend Phase 9, §4.8–§4.10
- [x] Appointments directory — a filterable table (`?filter[status]=`, `?filter[scheduled_from]=`/`?filter[scheduled_to]=` date range, `?filter[doctor_profile_id]=`/`?filter[patient_profile_id]=`), sortable by `scheduled_at`/`requested_at`. **Read-only** — no state-change actions (there's no admin endpoint for them)
- [x] Patients directory — search by first/last name (`?filter[first_name]=` / `?filter[last_name]=`, partial), filter by `city_id` and `is_account_holder`; each row shows `account_status`
- [x] Patient detail with tabs, one fetch per tab: Profile (`GET .../patients/{id}`), Appointments, Medical history, Prescriptions, Medical reports (clinical notes), Invoices. Expect the profile/history/prescriptions/medical-reports reads to be silently PHI-logged server-side — no client handling needed, but don't poll them needlessly
- [x] Patient activate/deactivate (`PATCH .../patients/{id}/status`) — **only render this control when `is_account_holder === true`**; for a dependent it returns `422` ("no account to toggle"). Gate behind `manage_patients`
- [x] Doctors directory — search by name, filter by `city_id`, sort incl. `average_rating`; row shows `account_status`
- [x] Doctor detail (no tabs — single profile view) + activate/deactivate (`PATCH .../doctors/{id}/status`, always valid for a real id). Gate behind `manage_doctors`
- [x] Handle the Phase 7/9 `400` "filter not allowed" error (§1.1d) defensively — only ever send the filter/sort keys this doc lists

**Verified (2026-09-11):**
- A real backend session was reachable this session (super-admin token still valid), so every shape below was **confirmed live** against `127.0.0.1:8000` rather than guessed from this doc's abbreviated examples — this caught two real discrepancies from what an earlier draft of this section assumed:
  - `AppointmentResource.treatment` is a **single** eager-loaded `Treatment` object, not an array or a joined description string (unlike `InvoiceResource.treatment_description`, which *is* a comma-joined string of multiple treatment names — the two resources describe the relationship differently).
  - `PatientProfileResource`/`DoctorProfileResource` as they appear **nested inside an `AppointmentResource`** (its own `patient_profile`/`doctor_profile`) omit `city`/`account_status`/`address` — those three fields are only populated by the two dedicated directory endpoints (`/admin/patients*`, `/admin/doctors*`), which eager-load `city`/`manager`/`user` specifically for that purpose. `core/models/patient.model.ts` and `doctor.model.ts` mark all three optional for exactly this reason.
  - `SpecializationResource` (`GET /v1/specializations`) is `{ id, name }` only — confirmed live, matches the plan's brief mention.
  - `AddressResource`'s shape is **still unconfirmed** — every seeded profile had `address: null`; `core/models/patient.model.ts`/`doctor.model.ts` type it as `unknown | null` and neither detail screen attempts to render it. Revisit if a screen ever needs to display one.
- `src/app/features/appointments/` — `AppointmentsService` (`GET /admin/appointments`, full `spatie/laravel-query-builder` filter set); `appointments-list.component` — status dropdown, a scheduled-date range (two native `<input type="date">`), and a sort dropdown, all server-side. Patient/doctor name cells fall back to `#<id>` when `first_name`/`last_name` are both `null` (common in this seed data — many profiles are incomplete), and the doctor cell shows "Unclaimed" for an `open`-status appointment with `doctor_profile: null`. Routed standalone at `/appointments`, no create/edit/delete UI (read-only per the plan).
- `src/app/features/patients/` — `PatientsService` (list with the documented `filter[]`/`sort`, plus one method per tab endpoint, plus `setStatus`); `patients-list.component` (first/last name search inputs + Enter-to-search, a city filter sourced from the new shared `CitiesService`, an account-holder/dependent filter, sort dropdown) routed at `/patients`; `patient-detail.component` at `/patients/:id` — profile fetched on load, the other five tabs (Appointments / Medical History / Prescriptions / Medical Reports / Invoices) fetched **only on first activation** and cached afterward (a `*Loaded` signal per tab guards against re-fetching, which matters since most of these tabs are PHI-logged server-side on every read). The deactivate control only renders when `is_account_holder === true`, gated by `*hasPermission="'manage_patients'"`.
- `src/app/features/doctors/` — `DoctorsService` (list/get/status, same shape as patients minus the tabs); `doctors-list.component` (name search, city filter, sort incl. `average_rating`) routed at `/doctors`; `doctor-detail.component` at `/doctors/:id` — single profile view (bio, education, specializations, rating), activate/deactivate gated by `*hasPermission="'manage_doctors'"` (always valid for a doctor, unlike the patient dependent-profile `422` case).
- `src/app/core/services/cities.service.ts` (new) — a thin wrapper around the shared, unauthenticated `GET /v1/cities` (§4.11), used only to populate the patients/doctors directory city filter dropdown. This is **not** the Phase 10 city-management screen — that still needs the admin-gated `/admin/cities` (list including unavailable cities, plus `PATCH`) and the city-requests queue.
- Sidebar gained three standalone entries — Appointments, Patients, Doctors (icons `cilCalendar`/`cilUser`/`cilUserFollow`) — as flat top-level items rather than a nav group, since these three don't share a natural parent the way Administration/Treatment Catalog/Legal Documents do.
- **A real, reproducible bug was found and fixed while building the patient detail tabs**: CoreUI's animated `<c-tabs>`/`<c-tab-panel>` (`@fadeInOut` trigger) got stuck mid-transition in this build — the outgoing panel never lost its `active show` class and the incoming panel never gained it, even though the underlying tab-selection state and per-tab data fetch were both correct. This reproduced reliably on a fresh page load with a single click, not just rapid clicking, and left the UI showing the *previous* tab's content indefinitely. **Fixed by not using `<c-tabs>` at all** for `patient-detail.component` — it's a plain Bootstrap `<ul class="nav nav-tabs">` of buttons toggling a `.active` class, paired with `@switch (activeTab()) { @case (...) { ... } }` to render only the selected panel's content (no animation, no CoreUI tabs components). See the new "CoreUI `<c-tabs>` fade animation" note in AGENTS.md before using `<c-tabs>` anywhere else in this app.
- **Live-verified end-to-end**: browsed the real Appointments/Patients/Doctors directories against the live backend (confirmed the `#<id>` fallback and "Unclaimed" doctor rendering on real incomplete-profile seed data), opened a real patient's detail page and clicked through all six tabs individually confirming each one fetches once and renders its own content (most were empty — "No results" — for this patient, which is itself confirmation the empty-state path works), and opened a real doctor's detail page. `ng build` clean; `ng test` **147/147** (23 new tests across 9 new spec files: `AppointmentsService`, `PatientsService`, `DoctorsService`, `CitiesService`, and list/detail component specs for all three directories).
- **Follow-up (2026-09-13)**: backend added `email`, `phone_number`, `is_email_verified`, `is_mobile_verified` to `PatientProfileResource` (both `GET /admin/patients` and `GET /admin/patients/{patient}`). `core/models/patient.model.ts` updated; `patients-list.component` gained an Email column; `patient-detail.component`'s Profile tab shows both fields with a Verified/Unverified badge next to each (badge only rendered when the value itself is non-null).

### Phase 10 — Cities & city requests (`manage_cities`) — backend Phase 9, §4.11
- [x] City catalog — list (use the `/admin/cities` variant so unavailable cities show), edit `name`/`state`/`is_available` via `PATCH /admin/cities/{city}`
- [x] City requests — a read-only informational queue (`GET /admin/city-requests`); no approve/reject action exists, its `status` flips as a side effect of enabling the requested city in the catalog

**Verified (2026-09-12):**
- `src/app/features/cities/` — `CitiesService` (`GET`/`PATCH /admin/cities`, `GET /admin/city-requests` — neither is a `spatie/laravel-query-builder` endpoint, only `page`/`per_page` are honoured). `cities-list.component` (`/cities/catalog`) is **edit-only**: there's no `POST /admin/cities` in the backend contract, so there is no "new city" button — `city-form.component` (a modal) only ever opens pre-filled from an existing row, and the saved row is patched in place from the response rather than reloading the whole list. `city-requests-list.component` (`/cities/requests`) is a plain read-only table — no actions at all, per §4.11's "informational queue" framing, with a hint line telling the admin how a request actually resolves (enabling its city in the catalog above, not anything on this screen).
- Routed as a single `cities.routes.ts` (`'' → catalog` redirect, `catalog`, `requests`) under one `manage_cities`-gated parent route, mirroring `treatment-catalog`'s two-screens-one-permission shape. Sidebar gained a **Cities** group (icon `cilLocationPin`) with City Catalog/City Requests children, nested under `/cities/...` for the auto-expand/collapse convention.
- Not click-tested against a live backend this pass (no admin session was available), but every request shape mirrors `RolesService`/`TreatmentCatalogService`'s already-live-verified plain page/per_page pattern, and the route/guard chain was confirmed to resolve without error (redirects cleanly to `/login` when unauthenticated, same as every other guarded route). `ng build` clean; `ng test` **162/162** (8 new tests: `CitiesService` request shapes, `cities-list` load/error/in-place-update, `city-requests-list` load/error).

### Phase 11 — PHI access audit log (`view_phi_access_logs`) — backend Phase 9, §4.12
- [x] Log viewer — paginated table with plain-query-param filters (`?action=`, `?resource_type=` full class string, `?actor_id=`); newest first
- [x] Export view/button (`GET .../export`) — unpaginated (max 5000), plain `data: []`, no `meta`; don't auto-refresh (10/min rate limit)
- [x] Make it explicit in the UI that opening this screen writes its own audit row — the viewer's own reads will appear in the results; that's intended, not a bug to filter away

**Verified (2026-09-12):**
- `src/app/features/phi-access-logs/` — `PhiAccessLogsService` (`GET /admin/phi-access-logs` + `.../export`, **plain `?action=`/`?resource_type=`/`?actor_id=` query params**, not the `filter[key]=` bracket convention the Phase 7/9 directory endpoints use — an important distinction since this is the one remaining admin list endpoint still on the older param style, per §1.1d/§7). `phi-access-logs-list.component` — action dropdown (the 6 `phi_access_action` values, hardcoded per the Phase 5-established stopgap pending `GET /enums` caching), a resource-type dropdown scoped to the PHP model classes this app's own endpoints actually PHI-log against (`PatientProfile`, `MedicalHistory`, `Prescription`, `ClinicalNote`, `Message`, plus the self-referential `PhiAccessLog`) rather than a free-text field, and a numeric actor-id filter. A permanent hint line states plainly that opening the screen (and every export) writes its own row and that seeing your own reads on the next page is expected. No auto-refresh anywhere (respects the 10/min `throttle:admin-export` limiter).
- **Export** calls `.../export` and saves the unpaginated result client-side as a downloaded `.json` file (`Blob` + a temporary anchor click) — this is a normal in-product export feature, not a network fetch of external content, so no special handling was needed beyond the existing error/toast pattern. `export`'s error path is covered by a unit test; the success path's `Blob`/anchor-click side effect wasn't exercised by a real download in CI to avoid flakiness in the headless test browser.
- Routed standalone at `/phi-access-logs` (single screen, no create/edit/delete — an insert-only, unfilterable-by-the-dashboard audit trail). Sidebar gained a flat **PHI Access Log** entry (icon `cilShieldAlt`, added to `icon-subset.ts`).
- Not click-tested against a live backend this pass (no admin session was available); route/guard resolution was confirmed to redirect cleanly to `/login` when unauthenticated. `ng build` clean; `ng test` **162/162** (7 new tests: `PhiAccessLogsService` plain-param request shapes incl. the `filter[]`-vs-plain-param distinction, `phi-access-logs-list` load/error/filter-refetch/export-error).

### Phase 12 — Messaging visibility (optional, product call) — backend Phase 8, §4.13
- [ ] Only if a support/dispute workflow needs it: a read-only conversation browser (`GET /api/v1/conversations` — an admin sees all), drilling into `GET /api/v1/conversations/{id}/messages`
- [ ] No compose box — an admin `POST` to messages is `403` by policy
- [ ] Reading a thread is PHI-logged (bodies are PHI); `attachment_url` is a short-lived signed URL, re-fetch don't cache

### Blocked / not buildable — nothing admin-facing remains
- [ ] Clinical documentation *authoring/review* (tooth charts, clinical notes, treatment plans, prescriptions themselves — as opposed to prescription *templates*, Phase 7 above, or the read-only clinical-notes tab in §4.9) — **backend Phase 6**, deliberately doctor/patient-only under HIPAA minimum-necessary scoping. The admin can *read* a patient's clinical notes via §4.9's medical-reports tab, but there is no admin endpoint to create or edit any of this, and none is planned
- [ ] Anything depending on **backend Phase 10** (HIPAA hardening close-out) or **Phase 11** (real payment gateway) — not yet built. Phase 11 changes no API shape the dashboard sees; Phase 10 may add "log out all devices" and tighten error envelopes, at which point revisit §1.1 and §2

---

## 7. Open items raised with the backend team

The first three items are resolved as of 2026-09-09 — kept for history, not as active work:
- ~~No "my own roles/permissions" endpoint~~ — resolved: `GET /admin/me` (§3.3).
- ~~No enum-label catalog endpoint~~ — resolved: `GET /enums` (§5).
- ~~No staff edit/deactivate endpoint~~ — resolved: `PATCH /admin/staff/{staff}` (§4.2).

Open, discovered while documenting backend Phases 7–9 (2026-09-15) — not blockers, but flag to the backend team if a screen needs them:
- **No action on a city request (§4.11).** `GET /admin/city-requests` is read-only; there's no endpoint to mark one handled/dismissed or to link it to the city that fulfilled it. A review-queue UI can only display them. Backend's stated intent is that `status` flips `pending → notified` as a side effect of adding the city, but nothing wires that up yet either.
- **Filter-param convention is inconsistent across admin list endpoints.** Phase 7/9's new lists (§4.7–§4.10) use `spatie/laravel-query-builder` (`?filter[key]=`, `?sort=key`, `400` on unknown keys); the PHI log (§4.12) uses plain `?key=` params; the older lists (§4.1–§4.6) mostly have no filtering at all. A generic "list screen" abstraction on the frontend needs to account for all three.
- **`AppointmentResource.invoice` is defined but never populated** by any current endpoint (§4.7) — treat it as not-there for now.

Discovered while building the frontend (2026-09-11):
- **No server-side category filter on `GET /admin/treatments` (§4.3).** The treatments list can't ask the backend for "just this category" — the endpoint isn't on `spatie/laravel-query-builder` and silently ignores unknown params rather than filtering or `400`ing. The frontend filters client-side over whatever page is currently loaded (Phase 5), which is fine for a small catalog but won't scale if the treatment list grows into the hundreds. Worth a `?filter[treatment_category_id]=` if this catalog ever gets that large.

If a new gap turns up while building against this plan, add it here the same way, with a §-reference to where it affects the plan above.

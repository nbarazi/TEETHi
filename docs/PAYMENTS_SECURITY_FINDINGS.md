# Payments rollout: security review findings

Findings from the `security-reviewer` passes over the payments rollout (`plans/PAYMENTS_ROLLOUT_PLAN.md`), ranked by severity. Tick an item and note the fixing commit when it lands.

| Review | Date | Commits reviewed |
|---|---|---|
| P1 + P2 | 2026-11-11 | TEETHi-Backend `59d32cf`, `8f55b38`; TEETHi-Frontend `e729228`, `86dd5dd`; TEETHi-Patient-App `32889c3` |
| P3 | 2026-11-11 | TEETHi-Backend `efb1a4a`; TEETHi-Frontend `0039054`; TEETHi-Patient-App `083494d`; TEETHI-Dentist-App `2131c7a` |

## Summary

| # | Severity | Phase | Finding | Status |
|---|---|---|---|---|
| S1 | High | P2 | Receipt email attaches a PDF with health information, sent through SendGrid | Fixed |
| S2 | High | P3 | Treatment check runs before authorization and reveals what is on any appointment | Fixed |
| S3 | High (money) | P3 | A dentist can add any treatment to their "expertise" mid-visit and bill it; inactive and wrong-visit-type treatments aren't blocked | Fixed |
| S4 | Medium | P2 | The fallback `log` mailer writes the receipt PDF to `laravel.log` | Fixed |
| S5 | Medium | P2 | `view_invoices` staff get receipt links showing patient names; no PHI access audit | Fixed |
| S6 | Medium | P3 | Recalculating the total can reopen a refunded invoice or undo an admin-set Paid | Fixed |
| S7 | Medium (money) | P3 | `refund_due` never clears, and the built-in refund refunds the wrong amount | Fixed |
| S8 | Medium-low (money) | P3 | A payment started at the same moment can still charge the old balance | Open |
| S9 | Low | P1 | The fake-gateway guard only recognises `APP_ENV=production` | Open |
| S10 | Low | P2 | Receipt links: bearer-only for 30 minutes, no rate limit, maybe `http://` behind a proxy | Open |
| S11 | Low | P3 | Backfill uses today's catalog price, so lines may not add up to the stored total | Open |
| S12 | Low | P3 | Treatment change, outcome and completion aren't one locked transaction | Open |
| S13 | Low | P3 | Smaller points: stale primary treatment, billing status shown to the dentist, prices sent to the dentist app | Open |

## Details

### S1. High (P2): receipt email carries health information as an attachment
- **Where:** `TEETHi-Backend/app/Notifications/ReceiptNotification.php` (`attachData` of the PDF); snapshot built in `app/Services/ReceiptService.php` `issueFor()`.
- **Issue:** every payment, refund and pay-later collection emails a PDF with the patient's name, the dentist, the visit date and the treatments, through SendGrid (the only production mail transport configured). If SendGrid has no BAA, health information goes to a vendor without one (unverified: check the actual agreement). It also goes unencrypted to whatever address is on the account, including dependents' receipts going to the account holder.
- **Fix:** keep the email body (it has no amounts or treatments) and drop the attachment, pointing to Records in the app. If an attachment is required, use a BAA-covered mail provider and make it opt-in.
- **Fixed: TEETHi-Backend `7bfcd9a`, merged as `439fb6c` (2026-11-11).** The email has no attachment and says the receipt is ready in the app under Records. The PDF is still rendered for the app's signed link. Test: `ReceiptTest` "emails a notice without the PDF".

### S2. High (P3): the treatment check runs before authorization
- **Where:** `TEETHi-Backend/app/Http/Requests/Api/V1/Doctor/StoreVisitOutcomeRequest.php` (`authorize()` returns `true`; `after()` reads the appointment's treatments); ownership is only checked later in `VisitOutcomeController::store`.
- **Issue:** any dentist can post `{"treatments":[X]}` to any appointment id: `422` means X isn't on it, `403` means it is. Looping over the catalog and appointment ids reveals every appointment's treatments. For unassigned appointments the answer is exactly the booked list.
- **Fix:** do the policy check in `authorize()` (`can('create', [VisitOutcome::class, $appointment])`), or move the expertise check after `authorize` in the controller. Add a test: another dentist gets `403`, never `422`.
- **Fixed: TEETHi-Backend `7bfcd9a`, merged as `439fb6c` (2026-11-11).** `StoreVisitOutcomeRequest::authorize()` runs the policy, so another dentist gets `403` whatever they send. Test in `FinalTreatmentsTest`.

### S3. High, money (P3): self-declared expertise lets a dentist inflate a bill
- **Where:** `StoreVisitOutcomeRequest::after()` (only checks the expertise row is active); `InvoiceService::confirmTreatments()`; `StoreTreatmentExpertiseRequest` (`exists:treatments,id`, no active check, no approval).
- **Issue:** mid-visit, a dentist can add the most expensive catalog treatments to their own services (`POST /doctor/services`) and complete the visit with up to 20 of them. Booking rules are skipped: retired (`is_active = false`) treatments can be billed, and virtual treatments can go on a home visit or physical ones on a tele call. "No cap" is a business decision (D6), but P4 will turn this straight into card charges.
- **Fix:** for added ids, require `is_active` and an `is_virtual` that matches the visit type (reuse the booking guards). Make `StoreTreatmentExpertiseRequest` accept only active treatments. Before P4, accept only expertise created before the visit started, or hold visit-time additions above a threshold for approval. Add tests for each case.
- **Fixed: TEETHi-Backend `7bfcd9a`, merged as `439fb6c` (2026-11-11).** A treatment added at the visit must be one the dentist offered, actively, before the visit started (expertise created and last changed before the move to in progress), active in the catalog, and virtual only on a tele call. Dentists can't claim retired treatments. Booked treatments are unaffected. Tests in `FinalTreatmentsTest`. Still open as a business choice: no cap and no patient approval on a rise (D6).

### S4. Medium (P2): the `log` mailer writes the PDF to the application log
- **Where:** `TEETHi-Backend/config/mail.php` (`'default' => env('MAIL_MAILER', 'log')`).
- **Issue:** with the `log` mailer (the `.env.example` default, and the fallback when unset) the whole message, including the base64 PDF, lands in `storage/logs/laravel.log`. This happens on every dev machine, and in production if `MAIL_MAILER` is unset.
- **Fix:** fail at boot in production when the mailer is `log` or `array`, like the Stripe guard, or skip receipts on the `log` mailer, and add `MAIL_MAILER` to the consistency report's production settings. Fixing S1 removes most of this.
- **Fixed: TEETHi-Backend `b7735b3`, merged as `0867191` (2026-11-11).** `AppServiceProvider::guardProductionMailer()` stops a production boot when `MAIL_MAILER` is `log` or `array`. `MAIL_MAILER` is on the consistency report's production settings list. Locally the `log` mailer still works; since S1 it only logs the short notice.

### S5. Medium (P2, widened by P3): receipt links and line items for `view_invoices` staff, unaudited
- **Where:** `Admin/InvoiceController.php` (loads `receipts` and `items`), `ReceiptResource::download_url`, the unauthenticated `ReceiptController::pdf`.
- **Issue:** a role with `view_invoices` but not `view_patients` gets 30-minute PDF links showing the patient's name, the dentist and the visit date. No `PhiAccessLog` row is written, and the PDF route has no actor to record. P3 adds treatment names per line to the same list (same category as `treatment_description`).
- **Fix:** drop `download_url` from the admin list. Add an authenticated `GET admin/invoices/{invoice}/receipts/{receipt}/pdf` under `view_invoices` that checks the receipt belongs to the invoice and calls `logPhiAccess`. Decide whether opening the PDF needs `view_patients`.
- **Fixed: TEETHi-Backend `b7735b3`, merged as `0867191`; TEETHi-Frontend `552da2e`, merged as `7ca9a54` (2026-11-11).** Admin responses carry no `download_url`. Staff open the PDF through `GET admin/invoices/{invoice}/receipts/{receipt}/pdf`, which needs `view_invoices` and `view_patients` and writes a `downloaded` PHI access log row. The dashboard fetches it as a file and shows Download only to staff with `view_patients`. Still open: the treatment names on invoice lines for `view_invoices` staff (same category as the existing `treatment_description`).

### S6. Medium (P3): recalculating the total overwrites admin-set invoice states
- **Where:** `InvoiceService::acceptsTreatmentChanges()` (unlocked read); `confirmTreatments()` (rebuilds the status from `advance_paid` alone); `reconcile()` (doesn't lock the invoice).
- **Issue:**
  - **Race:** an admin refunds the invoice while the dentist's request is waiting on the lock; confirm then writes `Pending` and the patient is asked to pay again.
  - **Without a race:** an invoice an admin marked Paid without an amount (waived) flips back to Pending on any treatment change.
- **Fix:** re-check the status inside the locked transaction (`409` unless Pending, PartiallyPaid or Paid), make `reconcile()` lock the row, and decide whether an admin-set Paid with nothing collected stays Paid.
- **Fixed: TEETHi-Backend `b7735b3`, merged as `0867191` (2026-11-11).** `confirmTreatments` re-checks the invoice status inside the lock (`409`), `reconcile` locks the invoice, and an invoice staff marked Paid with nothing collected stays Paid with nothing due when the treatments change.

### S7. Medium, money (P3): `refund_due` never clears
- **Where:** `Invoice::refundDue()`; `StripeWebhookService` (no `charge.refunded` handling); `InvoiceService::refundLatestGatewayPayment()` (always refunds the latest payment in full).
- **Issue:** a refund staff make in Stripe's dashboard (what the plan says to do until P6) never comes back to TEETHi, so "Refund due" stays forever and a second staff member may refund again. Reconcile → Refunded refunds the whole latest payment and closes the invoice. If a partial refund already happened, Stripe rejects the full refund, and the failure is ignored while the invoice is still marked Refunded.
- **Fix:** until P6, add a staff "record refund" action (Refunded payment row, lower `advance_paid`, receipt), or mirror dashboard refunds from `charge.refunded` idempotently. At minimum, have `reconcile` return an error when the refund fails.
- **Fixed: TEETHi-Backend `b7735b3`, merged as `0867191`; TEETHi-Frontend `552da2e`, merged as `7ca9a54` (2026-11-11).** New `POST admin/invoices/{invoice}/refund-difference` (`manage_invoices`): refunds `refund_due` to the card through Stripe (newest card payment first, idempotent per payment and amount), or with `record_only` records a refund already made in Stripe's dashboard or in cash. Each refund gets a payment row and a refund receipt, and the flag clears. A declined refund, or no card payment to refund from, is a `422` and changes nothing. `reconcile` → Refunded now returns `422` when Stripe declines instead of marking the invoice Refunded. The dashboard's Refund due box has Refund to card and Record a refund made elsewhere, each with a confirm step. Not handled: refunds made in Stripe's dashboard still don't sync on their own (`charge.refunded`); staff record them.

### S8. Medium-low, money (P3): a payment running at the same moment can charge the old balance
- **Where:** `Patient/AppointmentController::pay()` (reads `remaining_amount` with no lock); `InvoiceService::confirmTreatments()` (cancels only the PaymentIntents it sees).
- **Issue:** if the patient starts paying while the dentist lowers the total, the PaymentIntent can be created for the old amount after the cancel loop, and the patient pays the old balance (ending as S7's refund due). Stripe calls also run while the invoice row is locked.
- **Fix:** lock and re-read the invoice in `pay()` before creating the payment. Better: when a gateway payment completes, refund anything above the current balance automatically.

### S9. Low (P1): the fake-gateway guard only recognises `production`
- **Where:** `TEETHi-Backend/app/Providers/AppServiceProvider.php`.
- **Issue:** a public staging or pilot server with `APP_ENV=staging` and no Stripe key gets `FakePaymentGateway`, which approves every payment: invoices marked paid with no money taken.
- **Fix:** allow the fake gateway only in `local` and `testing` (`throw_unless($app->environment('local', 'testing'), ...)`), and update `PaymentGatewayBindingTest`.

### S10. Low (P2): receipt links
- **Where:** `routes/api/v1.php` (`signed` only, no `throttle`); `ReceiptResource` (30 minutes); patient app opens it in the external browser.
- **Issue:** a forwarded or synced link serves the PDF to anyone for 30 minutes, even after the account is deleted. Each hit renders a PDF with no rate limit. Behind a TLS-terminating proxy without `trustProxies`, links may be issued as `http://` (unverified: depends on hosting).
- **Fix:** shorten the link to about 5 minutes (the app fetches a fresh one per tap), add `throttle:60,1`, configure `trustProxies` for the host, optionally open in an in-app view.

### S11. Low (P3): backfill prices
- **Where:** `database/migrations/2026_11_11_000003_create_invoice_items_table.php`.
- **Issue:** old invoices get lines at today's catalog price. If a price changed since, lines don't add up to the stored total, and the first P3 change re-prices untouched lines. The backfill isn't in a transaction. (On the local database all 21 invoices matched.)
- **Fix:** when lines don't add up to `standard_amount`, take the line amount from the stored total or add an adjustment line; test that lines sum to the total after migrating.

### S12. Low (P3): non-atomic completion
- **Where:** `VisitOutcomeController::store`.
- **Issue:** the invoice change and the patient push commit before the outcome is saved and the visit completed; if completion fails, the patient was already told. The completed-status check isn't locked, so a second request from the same dentist can change the list just after completion.
- **Fix:** wrap confirm, outcome and completion in one transaction with the appointment locked, re-check the status inside it, and notify after commit.

### S13. Low (P3): smaller points
- **Stale primary treatment:** `appointments.treatment_id` isn't updated when the list changes, so lists can still show a removed treatment.
- **Billing status to the dentist:** `409 payments.invoice_closed` tells the dentist the patient's invoice was cancelled or refunded, though doctors are otherwise never shown invoices. Use a generic message.
- **Prices sent to doctors:** `GET /doctor/services` and `GET /treatments` include `standard_amount`; the dentist app only hides them on screen. Predates P3. If D2 or D6 mean dentists shouldn't see prices, give doctors a shape without the price.

## Checked and found fine
- Patient receipt list, admin resend and the signed PDF route are scoped correctly; request logs record only parameter names, so signatures don't reach the log.
- Dispute webhooks are signature-checked and idempotent; only ids go to Stripe as metadata.
- `Receipt`, `Payment` and `InvoiceItem` are not mass-assignable; receipt snapshots are encrypted and never rewritten by later changes; the PDF view escapes everything and dompdf runs with remote resources and PHP disabled.
- Visit-outcome ownership is enforced by `VisitOutcomePolicy` before any write (S2 is the validation leak before it). Prices come from the server; clients send ids only.
- The `invoice_total_changed` push carries only ids and generic text and goes to the account holder; a crafted `invoice_id` in the patient app can only open the patient's own invoice.
- Doctors still get `invoice: null` on appointments; the dashboard uses only Angular interpolation.

# TEETHi — Payments rollout plan

Written 2026-11-10. This plan implements the recommendation in the business summary "TEETHi payments: status and options":

- Save the card at booking, hold the visit price 24 hours before the visit, and charge the final amount when the dentist completes the visit.
- Pay dentists from an earnings ledger by monthly bank transfer. Move to Stripe Connect later.
- Launch market: United States, USD.

It builds on backend Phase 7 (invoices, pay later) and Phase 11 (Stripe payment sheet) in [IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md).

**How the phases are cut.** Each phase is one feature delivered end to end: the backend change and every dashboard and app change that depends on it ship together. No phase leaves a client calling an endpoint that doesn't exist, or a backend feature with no screen. Phases are named P1–P8 so they don't collide with the backend plan's phase numbers.

**Tracking.** Tick items here and add a "Verified" note with the project commit hashes, as in the other plans.

## 1. Phase overview

| Phase | Delivers | Depends on | Business decision needed first |
|---|---|---|---|
| P1 | Live payments with today's flow | — | Launch states (D1) |
| P2 | Receipts by email and in the app | P1 | — |
| P3 | Final price confirmed by the dentist at the end of the visit | — | Can the price change during the visit? (D6) |
| P4 | Card on file, 24-hour hold, charge on completion | P1, P3 | Pay before or after the visit (D6) |
| P5 | Pay later limited to patients with a saved card, or removed | P4 | Keep pay later? Cash at the door? (D8) |
| P6 | Late-cancellation and no-show fee, partial refunds | P4 | Cancellation policy (D7) |
| P7 | Dentist earnings ledger, statements, bank-transfer payouts | P1 | Revenue model (D2), who absorbs fees and disputes (D5) |
| P8 | Stripe Connect payouts | P7 | Seller of record (D4). Start when manual payouts become a burden |

P2, P3 and P7 don't depend on each other and can run in parallel after P1.

## 2. Decisions that gate the phases

These are the questions from the business summary. A phase doesn't start until its decision is recorded here.

| # | Decision | Gates | Answer |
|---|---|---|---|
| D1 | Which US states do we launch in? (with a healthcare lawyer) | P1 go-live | **California** (decided 2026-11-11, "for now"). Still open: a California healthcare lawyer's review before go-live |
| D2 | How TEETHi earns: commission per visit, flat fee per booking, or dentist subscription. Many states restrict a percentage of dental fees. | P7 | _open_ |
| D3 | Dentists paid per visit, or on salary or fixed fees | P7 (salary: P7 shrinks to an earnings view, P8 is dropped) | _open_ |
| D4 | Seller to the patient: TEETHi or the dentist | P8 | **TEETHi** (decided 2026-11-11, "for now"). TEETHi's name is on the card statement and TEETHi owns refunds and disputes. The lawyer must confirm a non-dentist company can be the seller of dental care in California |
| D5 | Who absorbs Stripe's processing fees and lost card disputes | P7 | _open_ |
| D6 | Patients pay before or after the visit; can the price change during the visit | P3, P4 | **After the visit; yes, the price can change** (decided 2026-11-11, "for now"). No cap on the rise for now: the patient is notified, no in-app approval (decided 2026-11-11) |
| D7 | Late-cancellation and no-show policy: window, fee amount, who can waive it | P6 | _open_ |
| D8 | Keep pay later? Allow cash at the door? | P5 | _open_ |

## 3. Rules for every phase

- **Contract.** After backend changes run `/sync-api-contract`, update the client plans that document the changed endpoints in the same root commit, and run the dentist app's contract tests.
- **No clinical data in Stripe.** Only ids go in Stripe metadata. Treatment names stay in TEETHi (already the rule in `InvoiceService::pay()`).
- **Stripe is the source of truth for money.** A balance moves only on a verified webhook, never on an app's word. Every Stripe call that creates or moves money carries an idempotency key.
- **No polling.** Screens refresh on a foreground push or after the user's own action (workspace rule). Every new money event below sends a push for that reason.
- **Tests.** Backend: Pest feature tests through `FakePaymentGateway`, PHPStan max, Pint. Apps: view-model and mapper tests, plus the l10n "every message is used" test. Both languages (en, ar) for every new string.
- **Review.** Run the `security-reviewer` agent before merging any phase that touches payments or the webhook, and `contract-reviewer` after any endpoint change.

---

## P1 — Go live with today's flow

**Goal:** real cards work in production with the flow that exists: pay now through the payment sheet, pay later, admin refund.

**Backend**
- [ ] Live Stripe account verified for the launch entity. Restricted live key (`rk_live_`) with only the permissions the gateway uses.
- [ ] Production env: `STRIPE_SECRET_KEY`, `STRIPE_PUBLISHABLE_KEY`, `STRIPE_WEBHOOK_SECRET`. Register the live webhook endpoint for the events in `StripeWebhookService::EVENTS`.
- [ ] Run `2026_10_22_000002_add_stripe_payments` in every environment.
- [x] Webhook end to end with the Stripe CLI against a test-mode backend: pay, cancel the sheet, refund, a redelivered event, an out-of-order event. Verified 2026-11-12, plus dispute created/closed: `docs/E2E_REPORT_2026-11-12.md`.
- [x] Handle `charge.dispute.created` and `charge.dispute.closed`: record the dispute on the payment and flag the invoice.
- [x] Fail closed: in production the gateway refuses to fall back to `FakePaymentGateway` when no secret key is set.

**Admin dashboard**
- [x] Invoice list and detail show a "Disputed" flag and the payment's Stripe reference, so staff can find it in the Stripe dashboard.

**Patient app**
- [ ] Apple Pay: merchant ID, payment-processing certificate uploaded to Stripe, `STRIPE_MERCHANT_ID` dart-define, capability in the iOS target.
- [ ] Google Pay: production access approved; `testEnv` already follows `kDebugMode`.
- [ ] Release build passes a real payment on Android and iOS (card, Apple Pay, Google Pay), then a refund from the dashboard.

**Dentist app**
- No change.

**Done when:** a live card payment, a cancelled sheet and an admin refund all leave the invoice correct in TEETHi and in Stripe, on both phones.

**Built and merged (2026-11-11), not yet run against Stripe:** TEETHi-Backend `59d32cf`, merged to `master` as `2ebb75c`; TEETHi-Frontend `e729228`, merged to `master` as `0d0af53`. On merged `master`: backend 827 tests, PHPStan max and Pint clean; dashboard 213 tests and a production build. The `security-reviewer` pass was not run before the merge.

- Backend: `payments.dispute_status` / `dispute_reference` / `disputed_at` (migration `2026_11_11_000001_add_dispute_to_payments`), enum `payment_dispute_status` (`open`, `won`, `lost`), `PaymentResource.dispute_status` / `disputed_at`, `InvoiceResource.disputed`, `GET /admin/invoices?filter[disputed]=1`. The balance is not touched by a dispute. A closed dispute stays closed if `created` arrives late.
- Register two more webhook events with the live endpoint: `charge.dispute.created` and `charge.dispute.closed`.
- Dashboard: a Disputed badge in the list and the detail, the dispute state beside each payment, a hint to respond in Stripe, and a Disputed entry in the status filter. The Stripe reference was already shown. Not checked in a browser.
- `docs/api/openapi.json` regenerated from merged `master`: `InvoiceResource.disputed`, `PaymentResource.dispute_status` / `disputed_at`, and the `filter[disputed]` note on `GET /admin/invoices`. The migration ran on the local database.
- Still open: everything that needs the live Stripe account, the Stripe CLI, or the Apple and Google merchant setup (the unticked items above).

---

## P2 — Receipts

**Goal:** every payment and refund produces a receipt the patient can keep and use for an insurance claim.

**Backend**
- [x] Receipt number per successful payment and per refund (sequential, never reused).
- [x] Receipt PDF generated by TEETHi, not Stripe, because it lists the treatments: patient, dentist, visit date, treatments and amounts, payment method (brand and last four), amount paid, balance. Needs a PDF library (none installed yet).
- [x] Stored on the private disk and served by a 30-minute signed URL, like every other file.
- [x] Email the receipt when the webhook confirms a payment or a refund, and when staff record a pay-later collection.
- [x] `GET /patient/invoices/{invoice}/receipts` and a `receipts` array on `InvoiceResource`.
- [x] `POST /admin/invoices/{invoice}/receipts/{receipt}/resend` (`manage_invoices`).

**Admin dashboard**
- [x] Invoice detail lists receipts with Download and Resend.

**Patient app**
- [x] Invoice detail (Records) lists receipts with a View action. Not on the visit detail: the invoice embedded in a visit doesn't carry receipts.
- [x] "Receipt sent to your email" in the payment success message.

**Dentist app**
- No change (invoices stay hidden from dentists).

**Later, from the insurance plan:** when treatments carry CDT codes and dentists carry NPI and tax ID, the same PDF becomes the claim-ready receipt. Keep the template ready for those fields.

**Done when:** paying, refunding and recording a pay-later collection each produce a numbered receipt by email and in the app.

**Built and merged (2026-11-11), not yet run with real mail or on a phone:** TEETHi-Backend `8f55b38` (master `16a78fb`), TEETHi-Frontend `86dd5dd` (master `c6a0adb`) and TEETHi-Patient-App `32889c3` (master `c006593`). Tests pass on merged master: backend 842, dashboard 214, patient app 211. Migration run locally.

- Backend: `receipts` table (migration `2026_11_11_000002_create_receipts_table`), one per succeeded payment, pay-later collection or refund, issued inside the payment's own transaction and emailed from the queue after it commits. 842 tests pass, PHPStan max and Pint clean.
  - The number is the row id (`R-000123`), so it runs in order and is never reused.
  - The receipt stores an encrypted snapshot (patient, dentist, visit date, treatments, total, balance, method, card). The PDF is rendered from the snapshot on request, not stored as a file, so the "private disk" item became "no file at all". The link is still signed and lasts 30 minutes (`GET /receipts/{receipt}/pdf`).
  - PDF library: `barryvdh/laravel-dompdf`. The PDF is in English only (launch market); the email follows the app language.
  - Since security fix S1 (TEETHi-Backend `439fb6c`) the email carries no PDF: it says the receipt is ready in the app under Records.
  - The card shows as "Visa ending 4242" when Stripe reports it (`PaymentGatewayService::retrievePayment()` now returns `card`).
  - A refund Stripe later fails voids its receipt (`voided`), which then prints VOID.
  - A deleted account gets no email; the receipt is still issued.
  - New enum `receipt_kind` (`payment`, `refund`).
- Dashboard: the invoice detail lists receipts with kind, a Void badge, when it was emailed, Download and Resend (`manage_invoices`). 214 tests pass and the production build succeeds. Not checked in a browser.
- Patient app: receipts on the Records invoice detail. View asks `GET /patient/invoices/{invoice}/receipts` for a fresh link each time and opens the PDF in the browser. The success message now ends "and we'll email your receipt". 211 tests pass, analyzer has no errors or warnings. Not checked on a device.
- `docs/api/openapi.json` regenerated after the merge.

---

## P3 — Final price confirmed at the end of the visit

**Goal:** the invoice reflects the care actually given. Today it is fixed at booking.

**Backend**
- [x] `invoice_items` table (treatment, name snapshot, amount snapshot, `added_at_visit`, `removed_at`). The invoice total is the sum of its items; `standard_amount` stays as the stored total. Backfill one item per booked treatment.
- [x] The completion call takes the final treatment list. Without it, the booked list stands. Only the treating dentist, only while the visit is in progress. Built on the existing completion endpoint, `POST /doctor/appointments/{appointment}/visit-outcome` (`treatments: [id]`), not a new `/complete`.
- [x] Rules: a dentist can add treatments from their own expertise and remove ones not done. No cap on the increase for now (decided 2026-11-11); the patient is told. Since security fix S3 (TEETHi-Backend `439fb6c`), an added treatment must be expertise the dentist had before the visit started, active in the catalog, and the right kind for the visit.
- [x] Recalculate `remaining_amount`. If the patient already paid more than the new total, flag the invoice "refund due" for staff (the refund itself is P6's partial refund; until then staff use Stripe's dashboard).
- [x] Push and notification to the patient: "Your visit total changed". The new total is in the app, not the push (pushes carry no amounts).
- [x] `InvoiceResource` returns `items` and `refund_due`.

**Admin dashboard**
- [x] Invoice detail shows the items, marks the ones added or removed at the visit, and shows a "Refund due" flag.

**Patient app**
- [x] Invoice detail shows the item list and the final total. The invoice view model refreshes on the push. The visit detail shows the final treatments (the appointment's list follows), not the invoice lines.

**Dentist app**
- [x] The complete-visit step in `visit_workflow` gets a "Confirm treatments" list (in the Complete section, not a separate screen): booked treatments pre-ticked, add from the dentist's own expertise, remove what wasn't done. The dentist sees treatments, not prices or the invoice.

**Done when:** a dentist adds one treatment and removes another at completion, and the patient, the admin and the invoice total all agree.

**Built and merged (2026-11-11), not yet run end to end:** TEETHi-Backend `efb1a4a` (master `1683477`), TEETHi-Frontend `0039054` (master `85faa77`), TEETHi-Patient-App `083494d` (master `79906e7`) and TEETHI-Dentist-App `2131c7a` (master `0d85281`). Tests pass on merged master: backend 853, dashboard 215, patient app 214, dentist app 251. Migration run locally: all 21 local invoices got lines, and every invoice's lines add up to its stored total.

- Backend: migration `2026_11_11_000003_create_invoice_items_table` (with the backfill). `visit-outcome` takes `treatments` (1 to 20 treatment ids): each must be booked or in the dentist's active expertise (`422` otherwise). After the visit is complete the list is locked: the same list is accepted again, a different one gets `409`. A cancelled or refunded invoice also gets `409`. A removed line keeps its row (`removed_at`) so staff see what changed. The total, balance, description and status follow; an open card payment for the old balance is cancelled when the total changes. `refund_due` = paid beyond the total. New notification type `invoice_total_changed` (`invoice_id`, `appointment_id`). 853 tests pass, PHPStan max and Pint clean.
- Dashboard: treatments table on the invoice detail (Added at visit, Not done) and a Refund due badge in the list and the detail. 215 tests pass and the production build succeeds. Not checked in a browser.
- Patient app: Treatments card and Refund due on the Records invoice, which follows the list after a refresh. A foreground `invoice_total_changed` push refreshes invoices and visits; tapping it opens the invoice. 214 tests pass, analyzer has no errors or warnings. Not checked on a device.
- Dentist app: Confirm treatments checklist in Complete visit, sent as `treatments`. Visits without treatment ids send nothing and keep the booked list. 251 tests pass, analyzer has no errors or warnings. Not checked on a device.
- `docs/api/openapi.json` regenerated after the merge.

---

## P4 — Card on file, 24-hour hold, charge on completion

**Goal:** the patient pays nothing until the visit is done, and the money is guaranteed before the dentist travels.

**How it works**
1. At booking the patient saves a card (SetupIntent through the payment sheet). Nothing is charged.
2. About 24 hours before the visit a scheduled job places a hold for the invoice total: an off-session PaymentIntent with manual capture.
3. A visit booked less than 24 hours ahead gets the hold at booking, in the sheet.
4. When the dentist completes the visit (P3), the backend captures the final amount. Less than the hold: the rest is released. More than the hold: a second off-session charge for the difference.
5. If the hold fails, the patient is told to fix their card. Staff see it. The visit is not cancelled automatically.

**Backend**
- [ ] `PaymentMethodType::PayAfterVisit` (name to confirm), offered beside pay now.
- [ ] `PaymentStatus::Authorized`; `payments.authorized_at`, `capture_before`.
- [ ] Gateway: `createHold()` (manual capture, off-session, saved payment method), `capture(amount)`, `releaseHold()`. Same methods on `FakePaymentGateway`.
- [ ] `payments:place-holds` command, hourly: visits starting within 24 hours with pay-after-visit and no live hold. Idempotency key per appointment and scheduled time.
- [ ] Reschedule releases the hold; the job places a new one for the new time. Cancel releases it (the fee is P6).
- [ ] Webhook: `payment_intent.amount_capturable_updated` (hold placed), `payment_intent.payment_failed` for off-session attempts (today deliberately ignored), `payment_intent.canceled` (hold released or expired).
- [ ] Capture inside the complete-visit transaction's after-commit hook. A failed capture or a failed extra charge leaves the invoice partly paid and flags it for staff.
- [ ] Booking rule: pay-after-visit needs a usable saved card (`422` otherwise). Removing the last card with an upcoming pay-after-visit booking is refused.
- [ ] Pushes: hold placed, hold failed (with a "fix your card" action), charged after the visit.
- [ ] Confirm Apple Pay and Google Pay cards can be saved for later off-session charges in our setup. If not, pay-after-visit is card-only.

**Admin dashboard**
- [ ] Invoice list filter and badges: Card saved, Hold placed, Hold failed, Capture failed.
- [ ] A "Payment problems" view of failed holds and captures for upcoming and recent visits, with Retry hold and Retry charge (`manage_invoices`).

**Patient app**
- [ ] Payment step: "Pay after the visit" (recommended) and "Pay now". The first opens the sheet in setup mode; copy says the card is held 24 hours before and charged after.
- [ ] Visit detail shows the payment state: card saved, hold placed, charged.
- [ ] A hold-failed push opens the visit with "Update card", which reopens the sheet and retries the hold.

**Dentist app**
- [ ] Visit card and visit detail show "Payment secured" or "Payment not secured" for visits in the next 24 hours. No amounts.

**Done when:** in Stripe test mode, a visit booked two days ahead gets its hold on time, the completed visit captures the final amount, an added treatment produces a second charge, a declined card (Stripe's decline test cards) shows up for the patient, the dentist and staff, and a reschedule moves the hold.

---

## P5 — Pay later

**Goal:** apply decision D8. Pay later with no card on file can't be charged a cancellation fee and depends on staff chasing money.

**Backend**
- [ ] Option A (recommended): pay later requires a saved card, and becomes "pay after the visit" for new bookings. Option B: remove pay later for new bookings. Existing pay-later invoices keep working either way.
- [ ] If cash at the door is allowed: `POST /doctor/appointments/{appointment}/collected` records a cash amount as a succeeded payment with method `cash`, visible to staff for settlement.

**Admin dashboard**
- [ ] Reconcile keeps "record collected amount" for old pay-later invoices. A "Cash collected by dentist" view lists amounts each dentist owes TEETHi (feeds P7).

**Patient app**
- [ ] Payment step shows only the options that remain. Old pay-later visits still show "Pay now".

**Dentist app**
- [ ] Only if cash is allowed: "Mark as collected" with the amount on a completed visit.

**Done when:** a new patient with no saved card can't book without choosing a payment path that guarantees payment, per D8.

---

## P6 — Cancellation fee and partial refunds

**Goal:** the app already says "cancel free up to 24 hours before". Make it true, and let staff refund part of a payment.

**Backend**
- [ ] Policy in app settings: free-cancellation window (default 24 h), late-cancellation fee, no-show fee (fixed or percent, per D7).
- [ ] Late cancel by the patient: capture the fee from the hold and release the rest, or charge the saved card if the hold isn't placed. A fee line on the invoice.
- [ ] No-show: `POST /doctor/appointments/{appointment}/no-show` (new `no_show` status in `AppointmentStatusService`), then the no-show fee the same way.
- [ ] A dentist cancelling never charges the patient.
- [ ] Partial refund: `reconcile` (or a new `POST /admin/invoices/{invoice}/refunds`) takes an amount and a payment, with an idempotency key per refund. Replaces "refund the last payment in full".
- [ ] `POST /admin/invoices/{invoice}/waive-fee` (`manage_invoices`).

**Admin dashboard**
- [ ] App settings: the cancellation policy fields.
- [ ] Invoice detail: Refund with an amount and a reason, per payment; Waive fee.

**Patient app**
- [ ] Cancel dialog states the fee before confirming when inside the window. Booking and payment steps show the policy from settings, not fixed copy.
- [ ] Invoice shows the fee line and any refund.

**Dentist app**
- [ ] "Patient didn't show" on an accepted visit after its start time.

**Done when:** a cancellation inside the window charges the fee and releases the rest of the hold, one outside charges nothing, a no-show charges its fee, and staff can refund half of a payment.

---

## P7 — Dentist earnings and bank-transfer payouts

**Goal:** every paid visit records what the dentist is owed, dentists can see it, and finance can pay it and prove it.

**Backend**
- [ ] Fee model in app settings, per D2 and D5: platform fee (flat or percent), who bears processing fees.
- [ ] `doctor_earnings`: one row per invoice payment event (gross, platform fee, processing fee share, net, status `pending` → `payable` → `paid`). Refunds, cancellation fees and lost disputes create reversing rows. Rows are never edited.
- [ ] An earning becomes `payable` when the visit is completed and paid, after a hold period for refunds (length is a D5 detail).
- [ ] `payout_batches`: finance closes a period, the batch freezes its rows and totals, and is later marked paid with a bank reference.
- [ ] `GET /doctor/earnings` (summary and paged rows), `GET /doctor/payouts`.
- [ ] `GET/POST /admin/payouts`, `POST /admin/payouts/{batch}/mark-paid`, CSV export per batch, per-dentist statement PDF, and a per-year total per dentist for 1099 forms. New permissions `view_payouts` / `manage_payouts`.
- [ ] Bank details are **not** stored in TEETHi in this phase. Finance keeps them in its own system; the export carries the dentist's id and name.
- [ ] Push to the dentist when a payout is marked paid.

**Admin dashboard**
- [ ] App settings: fee model.
- [ ] New Payouts feature: earnings by dentist, create batch, export, mark paid, statements.
- [ ] Doctor detail: an Earnings tab.

**Patient app**
- No change.

**Dentist app**
- [ ] Earnings screen: this month, pending, paid, and the per-visit rows (visit date, patient name, net amount).
- [ ] Bring back the `payout` inbox type, fed by the payout push (dentist plan §11, dropped in Phase 9).
- [ ] Replace the hidden "payments" profile block with a link to Earnings.

**Done when:** a paid visit creates an earning, a refund reverses it, finance closes a month, exports it, marks it paid, and the dentist sees the payout.

---

## P8 — Stripe Connect payouts (later)

**Start when:** manual payouts cost finance real time, or the dentist count makes bank transfers error-prone. Needs D4.

**Outline only, to be detailed then:**
- Backend: connected accounts per dentist, onboarding links, account status webhooks, payment split per the P7 fee model (D4: TEETHi is the merchant, so the charge stays on TEETHi's account and the dentist's share is transferred), payouts and 1099s through Stripe. The P7 ledger stays as the record and reconciles against Stripe's transfers.
- Admin dashboard: dentist payout status, onboarding state, transfer failures.
- Dentist app: "Set up payouts" onboarding, payout status, bank details managed in Stripe's hosted flow.
- Patient app: no change, unless D4 changes the name on the card statement.

---

## 4. Out of scope

- Buy now, pay later and dental financing partners. They are payment-method settings or partner agreements; revisit after P4.
- Insurance. See the insurance summary; P2's receipt is its first building block.
- Sales tax. Dental services are generally untaxed in most states; the accountant confirms for California (D1).

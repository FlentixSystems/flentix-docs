# Weekly Mailroom Operations — Implementation Plan

> **Execution model:** built via **Lovable** (one-writer-to-Lovable rule). Each task = one coherent
> Lovable build + an explicit **Verify** (DB query / file read / app test). No local test runner;
> Lovable typechecks each edit. Build **phase by phase**, verify before moving on. Design:
> [`DESIGN_weekly-mailroom-operations.md`](DESIGN_weekly-mailroom-operations.md).

**Goal:** Build the daily→weekly mailroom cycle — a mail-credit balance + top-ups, hold-by-default
per-item disposition, a Thursday-9pm off-session weekly charge reusing the booking balance-charge
machinery, and a pay-before-Correos hold/flag.

**Architecture:** New per-subscriber mail-credit ledger + a weekly `vo_mail_charges` table that
mirrors the existing bookings balance-charge fields, so the admin failure UI, retry cron, and
Factura all reuse the proven booking pattern. Charging is off-session on the operator's Stripe
Connect account (Medacrii app fee on NET, 21% IVA), draw-from-balance-else-card.

**Tech stack:** Vite/React/TS + shadcn frontend; Supabase (Postgres + Deno edge functions); Stripe
Connect (stripe@17.5.0); pg_cron; Brevo email.

## Global Constraints (verbatim, apply to every task)
- All prices **ex-IVA**; customer pays **+21% IVA**; operator = MoR retains IVA; **Medacrii fee on
  NET only, never IVA**.
- **Additive only** — must NOT break the working checkout / webhook / booking / telecom paths.
- Mail-credit **balance tracked in NET** cents. Top-ups + the €10 seed carry **+21% IVA**
  (prepayment/anticipo). Weekly draws take **NET** from balance; PAYG charges add 21% at point of service.
- **€5 handling fee**: PAYG only (or ad-hoc); credit-balance members' scheduled forwards = no handling fee.
- Postage rates = the `CORREOS_RATES` table already in `DispatchModal` (€0.85 → €10.20).
- **Hold by default:** mail item `disposition` defaults to `hold`; only `forward` items are charged/sent.
- Reuse (do not reinvent): `charge-balance`, `charge-due-balances`, `create-balance-payment-link`,
  `mark-balance-paid-offline`, `send-invoice-email`, `issue-credit-note`, `send-vo-mail-digest`.
- Never echo `RETELL_API_KEY` / `SUPABASE_SERVICE_ROLE_KEY` / Stripe secret. Service-role-only edge
  functions use the constant-time bearer gate (as in `vo-overage-topup`).

---

## PHASE 1 — Mail-credit ledger, seed deposit (with IVA), and top-ups

### Task 1.1 — Ledger schema
**Files:** DB migration.
**Build:** Create `vo_mail_credit` (`subscriber_id` uuid PK/unique FK → virtual_office_subscribers,
`balance_net_cents` int default 0, `currency` text default 'eur', `updated_at`) and
`vo_mail_credit_ledger` (`id`, `subscriber_id`, `entry_type` text check in
('seed','topup','draw','refund','adjustment'), `net_cents` int, `iva_cents` int default 0,
`reference` text, `stripe_payment_intent_id` text, `created_at`). RLS: client-ownership SELECT via
`current_user_subscriber_ids()` (the helper built for the receptionist console); service-role full;
admin via `is_current_user_admin()`. Add a `SECURITY DEFINER` helper
`vo_mail_credit_apply(_subscriber uuid, _entry_type text, _net int, _iva int, _ref text, _pi text)`
that inserts the ledger row and upserts the running `balance_net_cents`.
**Verify:** `SELECT * FROM information_schema.columns WHERE table_name IN ('vo_mail_credit','vo_mail_credit_ledger');`
returns the columns; `SELECT vo_mail_credit_apply(<a real subscriber>, 'adjustment', 0, 0, 'test', null);`
runs clean; then delete the test row.

### Task 1.2 — Seed the €10 deposit WITH IVA into the balance on first payment
**Files:** Modify `create-virtual-office-checkout/index.ts`; `stripe-webhook/index.ts`.
**Build:** (a) In the checkout, the Deposit-plan one-time line (added in commit 1591848 with NO tax)
must now carry the **21% IVA** — add the ES VAT tax rate to that `add_invoice_items` price (mirror
how the tier line gets `lineItemsWithTax`), because it's a prepayment, not a security deposit.
Rename the product to "Mailroom credit (initial €10)". (b) In the webhook VO block, when the order's
`mailroom_plan === 'deposit'`, call `vo_mail_credit_apply(subscriber_id,'seed',1000,210,'signup deposit', <invoice PI>)`
once (idempotent — guard on an existing 'seed' ledger row for that subscriber).
**Verify:** run a Deposit-plan test checkout → order total = €35 + €10 + 21% IVA on both = **€54.45**
(35+10=45 net, ×1.21). `SELECT balance_net_cents FROM vo_mail_credit WHERE subscriber_id=<new order>`
= 1000. `SELECT entry_type,net_cents,iva_cents FROM vo_mail_credit_ledger WHERE subscriber_id=<new order>`
= one 'seed' row (1000, 210).

### Task 1.3 — Top-up checkout function
**Files:** Create `supabase/functions/create-mail-topup-checkout/index.ts`; add to `config.toml`
(verify_jwt = true).
**Build:** verify_jwt=true; resolve the caller → their subscriber (via `current_user_subscriber_ids()`).
Body `{ amount_eur: 10|20|30 }` (reject others). Create a **one-off** (mode:'payment') Stripe Checkout
Session on the hub's connected account for `amount_eur*100` NET **+21% IVA**, Medacrii app fee on NET
(reuse the fee logic), metadata `{ kind:'mail_topup', subscriber_id, net_cents }`, success/cancel →
`/portal/mailroom`. Return `{ url }`.
**Verify:** invoke with a logged-in VO client + `{amount_eur:10}` → returns a Stripe URL; with
`{amount_eur:7}` → 400.

### Task 1.4 — Webhook: credit top-ups to the ledger
**Files:** Modify `stripe-webhook/index.ts`.
**Build:** handle `checkout.session.completed` where `metadata.kind === 'mail_topup'` →
`vo_mail_credit_apply(subscriber_id,'topup', net_cents, round(net_cents*0.21), 'portal top-up', <PI>)`;
idempotent on the session id.
**Verify:** complete a €10 test top-up → balance increases by 1000; one 'topup' ledger row (1000, 210).

### Task 1.5 — Portal balance + top-up UI
**Files:** Modify `src/pages/PortalMailroom.tsx` (`/portal/mailroom`).
**Build:** show the client's current mail-credit balance (net €, from `vo_mail_credit`); three top-up
buttons **€10 / €20 / €30** that call `create-mail-topup-checkout` and redirect to the returned url;
a low-balance note ("top up before Thursday or that week's forwards go Pay-As-You-Go"). Preview mode
(disabled) when no subscriber.
**Verify:** log in as a VO client → balance shows; click €10 → lands on Stripe checkout; on return the
balance reflects the top-up.

---

## PHASE 2 — Per-item disposition + daily digest

### Task 2.1 — Disposition column
**Files:** DB migration.
**Build:** add `disposition text NOT NULL DEFAULT 'hold' CHECK (disposition IN ('hold','forward','scan_only'))`
to `virtual_office_mail_items`; add `disposition_set_by` text ('client'|'admin'|'default') default 'default',
`disposition_set_at timestamptz`.
**Verify:** `SELECT disposition FROM virtual_office_mail_items LIMIT 1;` returns 'hold' for existing rows.

### Task 2.2 — Client per-item disposition UI
**Files:** Modify `src/pages/PortalMailroom.tsx`.
**Build:** list the client's non-dispatched mail items with a per-item control (Hold / Forward / Scan-only),
writing `disposition` (+ set_by='client', set_at) via a `save-mail-disposition` edge fn (verify_jwt=true,
ownership-checked) or direct RLS-guarded update. Default shown = Hold. Copy: "Anything left on Hold stays
free; choose Forward to have it posted to you on Friday."
**Verify:** as a client, set an item to Forward → `SELECT disposition,disposition_set_by FROM
virtual_office_mail_items WHERE id=<item>` = ('forward','client').

### Task 2.3 — Schedule + reword the daily digest
**Files:** Modify `send-vo-mail-digest/index.ts` (wording); DB migration (pg_cron).
**Build:** (a) reword the digest email: add a CTA button to `/portal/mailroom` and a line "Anything you
don't action stays on Hold (free) — choose Forward for items you want posted this Friday." (b) Add a
pg_cron job `send-vo-mail-digest-hourly` (`0 * * * *`) that `net.http_post`s the function with the
service-role key (the function's 19:00-Madrid gate does the once-a-day firing). Model the cron on the
existing `send-kyc-reminders-hourly` job.
**Verify:** `SELECT jobname,schedule FROM cron.job WHERE jobname='send-vo-mail-digest-hourly';` returns
the row; invoke the function with `{force:true}` (service role) → returns a processed/sent summary with no error.

---

## PHASE 3 — Weekly charge (Thu 9pm) reusing the booking pattern

### Task 3.1 — Weekly charge table
**Files:** DB migration.
**Build:** create `vo_mail_charges` (`id`, `subscriber_id`, `cycle_date date`, `net_cents`, `iva_cents`,
`total_cents`, `paid_from text check in ('balance','card')`, `charge_status text check in
('pending','succeeded','failed','held','paid_offline') default 'pending'`, `payment_intent_id`,
`attempts int default 0`, `last_attempt_at`, `failed_at`, `error`, `item_ids uuid[]`, `created_at`).
RLS: admin + service-role manage; client SELECT own. Unique (`subscriber_id`,`cycle_date`).
**Verify:** columns present; a manual insert + delete round-trips.

### Task 3.2 — `charge-mail` (off-session charger)
**Files:** Create `supabase/functions/charge-mail/index.ts`; add to config.toml (verify_jwt=false +
constant-time service-role gate like `vo-overage-topup`).
**Build:** body `{ chargeId, invokedBy:'cron'|'manual' }`. Model on `charge-balance`: hard-guard against
double-charge (skip if status ∈ succeeded/paid_offline). Load the `vo_mail_charges` row. Resolve the
subscriber's saved card (from `engine_metadata.telecom_payment_method_id` or the subscription's default
PM, on the connected account — same fallbacks as `vo-overage-topup`). Then:
  - If `paid_from='balance'` was pre-decided AND balance covers `net_cents`: draw via
    `vo_mail_credit_apply(sub,'draw',-net,-0,'weekly forward '||cycle_date, null)`, set status
    'succeeded', fire `send-invoice-email` (Factura). NO card charge.
  - Else (PAYG or balance shortfall): off-session `paymentIntents.create` on the connected account for
    `total_cents` (net+IVA), `application_fee_amount = net - kickback`-style Medacrii fee on NET,
    metadata `{ fx_kind:'mail', subscriber_id, cycle_date }`; on success set 'succeeded' + Factura; on
    Stripe error set 'failed' + persist error/attempts/failed_at (mirror `charge-balance`'s `fail()`).
Return the same success/failure shape as `charge-balance`.
**Verify:** with a test `vo_mail_charges` row (small amount, a subscriber with a saved test card) invoke
`{chargeId,invokedBy:'manual'}` → status flips to 'succeeded', a PI id is recorded; a bad-card test row →
status 'failed' with an error string. (Test-mode Stripe.)

### Task 3.3 — `charge-due-mail` cron
**Files:** Create `supabase/functions/charge-due-mail/index.ts` (service-role gate); DB migration (pg_cron).
**Build:** model on `charge-due-balances`: scan `vo_mail_charges` where `charge_status IN
('pending','failed')` AND `cycle_date <= today` → invoke `charge-mail` per row (invokedBy:'cron').
Idempotent. Two pg_cron jobs: `charge-mail-thu` (`0 20 * * 4` UTC ≈ 21:00 Madrid Thu — adjust for
DST/target 21:00 local) and `charge-mail-fri-retry` (`0 6,7,8 * * 5` — Friday morning retries before
10:00). Both `net.http_post` with the service-role key.
**Verify:** `SELECT jobname,schedule FROM cron.job WHERE jobname LIKE 'charge-mail-%';` returns both;
manual invoke of `charge-due-mail` (service role) processes pending rows without error.

### Task 3.4 — Admin: finalize the week's charge (Thursday) from the dispatch panel
**Files:** Modify `src/components/admin/VirtualOfficePanel.tsx` (`DispatchModal`);
`send-predispatch-invoice/index.ts` (already handles the €5 flag).
**Build:** in the DispatchModal, when the admin finalizes a dispatch, additionally create/update the
`vo_mail_charges` row for that subscriber+cycle_date: compute `net_cents` from the **forward-disposition**
items' postage (+ €5 handling only for PAYG members), `iva_cents` = 21% of net (unless drawing from
balance where IVA was pre-collected), `paid_from` = 'balance' if the credit member's balance covers it
else 'card', status 'pending'. Keep the existing pre-dispatch invoice email as the client's query-window
notice. (No auto-charge here — the Thu-9pm cron does it.)
**Verify:** finalize a dispatch for a test client → a `vo_mail_charges` 'pending' row appears with the
right net/iva/paid_from; the pre-dispatch email still sends.

---

## PHASE 4 — Admin visibility + pay-before-Correos gate

### Task 4.1 — Surface mail-charge status + failure actions in the admin
**Files:** Modify `src/components/admin/VirtualOfficePanel.tsx`.
**Build:** on the order row / detail modal, show this cycle's `vo_mail_charges` status badge
(pending/succeeded/**failed**/held). For 'failed', show a red banner + the three actions reusing the
booking pattern: **Retry** (`charge-mail` invokedBy:'manual'), **Send payment link** (a
`create-mail-payment-link` sibling of `create-balance-payment-link`), **Mark paid offline** (a
`mark-mail-paid-offline` sibling), plus a **"Payment failed" email** button (Brevo, branded).
**Verify:** with a 'failed' test charge → the red banner + all four actions render; Retry re-invokes
charge-mail; Mark-paid-offline flips status to 'paid_offline'.

### Task 4.2 — Correos send gate
**Files:** Modify `src/components/admin/VirtualOfficePanel.tsx` (`DispatchModal` confirm/handover step).
**Build:** block the "confirm dispatch / cleared for Correos" action unless this cycle's
`vo_mail_charges.charge_status` ∈ ('succeeded','paid_offline'). If not paid, show "Payment not received —
mail held" + keep the item(s) `hold`/undispatched and flag the order. A held+flagged badge on the row.
**Verify:** a client with a 'failed'/'pending' charge → the confirm-dispatch button is disabled with the
"held" message; after marking paid → it enables.

---

## PHASE 5 — Balance refund on closure

### Task 5.1 — "Refund balance" admin action
**Files:** Create `supabase/functions/refund-mail-balance/index.ts` (service-role/admin gate); modify
`VirtualOfficePanel.tsx`.
**Build:** admin button "Refund mail balance". The function: read `vo_mail_credit.balance_net_cents`; if
>0, Stripe-refund the unused amount to the client's card (refund the most recent top-up PIs up to the
balance, on the connected account), issue a **Factura Rectificativa** via `issue-credit-note` for
`-net` and `-21% IVA`, and `vo_mail_credit_apply(sub,'refund',-balance,-round(balance*0.21),'closure refund', <refund id>)`
zeroing the ledger.
**Verify:** on a test subscriber with a €5 net balance → the action refunds €6.05 gross (5 + 1.05 IVA),
a credit note is issued, and `balance_net_cents` → 0.

---

## Self-review notes
- **Spec coverage:** every §1–§9 design element maps to a task (ledger→1.1; seed+IVA→1.2; top-up→1.3-1.5;
  disposition+digest→2.x; weekly charge→3.x; admin+gate→4.x; refund→5.1). §7 (T2/T3 included) needs no new
  build — handled by the existing €5-checkbox default (off for non-PAYG).
- **Anticipo VAT** (confirmed by founder): balance NET, IVA at top-up/seed, draws NET, refund reverses
  base+IVA — reflected in 1.2/3.2/5.1.
- **Dependencies:** Phase 1 → 2 → 3 → 4 → 5 (each builds on the prior). Phase 3 depends on 1 (balance) + 2
  (disposition). Build in order.

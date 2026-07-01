# SPEC — Mail Handling, Scanning & Forwarding (Virtual Office)

**Status:** v1.0 — written 2026-07-01 (founder-directed, after the first Tier-1 test surfaced
it). This model was **built in the onboarding + admin dispatch code but never documented**;
this spec is the source of truth. It **expands** the Constitution's under-described lines
(T1 = "basic mail handling", T2 = "+ forwarding") — reconcile the Constitution wording as a
follow-up (see §8).

Money rule (inherited, non-negotiable): **all prices below are ex-IVA.** The customer pays
+21% IVA; the operator is Merchant of Record and retains the IVA; the Medacrii platform fee is
swept on NET, never on IVA.

---

## 1. What every Virtual Office tier gets (mail)
- **Registered business address** + the operator **receives** the customer's mail.
- **Digital mail scanning → scan-to-email.** Incoming items are scanned and delivered to the
  customer (portal + email). **Included from Tier 1 up.** (⚠️ current bug — checkout hardcodes
  `scan_to_email: false`; see §6.)
- **Mail forwarding** (physical re-mailing of the item to the customer's own address) — cadence
  and billing below.

## 2. Forwarding cadence — WEEKLY BATCH (not daily)
- Forwarding runs as a **weekly batch at end of week**. The operator **bundles ALL of a
  customer's accumulated mail into ONE dispatch** and sends it together — everyone processed at
  the same time, once a week.
- Rationale: one handling event + one postage + one admin fee per customer per week; efficient
  and predictable.

## 3. Billing — two mailroom plans (chosen at Tier-1 onboarding)
| Plan | Deposit | Per-mailing admin fee | Postage |
|---|---|---|---|
| **Deposit Member** *(recommended)* | **€10.00 refundable** verification deposit | **€0** | pays postage per weekly bundle |
| **Pay-As-You-Go (PAYG)** | none | **€5.00 flat per MAILING** | pays postage per weekly bundle |

**The €5 is per MAILING (per weekly bundle/dispatch), NOT per item.** A PAYG customer with 4
letters bundled into one weekly send pays **one** €5 admin fee (+ the bundle's postage), not €20.
(Decision confirmed by founder 2026-07-01; rationale — postage already scales with volume/weight,
so per-mailing avoids double-penalising and matches the batch model.)

## 4. Postage — the "cost of forwarding"
- Charged **per weekly bundle**, from the admin dispatch price list (by weight/type). Current
  values in the admin "Process Dispatch" panel: **Ordinary documents ≤500g = €2.75**,
  **Certified documents ≤500g = €6.70** (plus any further weight/type tiers held there).
- Applies to **both** plans — the €10 deposit removes the **admin fee**, not the postage.

## 5. What gets charged at the weekly "mail checkout" (per customer bundle)
1. **Postage** (§4), on the bundled weight/type.
2. **+ €5.00 admin fee IF the customer is PAYG** (per mailing). Deposit members = €0 admin fee.
3. One charge per weekly bundle (+21% IVA on top, per the money rule).

## 6. Current state & gaps (as of the first T1 test, 2026-07-01)
- ⚠️ The mailroom plan is **collected in the onboarding UI** (wording in
  `src/pages/VirtualOfficeOnboarding.tsx`, Mailroom billing step) but the checkout **DROPS it** —
  it is NOT in the order's `engine_metadata` (verified 2026-07-01: `mailroom_plan` absent), so the
  operator has **no record of Deposit vs PAYG** on the order.
- ⚠️ The **€10 deposit is NOT charged** — a T1 order totals **€42.35** (€35 + 21% IVA); no deposit
  line is ever added. The whole mailroom-billing model is currently **UI-only**.
- ✅ The **postage price list** exists in the admin `DispatchModal`.
- ❌ **Checkout hardcodes** `forwarding: "none"` and `scan_to_email: false` → the admin row shows
  a misleading "Forwarding: No Forwarding / Scan to email: OFF" that ignores both the tier's scan
  inclusion and the customer's mailroom plan.
- ❌ **The admin panel never surfaces `mailroom_plan`** → the operator can't see whether a
  customer is Deposit or PAYG.
- ❌ **No weekly-batch dispatch mechanism** → dispatch is currently ad-hoc per order; there's no
  "bundle a customer's week of mail and send once" flow.
- ❌ **The €5 PAYG admin fee is NOT auto-added at dispatch** → `DispatchModal` charges postage but
  doesn't add the €5 for PAYG customers.

## 7. To build (Finish-VO / T1 readiness — not yet done)
1. Stop hardcoding `forwarding` / `scan_to_email`. Drive **scan-to-email from tier inclusion**
   (ON for T1+); represent forwarding via the **mailroom plan**, not a dead on/off flag.
2. **Surface the mailroom plan** (Deposit €10 / PAYG €5-per-mailing) on the admin row + detail
   modal (fits the admin-UX pass already logged).
3. **Weekly batch dispatch** — bundle each customer's accumulated mail and dispatch once per week
   (operator-triggered end-of-week run, or a batched queue).
4. At the weekly **mail checkout**: **auto-add the €5 admin fee for PAYG** customers (per mailing)
   on top of postage; €0 for Deposit members; then charge (NET + 21% IVA, Medacrii fee on NET).
5. **Reconcile the Constitution** T1/T2 mail wording with this spec.

## 8. Decisions (resolved by founder 2026-07-01)
- **Postage basis: PER BUNDLE** — the total weight of the weekly bundle, not per item. ✅
- **VAT: charge IVA (21%) on EVERYTHING the customer is billed — including the postage we pass
  through. NO exempt lines on the customer invoice.** Nuance (input side only): **Correos** does
  not charge US IVA on the postage (universal-service exemption), so there is no input IVA to
  reclaim on that cost — BUT our OUTPUT (the postage + handling we bill the customer) is a VATable
  service, so we add 21% on the full amount. (Private couriers charge us IVA as normal.) So every
  line the customer sees — subscription, €5 handling, postage — carries 21% IVA.
- **T2 / T3 "included" forwarding = frequency AND postage included, always** (T2 twice-monthly,
  T3 weekly). Those scheduled sends cost the customer nothing extra.
- **The €5 is a HANDLING / AD-HOC fee, applied via a CHECKBOX at dispatch:**
  - **Auto-ON for PAYG (T1)** — their per-mailing model (€5 on every bundle).
  - **Admin-discretion, default OFF, for Deposit + T2/T3** — the operator ticks it for an
    **ad-hoc / immediate / out-of-schedule** physical send a customer requests (they always keep
    the free email scan; the €5 is the charge for jumping the weekly batch).
  - Net: scheduled sends carry no handling fee for Deposit/T2/T3; ad-hoc immediate sends = €5.
- **€10 deposit = a PREPAID CREDIT FUND, charged WITH 21% IVA** (founder corrected 2026-07-01 — it
  is NOT a no-IVA security deposit; a prepayment/*anticipo* for future mailing triggers IVA at
  payment). Charge **€10 + €2.10 IVA = €12.10** gross, IVA declared to Hacienda on that invoice.
  On refund of the unused balance, issue a **Factura Rectificativa (credit note)** reversing
  **base + IVA** — e.g. €5 unused → **−€5.00 −€1.05 IVA = −€6.05** returned to the client — and
  reclaim the €1.05 via **Modelo 303**. (`issue-credit-note` exists; the "Refund deposit" admin
  action wires it + the Stripe refund.) ⚠️ The current Part-1 build charges the €10 with **NO IVA**
  — to be corrected to +IVA once the fund model (§11) is locked.
- **Scan-to-email:** included T1+ (per-item cap TBD if ever needed).

## 9. Code references
- Onboarding mailroom-plan selection + wording: `src/pages/VirtualOfficeOnboarding.tsx`
  (`mailroomPlan`, "Mailroom billing" step) — and the hardcodes `forwarding: "none"`,
  `scan_to_email: false` in the `create-virtual-office-checkout` invoke body.
- Postage price list + dispatch UI: `src/components/admin/VirtualOfficePanel.tsx` → `DispatchModal`.
- Mail inventory logging: same panel → `LogMailModal`.

## 10. Build status (2026-07-01)
**DONE + deployed** (commits `1591848` / `28bc1f8a` / `d9505ff5`):
- ✅ `mailroom_plan` now **persisted** on the order (checkout metadata → `engine_metadata`).
- ✅ **€10 refundable deposit collected** for Deposit customers (one-time on the first invoice via
  `subscription_data.add_invoice_items`, **NO IVA** — outside VAT scope).
- ✅ **Scan-to-email flagged ON** (included T1+).
- ✅ Onboarding: the deposit/PAYG step **states the real costs**; tier cards carry mail-handling +
  ad-hoc captions.
- ✅ Admin: **mailroom plan shown** on the order row + detail modal (Deposit €10 / PAYG €5/mailing).
- ✅ The €5 handling fee is now a **checkbox at dispatch** — auto-ON for PAYG, OFF for Deposit/T2-T3
  (tick for an ad-hoc send); the emailed pre-dispatch invoice total matches the on-screen total.

**REMAINING (follow-ups):**
- ⬜ **Weekly batch dispatch** mechanism (bundle a customer's week of mail → one send). Dispatch is
  currently ad-hoc per order.
- ⬜ **"Refund deposit" admin action** (Stripe refund of the €10 + auto credit note) on closure.
- ⬜ Forwarding-cadence wiring for T2/T3 (the `forwarding` field still displays generically).
- ⬜ Reconcile the Constitution's T1/T2 mail wording with this spec.
- ⬜ Accountant confirms: deposit-VAT (currently €10 no-IVA) + postage VAT.

## 11. OPEN MODEL DECISION (2026-07-01) — drawn-down FUND vs flat deposit
The founder's "credit fund for future mailing costs" + "€5 balance remaining" framing implies the
€10 is a **prepaid mail-credit BALANCE that postage/handling DRAW DOWN** (€5 used → €5 left), topped
up when low, unused refunded via credit note — matching the "always running in credit" philosophy.
That differs from the **current build** (flat €10 deposit; postage billed SEPARATELY at each
dispatch). The two also bill VAT differently:
- **FUND model:** IVA charged **once** on each top-up; each dispatch **draws NET** from the
  pre-taxed balance (no second IVA); refund unused base + IVA via Factura Rectificativa.
- **FLAT model:** deposit taxed once; **each dispatch** taxed separately (postage + €5 + 21% IVA).
**Decision needed before finalising the deposit build.** If FUND: build a per-subscriber
mail-credit ledger (analogous to the telecom `vo_account_minutes`) + drawdown at dispatch +
top-up + the Factura-Rectificativa refund. If FLAT: just add 21% IVA to the one-time €10 line and
the "Refund deposit" action reverses €10 + IVA.
**DECIDED 2026-07-01: FLAT deposit.** (But see §12 — the founder also wants a client top-up
balance, which reintroduces a drawdown element; resolve there.)

## 12. Weekly mailroom OPERATIONS flow (founder intent 2026-07-01) + current state
**Intended weekly cycle:**
1. **Daily:** admin logs incoming letters/parcels per client.
2. **End of each day:** auto-email to any client who received mail that day.
3. **Auto top-up** of the client's prepaid mail credit when low (trigger TBD).
4. **Thursday:** admin sets that week's final mail charges per client (batch); then all invoices sent.
5. **By 10:00 Friday:** payment taken. Client has an overnight window (Thu → 10:00 Fri) to QUERY.
6. **After 10:00 Friday:** all charges paid → admin clear to send to Correos.
7. **Payment gate:** mail is NOT sent unless payment is received. Payment DECLINE (any tier) → a
   FLAG so that client's mail is held.
Plus: **client-facing top-up** in `/portal/mailroom` (€10 / €20 / €30 options — client picks by usage).

**Current state (verified 2026-07-01) — mostly NOT built:**
- ✅ Daily mail logging exists (`LogMailModal` → `virtual_office_mail_items`).
- ⚠️ EOD digest exists (`send-vo-mail-digest`, bilingual, fires 19:00 Europe/Madrid) but is **NOT
  scheduled** (no pg_cron job) → never runs. Quick fix = add the hourly cron.
- ❌ **No payment is actually taken.** `send-predispatch-invoice` only EMAILS a "card debited at
  courier handover in ~24h" notice; the dispatch `confirm` step just records the dispatch + resets
  counters — it does NOT charge the card. The email promises a debit that nothing performs.
- ❌ Dispatch is **per-order + manual** (open each client's DispatchModal, set qty, optionally email
  the notice, confirm) — NO Thursday batch, no "send all invoices together."
- ❌ No mail auto-top-up (telecom only), no payment-before-send gate, no declined-payment flag, no
  structured query window, no client-facing top-up.

**Tension to resolve:** FLAT deposit (postage billed separately) vs a client **top-up balance**
(€10/20/30) that weekly charges draw from — a top-up balance IS a drawdown model. Decide: does the
Friday mail charge (a) draw from a prepaid top-up BALANCE (card charged / auto-top-up when low), or
(b) charge the card directly each Friday (top-ups just a convenience prepay)?

**→ This is its own focused build ("Weekly Mailroom Operations").** Sub-builds: schedule the EOD
digest; a mail-charge PAYMENT mechanism (Stripe off-session on the connected account, IVA, Medacrii
fee) with the Friday cutoff + query window; the payment-required-to-send gate + declined flag; the
weekly batch charge-set UI; the portal top-up (+ a mail-credit ledger if the balance model wins).

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
- ✅ The chosen mailroom plan **is captured** at checkout (`mailroom_plan: "deposit" | "payg"`
  on the order) — wording lives in `src/pages/VirtualOfficeOnboarding.tsx` (Mailroom billing step).
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

## 8. Open questions / decisions for founder
- **Postage basis:** per **bundle** (total weight) or per item? The batch model implies per
  bundle by weight — please confirm.
- **Deposit refund mechanics:** when/how is the €10 refunded (on account closure)? Held where?
- **T2 / T3 forwarding inclusions:** tier copy says T2 "forwarding included twice monthly", T3
  "forwarding weekly". Do these **include postage**, or just waive the admin fee? How do they sit
  against the weekly batch?
- **Scan-to-email limits:** unlimited on T1, or a monthly item cap?
- **Postage VAT treatment:** postage may be VAT-exempt/pass-through vs the €5 admin fee (standard
  21%) — confirm with the accountant.

## 9. Code references
- Onboarding mailroom-plan selection + wording: `src/pages/VirtualOfficeOnboarding.tsx`
  (`mailroomPlan`, "Mailroom billing" step) — and the hardcodes `forwarding: "none"`,
  `scan_to_email: false` in the `create-virtual-office-checkout` invoke body.
- Postage price list + dispatch UI: `src/components/admin/VirtualOfficePanel.tsx` → `DispatchModal`.
- Mail inventory logging: same panel → `LogMailModal`.

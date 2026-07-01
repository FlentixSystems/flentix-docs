# DESIGN — Weekly Mailroom Operations

**Status:** v1.0 — approved by founder 2026-07-01 (via brainstorming). This is the implementable
design for the mailroom's daily→weekly operational cycle. Background/rules: [`SPEC_mail-handling.md`](SPEC_mail-handling.md)
(this design **supersedes** the evolving decisions in that spec's §8–§12 wherever they differ —
notably: the deposit is a **drawn-down credit balance**, and unchosen mail **holds by default**).

Money rule (inherited): all prices ex-IVA; customer pays +21% IVA; operator = Merchant of Record,
retains IVA; Medacrii platform fee swept on NET, never on IVA.

---

## 1. Billing model
Two states, per Tier-1 subscriber (T2/T3 have scheduled forwarding + postage **included**, see §7):

**Credit-balance member ("Deposit" plan)**
- The **€10 signup deposit seeds a mail-credit BALANCE** (charged **+21% IVA** — it's a prepayment/
  *anticipo*, not a security deposit).
- Client **tops up €10 / €20 / €30** in `/portal/mailroom` (each top-up +21% IVA).
- Weekly mail charges (postage only — **no €5 handling fee**, that's the perk) **draw NET from the
  balance**.
- If the balance can't cover a week's forward → that client **reverts to PAYG for that charge**
  (card charged directly; €5 handling then applies). Reminder to top up fires before Thursday.

**PAYG member**
- No balance. Each week the card is charged **postage + €5 handling + 21% IVA** directly.

**VAT handling — CONFIRMED by founder 2026-07-01 (accounting checked):** balance tracked in **NET**;
top-ups charged **NET + 21% IVA** (IVA declared at top-up); weekly draws take **NET** from the
balance (IVA pre-collected at top-up); PAYG charges add 21% at point of service. **Refund of unused
balance** on closure = a **Factura Rectificativa** reversing **base + IVA** (e.g. €5 unused → −€5.00
−€1.05 = −€6.05), reclaimed via Modelo 303.

## 2. Data
- **`vo_mail_credit`** (new) — per-subscriber balance ledger, modelled on the telecom
  `vo_account_minutes`: `subscriber_id`, `balance_net_cents`, plus a `vo_mail_credit_ledger`
  (entries: `topup` / `draw` / `refund`, amount, ref, created_at).
- **Per-item disposition** on `virtual_office_mail_items`: `disposition` enum `hold` (DEFAULT) /
  `forward` / `scan_only`, set by the client (or admin) each cycle.
- **`vo_mail_charges`** (new) — one row per client per weekly cycle, mirroring the **bookings
  balance-charge fields** so the existing admin failure UI works unchanged: `subscriber_id`,
  `cycle_date`, `net_cents`, `iva_cents`, `total_cents`, `paid_from` (`balance`|`card`),
  `charge_status` (`pending`|`succeeded`|`failed`|`held`), `payment_intent_id`, `attempts`,
  `error`, `failed_at`, item ids covered.

## 3. Daily flow
1. **Admin logs incoming mail** each day *(exists — `LogMailModal` → `virtual_office_mail_items`)*.
2. **7pm digest** — schedule the **existing `send-vo-mail-digest`** cron (built, fires 19:00
   Europe/Madrid, currently unscheduled). Extend its wording: list the day's items **+ a CTA to
   `/portal/mailroom` to choose each item's fate**; state that **anything not actioned stays HELD**
   (free) — nothing is forwarded or charged unless the client asks.

## 4. Client portal (`/portal/mailroom`)
- **Per-item disposition:** hold (default) / forward / scan-only.
- **Balance display** + **top-up €10 / €20 / €30** (Stripe, +IVA).
- **Low-balance reminder** (and a pre-Thursday nudge: top up or the week's forwards go PAYG).

## 5. Weekly cycle (timing locked)
| When | Action |
|---|---|
| **Thu, by 4pm** | Admin finalizes each client's **forward list** + charges (per-order, in the existing DispatchModal). Sends each client the pre-charge invoice → opens the query window. |
| **Thu 4–8pm** | **Client query window** — reply to adjust before the charge. |
| **Thu ~9pm** | **Charge** (scheduled job): for each client with a finalized forward total, **draw from balance** (credit member) else **charge the card** (PAYG / balance-shortfall). Off-session, on the connected account, +IVA, Medacrii app fee. Official **Factura on success**. Deliberately clear of the 8:15am booking-charge cron. |
| **Fri, before 10am** | The job **re-attempts declines** (like `charge-due-balances`); admin chases non-payers — **Retry / Send payment link / Mark paid offline / "payment failed" email**. |
| **10am Fri** | For every client **paid** → mail **cleared for Correos**. Unpaid → mail **HELD + flagged** (red banner). |

**Hard gate:** mail is only sent to Correos once its charge is **succeeded** (or `paid_offline`).

## 6. Payment mechanics — reuse, don't reinvent
Model the mail charge on the existing **booking balance-charge system**:
- **`charge-balance`** → the off-session PaymentIntent pattern (saved card, connected account,
  double-charge guard, success→Factura, fail→persist status+error). New sibling: `charge-mail`.
- **`charge-due-balances`** → the daily cron + failed-retry pattern. New sibling:
  `charge-due-mail` (runs Thu 9pm + Fri mornings; idempotent; scans `vo_mail_charges` where status
  ∈ {pending, failed}).
- Admin actions reuse the booking pattern: **Retry** (`charge-mail` invokedBy:'manual'),
  **Send payment link** (`create-balance-payment-link` sibling), **Mark paid offline**
  (`mark-balance-paid-offline` sibling).
- **`send-invoice-email`** → the Factura on success; **`issue-credit-note`** → the refund.

## 7. Tier interaction
- **T1:** as above (balance or PAYG).
- **T2 / T3:** scheduled forwarding **frequency + postage included** — no per-cycle charge for the
  scheduled send; only an **ad-hoc / immediate** out-of-schedule forward triggers a €5 (the existing
  DispatchModal checkbox) + postage charge.

## 8. Refund on closure
A single admin **"Refund balance"** action: Stripe-refund the unused NET balance + auto-issue a
**Factura Rectificativa** (base + IVA) via `issue-credit-note`; zero the ledger.

## 9. Build phases (for the plan)
1. **Ledger + top-up:** `vo_mail_credit` (+ledger), collect the €10 seed at signup **with IVA**
   (adjust the Part-1 deposit line), portal top-up (€10/20/30) + balance display.
2. **Disposition + digest:** `disposition` on mail items (hold default), portal per-item UI,
   schedule + reword `send-vo-mail-digest`.
3. **Weekly charge:** `vo_mail_charges`, `charge-mail` + `charge-due-mail` (Thu 9pm / Fri retry),
   balance-draw-else-card, Factura, decline persistence.
4. **Admin + gate:** surface charge status + Retry/link/mark-paid/failed-email in the VO panel;
   the pay-before-Correos hold + flag.
5. **Refund action.**

## 10. Out of scope / open
- *anticipo* VAT reconciliation — **CONFIRMED by founder** (top-up carries IVA as a prepayment;
  unused-balance refund reverses base + IVA via Factura Rectificativa, reclaimed on Modelo 303).
  Not an open item.
- Auto-top-up (client-initiated only for now, per founder — no forced auto-charge).
- Correos label/API integration (physical send stays manual).

# Note for the Architect (Gemini) — Billing, Fees & Tax: current decisions

**Audience:** Gemini (architect AI) and any human/AI joining this work.
**Date:** 2026-06-19. **Status:** proof of concept / sandbox, one operator, test
Stripe keys only.

This note (a) records the entity facts, (b) corrects recurring factual errors
about Stripe and tax that have appeared in generated blueprints, and (c) specifies
the **agreed** implementation for the fixed per-tier platform fee. Read this before
proposing or pasting any payment/tax changes.

> ### Update 2026-06-23 (Flentix-native / de-TC) — read alongside the locked decisions
> - **Money-path naming changed.** The whole platform was de-TC'd (no TC Hub
>   legacy in code). Stripe metadata `brand` is now **`'FLENTIX'`** (was `'TCH'`);
>   metadata keys are prefixed **`fx_`** (were `tc_hub_`); statement descriptor
>   `'FLENTIX'`; reservation codes **`FLX-…`**. The webhook reads `brand:'FLENTIX'`.
>   The default workspace `0fc40c16` is now "Flentix Systems" (slug `flentix`).
>   The real TC Hub re-enters as a fresh onboarded hub (data) at migration — and
>   the cutover webhook needs a **temporary dual-accept** for TC Hub's old
>   `'TCH'`/`tc_hub_*`-tagged Stripe objects (GO_LIVE_CHECKLIST §A).
> - **⚠️ Per-hub Factura fiscal identity is now an OPEN fiscal task.** TC Hub's
>   hardcoded Spanish issuer identity (NIF `B19703941`, Almuñécar) was removed, so
>   customer Facturas currently issue with a **blank seller NIF/address**. The
>   customer Factura is issued by each **HUB's own Spanish legal entity** (per-hub)
>   — store each hub's legal name / NIF / registered address on its `workspaces`
>   row and pass it via `opts.seller` to `buildInvoicePdf` / `mintVoFactura`
>   (default `DEFAULT_SELLER` is intentionally blank). **Medacrii (UK) remains the
>   PLATFORM entity** that invoices the *operators* (the application-fee /
>   reverse-charge B2B invoices), NOT the customer-Factura seller. Blocker logged
>   in GO_LIVE_CHECKLIST §B.

---

## 0. LOCKED DECISIONS (confirmed by Grant, 2026-06-19)

These are settled. Do not reopen or contradict them in future blueprints.

1. **Billing model = Model 1 (end-users pay through Flentix).** Everything runs
   inside Flentix. The end-member pays via the Flentix checkout; the charge is a
   **direct charge** on the operator's **connected** Stripe account
   (`{ stripeAccount: connectAcct }`); the **operator is Merchant of Record**;
   Medacrii's cut is split out instantly as a Stripe **application fee**.
   ❌ The "wholesale-only — operators bill their own end-users in Nexudus / Cobot /
   their own ERP" model is **REJECTED**. Operators use *only* Flentix; no external
   billing system exists in the flow.

2. **Fixed-fee mechanism — the `application_fee_amount` rule (stop getting this
   wrong):** `application_fee_amount` is a **real field, but only for ONE-OFF
   payments**, never for subscriptions.

   | Charge type | Fixed-fee field | Where it goes |
   |---|---|---|
   | One-off (`mode: 'payment'`) | `application_fee_amount` ✅ | `payment_intent_data` |
   | **Subscription (`mode: 'subscription'`)** | **only `application_fee_percent`** | `subscription_data` |

   VO products are **subscriptions**, so a fixed €/tier fee is achieved as:
   **(a)** first invoice → `application_fee_percent` computed to *equal* the fixed
   fee; **(b)** every renewal → `application_fee_amount` set on the draft invoice
   via an `invoice.created` webhook. **NEVER put `application_fee_amount` in
   `subscription_data` — Stripe rejects it and checkout breaks.**

3. **Two-stage Phone checkout is REMOVED.** The old €104 "workspace domiciliation"
   line + separate €15/mo telecom subscription (on Medacrii's own account) was
   pre-Stripe-Connect cruft and should never have shipped. The **Phone tier is now
   a single €119 charge** on the operator's connected account, like every other
   tier, routed through normal KYC. There is **no** regulatory requirement for
   Medacrii to invoice the end-user — the operator is the retail biller (via the
   platform), and the telecom reseller VAT chain (Zadarma → Medacrii → operator →
   end-user) is satisfied by invoicing, not by a second charge.

4. ~~**Fee figures are PROVISIONAL.** €2 / €30 / €50 are placeholders.~~ **CONFIRMED
   2026-06-27 (fixed €/tier, ex-IVA): Tier 1 €5 / Tier 2 €20 / Tier 3 €60**, on list
   prices €35 / €65 / €129. The mechanism is unchanged (this §0.2 + §4 — `application_fee_percent`
   on invoice #1, `application_fee_amount` on renewals; one-off overage uses
   `application_fee_amount`). Detail in
   [SPEC_ai-voice-receptionist.md](SPEC_ai-voice-receptionist.md) §1.4. Re-seed the §4
   `FEE_CENTS` placeholders (`200/3000/5000`) to `500/2000/6000` when implementing.

---

## 1. Entities (do not conflate brand with legal entity)

- **Medacrii Associates Ltd** — UK limited company. The current legal /
  contracting / invoicing entity and the Stripe **platform** account holder.
- **Flentix (Systems)** — product/system brand only. Will become its own legal
  entity **after** PoC; **jurisdiction TBD**. Not a legal entity yet.
- **Operators ("hubs")** — Medacrii's clients (Spanish co-working / virtual-office
  businesses, VAT-registered). Each has its own Stripe **connected** account.
- **End-members** — the operators' customers.

Where existing docs/code treat "Flentix Systems" as the legal/billing company,
that is **incorrect** — the contracting entity is **Medacrii Associates Ltd**.

---

## 2. Canonical billing & tax model

1. **End-member → operator (connected account):** net price **+ 21% Spanish IVA**.
   IVA belongs to the operator to remit. Already implemented correctly
   (`create-virtual-office-checkout` adds an exclusive 21% ES tax rate to the line
   items, on the connected account).
2. **Medacrii's cut → from the operator:** a **fixed monthly fee per tier** (the
   PBX/software cost), taken as a Stripe Connect **application fee**. Charged on the
   **NET** amount, **never** on the IVA slice.
3. **Reverse charge** on Medacrii's fee is handled by **Medacrii issuing its own
   B2B invoices** to operators — **not** by Stripe. See `ACCOUNTANT_BRIEF.md`.
4. **Usage pass-through (future):** metered, billed at 100% cost, no margin.

---

## 3. Stripe facts — do not get these wrong again

These were asserted incorrectly in earlier generated blueprints. They are wrong:

1. **`application_fee_amount` is NOT a valid field inside Checkout
   `subscription_data`.** It is valid **only for one-off payments**
   (`payment_intent_data`). Checkout subscription mode accepts only
   `application_fee_percent` there; a **fixed** per-invoice fee must be set as
   `application_fee_amount` **on the invoice** (via an `invoice.created` webhook).
   See the locked rule and table in **§0.2**. (Pasting "swap application_fee_percent
   for application_fee_amount in subscription_data" produces code Stripe rejects.)
2. **`application_fee_percent` applies to the whole invoice total *including
   tax*** — not the net. To fee the net only with a percentage under a single
   uniform tax rate, divide by `(1 + rate)` (e.g. `÷1.21`); but the moment usage or
   mixed-rate items share the invoice, a percentage stops being "fixed/net-only".
3. **Stripe does NOT auto-generate reverse-charge / zero-rated VAT invoices** for
   the application fee based on the platform's address. Medacrii issues those.
4. **The first subscription invoice created via Checkout finalises immediately**,
   so an `invoice.created` webhook may be **too late** to set
   `application_fee_amount` on invoice #1. The first invoice must be handled
   separately (see §4).
5. **Direct-charge (connected-account) invoice events fire on the connected
   account.** The webhook endpoint must have **Connect events enabled** and every
   Stripe call acting on those objects must pass `{ stripeAccount: event.account }`.
6. **Write the DB money columns from the *actual finalised invoice*** (in
   `invoice.payment_succeeded`), never from a `× 0.21` guess at checkout-session
   creation time — at that moment no invoice/fee exists yet.

---

## 4. Agreed implementation — fixed per-tier application fee via invoice webhook

**Goal:** Medacrii takes a fixed €/tier fee from each operator subscription
invoice, isolated from IVA and from future usage lines.

> ⚠️ **All of this MUST be verified in Stripe *test mode* before live keys.** The
> first-invoice timing and the interaction between subscription-level
> `application_fee_percent` and invoice-level `application_fee_amount` are the two
> things most likely to behave differently than expected — confirm with real test
> transactions and the Stripe Dashboard, not by assumption.

### Prerequisites (Stripe Dashboard / config)
- The webhook endpoint must **listen to events on connected accounts** and be
  subscribed to **`invoice.created`** (keep `invoice.payment_succeeded`).
- Define the per-tier fee in **cents**, ideally in config/env or a DB column on
  `virtual_hubs`/the tier, **not** hardcoded. **PROVISIONAL placeholders** (Grant
  researching final split — see §0.4): `starter: 200`, `workspace: 3000`,
  `phone: 5000`.

### Change 1 — `create-virtual-office-checkout`
- **Remove** `application_fee_percent: PLATFORM_FEE_PERCENT`.
- Add the tier's fixed fee to `subscription_data.metadata` so the webhook can read
  it, e.g. `flentix_fee_cents: String(feeCents)`.
- **First-invoice handling:** because Checkout finalises invoice #1 immediately,
  set `subscription_data.application_fee_percent` to the value that *equals* the
  fixed fee for the first invoice total:
  `round((feeCents / firstGrossCents) * 100, 2)`, where
  `firstGrossCents = netUnitAmount * 1.21 (+ any add-ons)`. This makes invoice #1
  exactly the fixed fee. (Exact only while the first invoice has no usage/proration
  — true today.)

### Change 2 — `stripe-webhook` (new handler)
- Handle **`invoice.created`** for events where `event.account` is set:
  - Only act when `invoice.billing_reason === 'subscription_cycle'` (renewals) and
    the subscription metadata contains `flentix_fee_cents`.
  - Guard: only if `invoice.status === 'draft'` (cannot modify a finalised invoice).
  - `await stripe.invoices.update(invoice.id, { application_fee_amount: Number(fee) }, { stripeAccount: event.account })`.
- **Verify in test mode** whether the subscription-level `application_fee_percent`
  (set for invoice #1) bleeds into renewal invoices; if it does, either clear it
  after the first payment (`stripe.subscriptions.update(...)`) or rely on the
  invoice-level `application_fee_amount` taking precedence — confirm which.

### Change 3 — record actuals (in `invoice.payment_succeeded`)
Write `public.bookings` (the four new columns) from the **real** invoice:
- `base_amount` ← invoice subtotal excluding tax
- `tenant_tax_amount` ← invoice tax
- `gross_amount` ← invoice total / `amount_paid`
- `platform_fee_net` ← invoice `application_fee_amount`

### Out of scope / future
- **Usage pass-through.** Note a **cleaner alternative** exists for both the fixed
  fee and usage: a **separate Medacrii→operator subscription** on Medacrii's own
  account (fixed item + metered usage item), with **no** application fee on the
  end-member transaction. This is arguably more tax-clean (the reverse-charge
  invoice becomes a real Stripe invoice) and makes usage trivial. Revisit before
  building usage billing.

---

## 5. Paste-ready Lovable prompt (only when ready to implement §4, Changes 1–3)

```
Implement a FIXED per-tier Stripe Connect application fee, replacing the current
percentage fee. Do not invent Stripe fields — follow this exactly.

CONTEXT: subscriptions are created via Checkout as DIRECT CHARGES on the operator's
connected account. We want Medacrii to take a fixed €/tier fee from each invoice,
isolated from the 21% IVA. `application_fee_amount` is NOT valid in
subscription_data — only application_fee_percent is. Fixed amounts must be set on
the INVOICE via an invoice.created webhook.

1) supabase/functions/create-virtual-office-checkout/index.ts
   - Add a per-tier fee map in cents: FEE_CENTS = { starter: 200, workspace: 3000,
     phone: 5000 } (confirm values).
   - Remove `application_fee_percent: PLATFORM_FEE_PERCENT`.
   - Add `flentix_fee_cents: String(FEE_CENTS[tier])` to subscription_data.metadata.
   - For the FIRST invoice only (Checkout finalises it immediately), set
     subscription_data.application_fee_percent =
     Math.round((FEE_CENTS[tier] / firstGrossCents) * 100 * 100) / 100,
     where firstGrossCents = netUnitAmountCents * 1.21 (plus any add-on line totals).
   - Do not add transfer_data or on_behalf_of. Keep all existing tax_rates / Connect
     routing logic untouched.

2) supabase/functions/stripe-webhook/index.ts
   - Add an `invoice.created` handler. Only when event.account is set,
     invoice.billing_reason === 'subscription_cycle', invoice.status === 'draft',
     and the subscription metadata has flentix_fee_cents:
       await stripe.invoices.update(invoice.id,
         { application_fee_amount: Number(fee) },
         { stripeAccount: event.account });
   - In the existing invoice.payment_succeeded handler, populate bookings columns
     base_amount/tenant_tax_amount/gross_amount/platform_fee_net from the real
     invoice (subtotal excl tax / tax / total / application_fee_amount).

3) Do NOT claim Stripe handles reverse-charge VAT — it does not. No tax-invoice
   logic for the application fee here.

After implementing, STOP and let us test in Stripe test mode before anything else.
```

### Test-mode checklist (run before live)
- [ ] Tier 2 €100 net → end-member charged €121; operator balance receives €121.
- [ ] First invoice application fee = exactly the fixed tier fee (€30).
- [ ] A simulated renewal (`subscription_cycle`) invoice gets the fixed
      `application_fee_amount` via the webhook, and is **not** double-charged a
      percentage fee.
- [ ] `bookings` row shows base 100.00 / tax 21.00 / gross 121.00 / fee 30.00 from
      the real invoice.
- [ ] Webhook receives **connected-account** `invoice.created` events (Connect
      events enabled).

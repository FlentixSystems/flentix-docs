# Flentix — Go-Live Checklist (Stripe + System)

**Status:** living document — add to it as we go. **Owner:** Grant.
**Context:** Sandbox/test today. Target live: ~this week. Nothing here is "done" until ticked.
See [NOTE_FOR_ARCHITECT.md](NOTE_FOR_ARCHITECT.md) (billing/tax facts) and
[PLAN_country-rollout-and-operator-pricing.md](PLAN_country-rollout-and-operator-pricing.md).

---

## A. Stripe Connect & payments (the money model)

- [ ] **🔴 NEEDED NOW (blocks the whole VO flow) — register a Connect webhook endpoint.** Because
  VO checkout is a direct charge on the connected account, `checkout.session.completed` is a
  **Connect event** delivered only to a "Events on Connected accounts" endpoint. Without it, the
  webhook never fires → no subscriber/email/admin row (confirmed in the first test). Steps:
  Stripe → Developers → Webhooks → **Add endpoint** → choose **"Events on Connected accounts"** →
  URL = the same `stripe-webhook` function URL as the platform endpoint
  (`https://wdbxhexqilgqxvkpbsqy.supabase.co/functions/v1/stripe-webhook` — confirm against the
  existing endpoint) → events: `checkout.session.completed` (+ `invoice.payment_succeeded`,
  `invoice.created` for renewals/fees) → copy its signing secret → set project secret
  **`STRIPE_CONNECT_WEBHOOK_SECRET`**. Code already handles both secrets + connected-account context.

- [x] **Account hierarchy verified (2026-06-21):** Platform/Master = **`acct_1TeFkSLUNJ0dTYxa`**;
  connected tenant (Spacio-Lab test hub) = **`acct_1TjSGaLUnJuGzBr8`**. The hub's
  `stripe_connect_id` correctly = the **connected** account, so the test setup is CORRECT — the
  full charge landing on `acct_1TjSGaLUnJuGzBr8` is expected (it's the operator/Merchant of Record).
  (Earlier "hub points at the platform's own account" worry was a misread before the platform id
  was known.)
- [ ] **Confirm the fee on the PLATFORM account:** in `acct_1TeFkSLUNJ0dTYxa` look under
  **Connect → Application fees** (and **Balance → Transactions**, type *application fee*) — NOT the
  *Payments* list. Each test subscription's first invoice should show the fixed per-tier fee. Verify
  the fixed fee equals the intended €/tier against the gross.
- [ ] **Every real operator gets their OWN connected account** (onboarded via Connect); a hub's
  `stripe_connect_id` must never be the platform id `acct_1TeFkSLUNJ0dTYxa`.
- [ ] **NEVER** use `application_fee_amount` in `subscription_data` (Stripe rejects it → breaks
  checkout). First invoice uses `application_fee_percent`; renewals set `application_fee_amount` on
  the draft invoice via the `invoice.created` Connect webhook. (Flagged because a blueprint
  re-proposed the wrong form on 2026-06-21.)
- [ ] **Swap test keys → live keys:** `STRIPE_SECRET_KEY`, `STRIPE_PUBLISHABLE_KEY` /
  `VITE_SUPABASE_PUBLISHABLE_KEY` usage, and the **live webhook signing secret**.
- [ ] **Finalise the platform fee** — replace the PROVISIONAL `FEE_CENTS` (starter €2 /
  growth €30 / premium €50) and confirm the **annual policy** (currently `fee × 12`). Update in
  `create-virtual-office-checkout`.
- [ ] **Verify both fee paths fire:** first subscription invoice uses `application_fee_percent`
  (computed to equal the fixed fee vs gross); renewals set `application_fee_amount` on the draft
  invoice via the `invoice.created` Connect webhook (`{ stripeAccount: event.account }`). Never
  put `application_fee_amount` in `subscription_data`.
- [ ] **VAT on live:** confirm the live `STRIPE_VAT_RATE_ES_21` tax rate exists, and that
  `getOrCreateSpanishVatRate` creates/uses the rate **on each connected account** (tax rates are
  per-account on Connect).
- [ ] **Medacrii reverse-charge invoices:** Stripe does NOT auto-issue these. Medacrii must
  issue its own B2B UK→ES reverse-charge invoices (no IVA) to operators for the platform fee.
- [ ] **Connect onboarding live:** operators complete Stripe hosted onboarding (`account_onboarding`)
  with payouts enabled; platform profile/branding set for hosted invoices & onboarding.
- [ ] **Annual billing live-tested:** monthly AND annual each tested end-to-end (correct €, 21%
  IVA, `recurring.interval` month/year, correct fee).

## B. System / data

- [ ] **Persist `hub_id` on `virtual_office_subscribers`.** Checkout receives `hub_id` but the
  webhook does not store it, so KYC/compliance country resolution defaults to `'ES'`. Persist it
  (column or `engine_metadata`), then in `resolve-kyc-token` + `submit-kyc-documents` swap the
  `const countryId = 'ES'` line for a `virtual_hubs.country_code` lookup. Required before a 2nd country.
- [ ] **Confirm `customer_services.legacy_order_id` → `virtual_office_subscribers.id`** maps
  correctly on a real signup (the compliance-row write depends on it; currently fails safe/silent).
- [ ] **Storefront country dropdown** (when multi-country): must filter `countries_config.is_enabled = true`.
- [ ] **Per-country VAT rates + registration** — accountant to confirm each country's rate and
  *who is VAT-registered where* before enabling it.
- [ ] **Re-skin to canonical brand** (Emerald #10B981 + Charcoal #1F2937, Montserrat/Open Sans) —
  sandbox currently uses the TC Hub sunset/orange skin.
- [ ] **Tier display names** — decide final customer-facing names (current marketing names
  "Workspace Hybrid" / "AI Executive Suite" vs the tier labels Starter/Growth/Premium) and make
  them consistent across storefront, Stripe descriptions, portal and admin.

## B2. Virtual Office journey — config to switch on (needed for the flow to work end-to-end)

The VO journey is wired in code (2026-06-21): order-ack email (`vo_order_ack`), KYC link email,
KYC-received email (`vo_kyc_received`), passwordless `/portal/login`, subscriber↔customer link,
mailroom pending state, day-credit→bono→TTLock. These config items must be ON for it to run:

- [ ] **Supabase Auth — Redirect URLs:** add the preview/live origin + `/portal/mailroom` to
  Auth → URL Configuration (Site URL + Redirect URLs), or the magic-link login will reject the
  redirect.
- [ ] **Supabase Auth — email/SMTP:** the magic-link email is sent by Supabase Auth (not our
  Brevo functions). Configure Auth SMTP (ideally Brevo) + the email template for production; in
  test the default sender works but is rate-limited.
- [ ] **`BREVO_DIRECT_API_KEY` secret** must be set (the `vo_kyc_received` email uses it; it
  skips silently if missing).
- [ ] **`FRONTEND_URL` secret** must point at the real app domain (the activation email's
  "Access your portal" button uses it; defaults to `https://flentixsystems.com`).
- [ ] **Automated KYC reminder** (optional) — currently only the manual admin "Resend KYC" exists.
  A scheduled function could nudge `kyc_submitted = false` subscribers after N days (cap ~2).
- [ ] **Email re-brand:** the new VO emails use the current TC Hub orange; re-skin to emerald
  with the app.

## C. Deferred features (do before selling them)

- [ ] **Outbound-telesales add-on fee routing** onto the Connect rails (flag column already exists).
- [ ] **Tier-3 (Premium) geographic number provisioning** + strict hub↔zone validation
  (`virtual_hubs.phone_prefix` vs `prefix_registry.geographic_prefix`).
- [ ] **Tier-2 (Growth) optional geographic number** for customers who already independently
  qualify with their own in-zone address (eligibility check + opt-in; schema already supports it).
- [ ] **Asset (desk/office) operator pricing** — reuse the recommend+override pattern.

# Flentix — Go-Live Checklist (Stripe + System)

**Status:** living document — add to it as we go. **Owner:** Grant.
**Context:** Sandbox/test today. Target live: ~this week. Nothing here is "done" until ticked.
See [NOTE_FOR_ARCHITECT.md](NOTE_FOR_ARCHITECT.md) (billing/tax facts) and
[PLAN_country-rollout-and-operator-pricing.md](PLAN_country-rollout-and-operator-pricing.md).

---

## A. Stripe Connect & payments (the money model)

- [ ] **CRITICAL — every operator must have their OWN Stripe connected account.** In the
  current sandbox the test hub `XY Labs - Granada Coast` has `stripe_connect_id =
  acct_1TjSGaLUnJuGzBr8`, which is **Flentix's own platform account**. That is why a test
  charge shows the **full amount** in the Flentix account with **no fee split** — you cannot
  take an application fee from yourself. Before go-live, each hub's `stripe_connect_id` must be
  a **distinct** connected account belonging to that operator, onboarded via Stripe Connect.
  Platform account ID must NEVER appear as a hub's `stripe_connect_id`.
- [ ] **Verify the split on a real connected account:** after onboarding a separate operator
  test account and pointing the hub at it, a checkout should show: **full charge in the
  OPERATOR's** connected account (operator = Merchant of Record), and only the **application
  fee** in the **Flentix/Medacrii platform** account.
- [ ] **Where the fee is visible (educate / confirm):** the application fee does NOT appear in
  the platform's main *Payments* list. It appears in the platform account under **Balance →
  Transactions** (type: *application fee*) and **Connect → Application fees**. The *Payments*
  list shows charges the account is Merchant of Record for.
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

## C. Deferred features (do before selling them)

- [ ] **Outbound-telesales add-on fee routing** onto the Connect rails (flag column already exists).
- [ ] **Tier-3 (Premium) geographic number provisioning** + strict hub↔zone validation
  (`virtual_hubs.phone_prefix` vs `prefix_registry.geographic_prefix`).
- [ ] **Tier-2 (Growth) optional geographic number** for customers who already independently
  qualify with their own in-zone address (eligibility check + opt-in; schema already supports it).
- [ ] **Asset (desk/office) operator pricing** — reuse the recommend+override pattern.

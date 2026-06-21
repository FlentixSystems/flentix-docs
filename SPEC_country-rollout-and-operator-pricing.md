# Spec — Virtual Office: Country Rollout + Operator-Adjustable Pricing

**Audience:** Gemini (architect), Lovable build, and any human/AI joining this work.
**Date:** 2026-06-21. **Owner:** Grant (Founder). **Status:** Decided — ready to turn into a Lovable build prompt.
**Scope of this round:** Virtual offices only. Assets (desks/offices) reuse the same pricing pattern in a later round — designed for, not built here.

This spec corrects and extends the country-rollout blueprint proposed by Gemini. It
should be read alongside [NOTE_FOR_ARCHITECT.md](NOTE_FOR_ARCHITECT.md) (billing/fees/tax
locked decisions) and the European Expansion Strategy doc. Where this spec and a prior
blueprint disagree, this spec wins.

---

## 0. Why Gemini's blueprint could not be pasted as-is

The instinct was right (a country must be **configuration, not code**), but the blueprint
was written against tables that do not exist in the live repo, and it omitted the things
that actually matter for our confirmed billing model. Verified against the repo on 2026-06-21:

1. **No `virtual_office_tiers` table exists.** Tiers (`starter`, `mail`, `workspace`,
   `phone`) and their prices are **hardcoded in the checkout edge function**
   (`create-virtual-office-checkout/index.ts` — `unit_amount: 3500/5900/8900/11900` and
   fixed Stripe price IDs). Gemini's `alter table virtual_office_tiers …` would fail.
2. **No generic `orders` table exists.** A virtual office is a **subscription** →
   `virtual_office_subscribers`. Bookings (`bookings`) are the desk/room side. Compliance
   must attach to the subscriber, not an invented "order".
3. **Tax was omitted.** `countries_config` had no VAT rate. Switching on a new country
   without one means every invoice charges Spain's 21% — wrong tax, real liability.
4. **Add-on charging was unspecified.** The outbound-telesales fee must ride the existing
   Connect rails (see NOTE_FOR_ARCHITECT §0), not a side charge on Medacrii.
5. **Numbering already has a home.** `prefix_registry` (shipped 2026-06-17) is the source
   of truth for numbers; assigned-number fields must reference it, not be loose text.

**What Gemini got right and is kept:** config-driven country model; outbound telesales as an
add-on flag (not a 4th tier); dynamic KYC UI from a per-country document list; Tier-3 hub↔zone
validation; `has_geographic_moat = false` for France.

---

## 1. Locked decisions for this round (confirmed by Grant)

1. **Canonical tiers = 3:** `starter` (Tier 1), `growth` (Tier 2), `premium` (Tier 3),
   per the strategy doc. The legacy 4-key set (`starter`/`mail`/`workspace`/`phone`) is
   **removed completely**; any existing test rows are remapped to the 3-tier keys (PoC test
   data only — not a real migration).
2. **Operators set their own prices** — per tier, including in Spain at launch. We
   **heavily recommend** a market price and pre-fill it, but the operator can override.
3. **Recommend, don't dictate.** Recommended price = our number per country × tier × interval.
   Operator price = their chosen number; defaults to the recommendation until they change it.
4. **Monthly and annual billing.** Each tier carries a monthly price and an **optional**
   annual price (the operator's discounted figure). Blank annual = that operator does not
   offer annual for that tier. Both are operator-adjustable.
5. **Pricing is the foundation; country compliance bolts on top.** Prices move out of code
   into tables *first*, so adding a country and letting operators price are the same
   underlying capability — no rebuild.
6. **VAT rate is country law, NOT an operator setting.** The rate is set **once per country
   by us** (with the accountant) in `countries_config`; every operator in that country
   inherits it. An operator never confirms or edits the rate. The only tax item an operator
   supplies is **their own VAT registration number** (captured at onboarding, printed on
   their invoices as Merchant of Record). Actual rates + registration positions are
   **accountant territory** — this spec provides the columns, not the numbers.

---

## 2. Data model

> **Tier keys throughout:** `starter` (Tier 1), `growth` (Tier 2), `premium` (Tier 3).
> **Interval keys:** `monthly`, `annual`. Net amounts are minor units (cents), NET of VAT.

### 2.1 `tier_price_recommendations` — our "heavily recommended" price
One row per country × tier. Monthly required, annual optional. This is the market-intelligence
number the UI pre-fills.

```sql
create table public.tier_price_recommendations (
  id uuid primary key default gen_random_uuid(),
  country_id text not null references public.countries_config(id),
  tier text not null check (tier in ('starter','growth','premium')),
  recommended_monthly_net integer not null,  -- cents, NET of VAT
  recommended_annual_net integer,            -- cents, NET of VAT; null = no annual recommendation
  currency text not null default 'EUR',
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (country_id, tier)
);
```

### 2.2 `virtual_office_plans` — the operator's chosen price (the override)
**The operator is identified by `hub_id` → `virtual_hubs`** — that is already how the live
checkout routes the direct charge (`virtual_hubs.stripe_connect_id`), and a hub carries its
own `country_code`. So plans key on **hub × tier**; the country is derived from the hub, not
stored again. Monthly required; annual optional (blank = that hub does not offer annual for
that tier). If no row exists for a tier, the checkout falls back to the recommendation in 2.1
(looked up by the hub's `country_code`).

```sql
create table public.virtual_office_plans (
  id uuid primary key default gen_random_uuid(),
  hub_id uuid not null references public.virtual_hubs(id),  -- the operator/location (carries stripe_connect_id + country_code)
  tier text not null check (tier in ('starter','growth','premium')),
  monthly_net integer not null,             -- cents, NET of VAT — operator's chosen monthly price
  annual_net integer,                       -- cents, NET of VAT — operator's chosen annual price; null = not offered
  currency text not null default 'EUR',
  is_active boolean not null default true,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (hub_id, tier)
);
```

> **Checkout change:** `create-virtual-office-checkout` already receives `hub_id` and looks up
> `virtual_hubs` for `stripe_connect_id`; extend that same lookup to also read `country_code`.
> It stops reading `CONNECT_TIER_PRICE_DATA` hardcoded amounts. The customer picks tier +
> interval (monthly/annual); the function resolves the net price as:
> `virtual_office_plans.{monthly_net|annual_net}` for that `hub_id` → else the matching
> `tier_price_recommendations` value for the hub's `country_code`. It creates/uses the Stripe Price for the chosen
> interval, then adds VAT per §2.3 (still exclusive/added at line-item time, as today). The
> fixed application fee and direct-charge-on-connected-account logic is unchanged
> (NOTE_FOR_ARCHITECT §0) — note the fixed per-period fee math differs for annual vs monthly
> invoices and must be set accordingly.

### 2.3 `countries_config` — master switchboard (Gemini's, corrected)
A country becomes selectable only once its row exists and `is_enabled = true`.

```sql
create table public.countries_config (
  id text primary key,                      -- 'ES','DE','NL','PT','IT', …
  country_name text not null,
  default_currency text not null default 'EUR',
  vat_rate numeric(5,4) not null,           -- e.g. 0.2100 ES, 0.1900 DE, 0.2300 PT  [ACCOUNTANT to confirm]
  vat_label text not null,                  -- 'IVA','MwSt','IVA',…  [ACCOUNTANT to confirm]
  tier2_prefix_zone text,                   -- references prefix_registry zone for national/Tier-2 range
  kyc_weight text not null check (kyc_weight in ('light','medium','heavy')),
  required_kyc_docs text[] not null default '{}',  -- drives the dynamic upload UI (Rule 2)
  has_geographic_moat boolean not null default true,  -- false for France
  is_enabled boolean not null default false,
  created_at timestamptz not null default now()
);
```

### 2.4 Compliance — attached to the subscriber (not a phantom "order")
Either extend `virtual_office_subscribers` or add a 1:1 companion keyed by `subscriber_id`.
Presented as a companion table for clarity:

```sql
create table public.virtual_office_subscriber_compliance (
  id uuid primary key default gen_random_uuid(),
  subscriber_id uuid not null references public.virtual_office_subscribers(id),
  country_id text not null references public.countries_config(id),
  kyc_status text not null default 'pending' check (kyc_status in ('pending','submitted','verified','rejected')),
  uploaded_docs_manifest jsonb not null default '{}',
  outbound_telesales_flag boolean not null default false,  -- Spain 400-prefix add-on (Rule 1)
  assigned_geographic_number_id uuid references public.prefix_registry(id),  -- Tier 3
  assigned_national_number_id uuid references public.prefix_registry(id),    -- Tier 2
  verified_at timestamptz,
  created_at timestamptz not null default now()
);
```
> `prefix_registry` FK column names to be confirmed against its actual PK during build.

### 2.5 Admin "Country Setup" flow — the confirmation gate before a market opens
Adding a country is an **admin-only setup step** that *prompts the operator-platform admin
(us) to enter and confirm* the country's compliance configuration before it can go live.
It writes one `countries_config` row. The form prompts for:

- **VAT rate + VAT label** (e.g. 21% / "IVA") — entered and confirmed here, by us.
- **KYC weight + required KYC document list** — drives the dynamic upload UI (Rule 2).
- **Numbering** (Tier-2 national range / Tier-3 geographic zone via `prefix_registry`).
- **Geographic-moat flag** (off for France).
- Recommended launch prices (seeds `tier_price_recommendations`).

**Hard gate:** `is_enabled` cannot be set `true` — and the country cannot appear in the
customer sign-up dropdown — until **VAT rate, VAT label, and at least one required KYC
document** are present. Enforce at the application layer and ideally with a DB constraint /
trigger. This is the "confirm before we move into the market" checkpoint: a half-configured
country can never leak live.

---

## 3. The three system rules (kept from Gemini, with the fixes)

- **Rule 1 — Outbound add-on is a flag, charged on the existing rails.** Store
  `outbound_telesales_flag` on the compliance row. The *fee* is added through the same
  Connect direct-charge + fixed application-fee mechanism as everything else — **never** a
  separate charge on Medacrii, and **never** `application_fee_amount` in `subscription_data`
  (NOTE_FOR_ARCHITECT §0). Not a 4th tier.
- **Rule 2 — Dynamic KYC UI.** On country select, the frontend renders upload slots from
  `countries_config.required_kyc_docs`. No hardcoded per-country forms.
- **Rule 3 — Tier-3 strict hub↔zone validation.** For Tier 3 the chosen `hub_id` must match
  the geographic zone of the number being provisioned (via `prefix_registry`). Tiers 1–2 may
  use a national/generic zone.

---

## 4. Out of scope this round (designed for, not built)

- **Asset (desk/office) operator pricing** — same recommend+override pattern, separate table,
  later round.
- **Actual VAT rates and registration positions** — accountant brief.
- **Belgium / Ireland / Poland** country specifics — verify before switch-on (strategy doc §7).

---

## 5. Open items for the build prompt

1. ~~Confirm operator source of truth.~~ **RESOLVED:** operator = `virtual_hubs` row; key is
   `hub_id`; connected account = `virtual_hubs.stripe_connect_id`; country = `virtual_hubs.country_code`.
2. ~~Confirm `prefix_registry` PK.~~ **RESOLVED:** `prefix_registry.id` (uuid). Note it keys on
   `(country_code, postal_prefix)` with `geographic_prefix` — zone match for Tier 3 is on
   `geographic_prefix` vs the hub's `phone_prefix`.
3. **Remove legacy 4-key tiers** (`mail`/`workspace`/`phone` and any `starter` price-map
   entries) from `create-virtual-office-checkout`; remap any existing test subscriber/booking
   rows to `starter`/`growth`/`premium`.
4. Seed Spain `tier_price_recommendations` with the confirmed launch numbers (all ex-VAT):
   - **Monthly (no-commit):** Starter €29 / Growth €69 / Premium €119 → `recommended_monthly_net`.
   - **Annual (per-month rate, billed yearly):** Starter €24 / Growth €59 / Premium €99.
     Store `recommended_annual_net` as the **full yearly charge** (e.g. €24 → 28800 cents),
     display as "€24/mo billed annually". Stripe charges the yearly total once.
   - **Do not** seed from the legacy 4 hardcoded amounts.
5. **Launch offer (€19 / €49 / €89) is a time-limited promotion, NOT a base price.** Model it
   via the existing `promo_codes` table (or a dated promo override) with a start/end date so it
   expires on its own and the base monthly/annual numbers stay clean. Do not write it into
   `tier_price_recommendations`.
6. Confirm the per-period fixed application-fee math for **annual** invoices (the
   `application_fee_percent`-on-first-invoice trick from NOTE_FOR_ARCHITECT §0 must be
   recomputed against the annual gross, not the monthly gross).

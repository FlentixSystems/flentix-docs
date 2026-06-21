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

1. **Operators set their own prices** — per tier, including in Spain at launch. We
   **heavily recommend** a market price and pre-fill it, but the operator can override.
2. **Recommend, don't dictate.** Recommended price = our number per country × tier.
   Operator price = their chosen number; defaults to the recommendation until they change it.
3. **Pricing is the foundation; country compliance bolts on top.** Prices move out of code
   into tables *first*, so adding a country and letting operators price are the same
   underlying capability — no rebuild.
4. **VAT rate/label is per-country configuration.** The actual rates, and who is
   VAT-registered in each country, are **accountant territory** — this spec provides the
   columns; it does not invent the numbers.

---

## 2. Data model

### 2.1 `tier_price_recommendations` — our "heavily recommended" price
One row per country × tier. This is the market-intelligence number the UI pre-fills.

```sql
create table public.tier_price_recommendations (
  id uuid primary key default gen_random_uuid(),
  country_id text not null references public.countries_config(id),
  tier text not null,                       -- 'starter' | 'mail' | 'workspace' | 'phone' (existing keys)
  recommended_net_amount integer not null,  -- minor units (cents), NET of VAT
  currency text not null default 'EUR',
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (country_id, tier)
);
```

### 2.2 `virtual_office_plans` — the operator's chosen price (the override)
One row per operator × country × tier. Replaces the hardcoded price map in the checkout
function. If no row exists for a tier, the checkout falls back to the recommendation in 2.1.

```sql
create table public.virtual_office_plans (
  id uuid primary key default gen_random_uuid(),
  operator_id uuid not null,                -- the connected operator (organization/workspace owner)
  country_id text not null references public.countries_config(id),
  tier text not null,
  net_amount integer not null,              -- minor units (cents), NET of VAT — operator's chosen price
  currency text not null default 'EUR',
  is_active boolean not null default true,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (operator_id, country_id, tier)
);
```

> **Checkout change:** `create-virtual-office-checkout` stops reading `CONNECT_TIER_PRICE_DATA`
> hardcoded amounts and instead resolves price as: operator's `virtual_office_plans.net_amount`
> → else `tier_price_recommendations.recommended_net_amount`. VAT is then added on top per
> §2.3 (still exclusive/added at line-item time, as today). The fixed application fee and
> direct-charge-on-connected-account logic is unchanged (NOTE_FOR_ARCHITECT §0).

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

1. Confirm `operator_id` source of truth (organization vs workspace vs connected-account id)
   against the live schema before writing the FK.
2. Confirm `prefix_registry` primary key column name for the two number FKs.
3. Backfill: seed `tier_price_recommendations` for Spain from current hardcoded amounts
   (3500/5900/8900/11900) so behaviour is unchanged on day one.
4. **Tier vocabulary mismatch to resolve:** the strategy doc uses a 3-tier model
   (Starter / Growth / Premium), but the live checkout uses 4 keys
   (`starter`, `mail`, `workspace`, `phone`). Confirm the canonical tier set before
   seeding `tier_price_recommendations`, so the recommended-price rows line up 1:1 with
   what checkout actually requests.

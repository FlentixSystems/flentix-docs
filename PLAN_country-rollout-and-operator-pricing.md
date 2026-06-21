# VO Country Rollout + Operator Pricing — Implementation Plan

> **For the builder (Lovable):** Execute task-by-task, in order. Each task is one Supabase
> migration and/or one focused code change with a verification step. Do not start a task until
> the previous one verifies. Each task's paste-ready Lovable instruction is in its **Prompt** block.

**Goal:** Move virtual-office pricing out of hardcoded code into operator-adjustable, per-country
tables (monthly + annual), and make adding a country a gated configuration task — without
breaking the live Spanish checkout.

**Architecture:** Three pricing/config tables (`countries_config`, `tier_price_recommendations`,
`virtual_office_plans`) + one compliance table, all keyed to the existing `virtual_hubs`
(operator) and `prefix_registry` (numbers). The `create-virtual-office-checkout` edge function
stops using its hardcoded `CONNECT_TIER_PRICE_DATA` map and resolves price from the tables, keyed
by the `hub_id` it already receives. Stripe Connect direct-charge + application-fee wiring is
untouched.

**Tech Stack:** Supabase (Postgres + RLS), Deno edge functions, Stripe Connect, Vite/React/shadcn
frontend, built via Lovable.

## Global Constraints

- **Canonical tiers = `starter`, `growth`, `premium`** (3 only). Legacy `mail`/`workspace`/`phone`
  removed. All tier columns: `check (tier in ('starter','growth','premium'))`.
- **Net amounts are integers in cents, ex-VAT.** Currency default `EUR`.
- **Operator = `hub_id` → `virtual_hubs`.** Connected account = `virtual_hubs.stripe_connect_id`;
  country = `virtual_hubs.country_code`. Never invent an `operator_id`.
- **VAT rate is per-country config in `countries_config`, set by us — never an operator field.**
- **Billing rails unchanged** (see NOTE_FOR_ARCHITECT §0): subscription-mode Checkout, direct
  charge on the connected account, fixed fee via `application_fee_percent` on first invoice +
  `application_fee_amount` on later invoices. **Never** put `application_fee_amount` in
  `subscription_data`.
- **A country cannot be `is_enabled = true` unless `vat_rate`, `vat_label`, and a non-empty
  `required_kyc_docs` are present.**

---

### Task 1: `countries_config` table + seed Spain

**Files:** New Supabase migration.
**Interfaces — Produces:** table `public.countries_config(id text pk, …)`, with row `ES` enabled.
Every later table FKs to `countries_config(id)`.

- [ ] **Step 1: Create the table and seed Spain.**

**Prompt (to Lovable):**
> Create a Supabase migration that adds table `public.countries_config` exactly as below, enables
> RLS with a public-readable SELECT policy and service_role ALL policy, and seeds the Spain row.

```sql
create table public.countries_config (
  id text primary key,
  country_name text not null,
  default_currency text not null default 'EUR',
  vat_rate numeric(5,4) not null,
  vat_label text not null,
  tier2_prefix_zone text,
  kyc_weight text not null check (kyc_weight in ('light','medium','heavy')),
  required_kyc_docs text[] not null default '{}',
  has_geographic_moat boolean not null default true,
  is_enabled boolean not null default false,
  created_at timestamptz not null default now()
);
alter table public.countries_config enable row level security;
create policy "countries_config public read" on public.countries_config for select using (true);
create policy "countries_config service all" on public.countries_config for all to service_role using (true) with check (true);

insert into public.countries_config
  (id, country_name, default_currency, vat_rate, vat_label, tier2_prefix_zone, kyc_weight, required_kyc_docs, has_geographic_moat, is_enabled)
values
  ('ES','Spain','EUR',0.2100,'IVA','51x','heavy',
   '{passport,company_registry,proof_of_address}', true, true);
```

- [ ] **Step 2: Verify.** Run in Supabase SQL: `select id, vat_rate, vat_label, is_enabled from public.countries_config;`
  Expected: one row `ES | 0.2100 | IVA | true`.

---

### Task 2: `tier_price_recommendations` table + seed Spain launch prices

**Files:** New migration.
**Interfaces — Consumes:** `countries_config(id)`. **Produces:** table with 3 Spain rows used as
the checkout fallback price and the operator-UI prefill.

- [ ] **Step 1: Create and seed.**

**Prompt (to Lovable):**
> Create a Supabase migration adding `public.tier_price_recommendations` as below (RLS:
> public-read, service_role all), and seed the three Spain rows. Amounts are cents, ex-VAT;
> `recommended_annual_net` is the full yearly charge.

```sql
create table public.tier_price_recommendations (
  id uuid primary key default gen_random_uuid(),
  country_id text not null references public.countries_config(id),
  tier text not null check (tier in ('starter','growth','premium')),
  recommended_monthly_net integer not null,
  recommended_annual_net integer,
  currency text not null default 'EUR',
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (country_id, tier)
);
alter table public.tier_price_recommendations enable row level security;
create policy "tpr public read" on public.tier_price_recommendations for select using (true);
create policy "tpr service all" on public.tier_price_recommendations for all to service_role using (true) with check (true);

-- Spain launch: monthly 29/69/119; annual 24/59/99 per-month => yearly total 288/708/1188
insert into public.tier_price_recommendations (country_id, tier, recommended_monthly_net, recommended_annual_net) values
  ('ES','starter', 2900,  28800),
  ('ES','growth',  6900,  70800),
  ('ES','premium', 11900, 118800);
```

- [ ] **Step 2: Verify.** `select tier, recommended_monthly_net, recommended_annual_net from public.tier_price_recommendations where country_id='ES' order by tier;`
  Expected: `growth 6900 70800`, `premium 11900 118800`, `starter 2900 28800`.

---

### Task 3: `virtual_office_plans` table (operator override, hub-keyed)

**Files:** New migration.
**Interfaces — Consumes:** `virtual_hubs(id)`. **Produces:** table `virtual_office_plans`; absence
of a row for a (hub, tier) means "use the recommendation."

- [ ] **Step 1: Create the table.** Leave it empty — TC Hub inherits recommendations until it
  sets overrides (exercises the fallback path).

**Prompt (to Lovable):**
> Create a Supabase migration adding `public.virtual_office_plans` as below. RLS: a hub owner may
> read/write only rows for hubs they own (match your existing hub-ownership pattern — reuse the
> same policy shape used elsewhere for `virtual_hubs`); service_role all. Do not seed.

```sql
create table public.virtual_office_plans (
  id uuid primary key default gen_random_uuid(),
  hub_id uuid not null references public.virtual_hubs(id) on delete cascade,
  tier text not null check (tier in ('starter','growth','premium')),
  monthly_net integer not null,
  annual_net integer,
  currency text not null default 'EUR',
  is_active boolean not null default true,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (hub_id, tier)
);
alter table public.virtual_office_plans enable row level security;
create policy "vop service all" on public.virtual_office_plans for all to service_role using (true) with check (true);
-- Hub-owner read/write policy: mirror the existing ownership rule used for virtual_hubs.
```

- [ ] **Step 2: Verify.** `select count(*) from public.virtual_office_plans;` Expected: `0`, and the
  table exists with the FK to `virtual_hubs`.

---

### Task 4: Switch checkout to table-driven pricing + remove legacy tiers

**Files:** Modify `supabase/functions/create-virtual-office-checkout/index.ts`.
**Interfaces — Consumes:** `virtual_hubs.country_code` (extend the existing hub lookup),
`virtual_office_plans`, `tier_price_recommendations`. This is the highest-risk task — it touches
live payments. Change pricing source only; leave Connect/VAT/fee wiring exactly as is.

- [ ] **Step 1: Extend the existing hub lookup to also fetch `country_code`.** The function already
  does `.from('virtual_hubs').select('stripe_connect_id, is_active').eq('id', hub_id)` — add
  `country_code` to that select. No new query.

- [ ] **Step 2: Accept `interval` from the request** (`'monthly' | 'annual'`, default `'monthly'`)
  and validate `tier ∈ {starter, growth, premium}`. Reject any legacy tier value.

- [ ] **Step 3: Replace the hardcoded `CONNECT_TIER_PRICE_DATA` map with a resolver.** Delete the
  `{ starter, mail, workspace, phone }` map and the `3500/5900/8900/11900` literals. Insert:

```ts
// net price (cents, ex-VAT) for this hub/tier/interval
async function resolveNetAmount(
  supabaseAdmin: SupabaseClient, hubId: string, countryCode: string,
  tier: 'starter'|'growth'|'premium', interval: 'monthly'|'annual'
): Promise<number> {
  const { data: plan } = await supabaseAdmin
    .from('virtual_office_plans')
    .select('monthly_net, annual_net')
    .eq('hub_id', hubId).eq('tier', tier).eq('is_active', true).maybeSingle();
  if (plan) {
    const v = interval === 'annual' ? plan.annual_net : plan.monthly_net;
    if (typeof v === 'number') return v;            // null annual => fall through to recommendation
  }
  const { data: rec } = await supabaseAdmin
    .from('tier_price_recommendations')
    .select('recommended_monthly_net, recommended_annual_net')
    .eq('country_id', countryCode).eq('tier', tier).single();
  const v = interval === 'annual' ? rec.recommended_annual_net : rec.recommended_monthly_net;
  if (typeof v !== 'number') throw new Error(`No ${interval} price for ${tier}/${countryCode}`);
  return v;
}
```

- [ ] **Step 4: Use the resolved amount and the right recurring interval in the existing
  `price_data` block.** Set `unit_amount` to the resolved net amount and
  `recurring.interval` to `'month'` for monthly or `'year'` for annual. Keep the existing 21% IVA
  application and the existing Connect/`application_fee_percent` logic unchanged. (Annual fee math:
  see Task 6 / NOTE_FOR_ARCHITECT §0 — percent computed against the **annual** gross.)

- [ ] **Step 5: Verify (Lovable preview, Stripe test mode).** Start a `starter` `monthly` checkout
  for the TC Hub hub. Expected: line item €29.00 net + 21% IVA = €35.09 gross, charged on the
  hub's connected account; no reference to `mail`/`workspace`/`phone` anywhere. Repeat with
  `interval: annual` → €288.00/yr net + IVA, `recurring.interval = year`.

---

### Task 5: Compliance table attached to the subscriber

**Files:** New migration.
**Interfaces — Consumes:** `virtual_office_subscribers(id)`, `countries_config(id)`,
`prefix_registry(id)`. **Produces:** one compliance row per subscriber.

- [ ] **Step 1: Create the table.**

**Prompt (to Lovable):**
> Create a Supabase migration adding `public.virtual_office_subscriber_compliance` as below. RLS:
> service_role all; the subscriber's owner may read their own row (mirror the existing
> `virtual_office_subscribers` ownership policy).

```sql
create table public.virtual_office_subscriber_compliance (
  id uuid primary key default gen_random_uuid(),
  subscriber_id uuid not null references public.virtual_office_subscribers(id) on delete cascade,
  country_id text not null references public.countries_config(id),
  kyc_status text not null default 'pending' check (kyc_status in ('pending','submitted','verified','rejected')),
  uploaded_docs_manifest jsonb not null default '{}',
  outbound_telesales_flag boolean not null default false,
  assigned_geographic_number_id uuid references public.prefix_registry(id),
  assigned_national_number_id uuid references public.prefix_registry(id),
  verified_at timestamptz,
  created_at timestamptz not null default now(),
  unique (subscriber_id)
);
alter table public.virtual_office_subscriber_compliance enable row level security;
create policy "vosc service all" on public.virtual_office_subscriber_compliance for all to service_role using (true) with check (true);
```

- [ ] **Step 2: Verify.** Insert a test row against an existing subscriber id and confirm the FKs
  accept it; `select country_id, kyc_status from public.virtual_office_subscriber_compliance;`.

---

### Task 6: The "enable country" gate (DB trigger)

**Files:** New migration.
**Interfaces — Consumes:** `countries_config`. **Produces:** a trigger that blocks enabling an
under-configured country.

- [ ] **Step 1: Add the guard trigger.**

```sql
create or replace function public.enforce_country_enable_gate()
returns trigger language plpgsql as $$
begin
  if new.is_enabled is true then
    if new.vat_rate is null or coalesce(new.vat_label,'') = ''
       or array_length(new.required_kyc_docs, 1) is null then
      raise exception 'Cannot enable country %: VAT rate, VAT label and at least one required KYC document are mandatory', new.id;
    end if;
  end if;
  return new;
end $$;

create trigger trg_country_enable_gate
  before insert or update on public.countries_config
  for each row execute function public.enforce_country_enable_gate();
```

- [ ] **Step 2: Verify.** `update public.countries_config set is_enabled=true where id='ES';`
  succeeds (Spain is fully configured). Then try inserting a half-configured country with
  `is_enabled=true` and empty `required_kyc_docs` → expect the exception. Insert it with
  `is_enabled=false` → succeeds.

---

### Task 7: Admin "Country Setup" screen

**Files:** New admin route/page + form (follow existing admin UI patterns).
**Interfaces — Consumes:** `countries_config` (write), `prefix_registry`, `tier_price_recommendations`.

- [ ] **Step 1: Build the form.**

**Prompt (to Lovable):**
> In the admin area, add a "Country Setup" page that lists `countries_config` rows and lets an
> admin create/edit one. Fields: country id (2-letter), name, currency, **VAT rate** and **VAT
> label** (required), KYC weight, **required KYC documents** (add/remove list, at least one to
> enable), Tier-2 prefix zone, geographic-moat toggle, and the three recommended prices (monthly
> required, annual optional) which write to `tier_price_recommendations`. An "Enable country"
> toggle maps to `is_enabled`; if VAT or KYC docs are missing, disable the toggle in the UI and
> show "Add VAT + KYC documents before enabling." The DB trigger from Task 6 is the backstop.

- [ ] **Step 2: Verify.** Create a disabled draft country (e.g. `PT`) with no KYC docs → enable
  toggle is blocked. Add VAT + one KYC doc → toggle enables and saves.

---

### Task 8: Dynamic KYC upload UI at sign-up (Rule 2)

**Files:** Sign-up flow component.
**Interfaces — Consumes:** `countries_config.required_kyc_docs`. **Produces:** writes to
`virtual_office_subscriber_compliance.uploaded_docs_manifest`, sets `kyc_status='submitted'`.

- [ ] **Step 1: Render slots from config.**

**Prompt (to Lovable):**
> On the virtual-office sign-up flow, after the customer selects a country (only `is_enabled`
> countries appear), read that country's `required_kyc_docs` and render exactly one upload slot
> per document — no hardcoded per-country forms. On submit, store each file URL + state in the
> subscriber's `uploaded_docs_manifest` and set `kyc_status='submitted'`.

- [ ] **Step 2: Verify.** Select Spain → three slots appear (passport, company registry, proof of
  address). Confirm a draft `PT` with different docs renders different slots.

---

### Task 9: Operator pricing screen (the override UI)

**Files:** Hub-owner settings page.
**Interfaces — Consumes:** `tier_price_recommendations` (prefill), `virtual_office_plans` (write).

- [ ] **Step 1: Build the screen.**

**Prompt (to Lovable):**
> Add a "Pricing" screen for a hub owner. For each tier (Starter/Growth/Premium) show our
> recommended monthly and annual price for the hub's country (from `tier_price_recommendations`)
> as the prefilled default, labelled "Recommended". Let the owner enter their own monthly
> (required) and annual (optional) price, saved to `virtual_office_plans` keyed by `hub_id`+tier.
> Leaving annual blank means that tier has no annual option. Show the effective customer price
> (their value, else the recommendation).

- [ ] **Step 2: Verify.** Set TC Hub `starter` monthly to €25 → a new `starter monthly` checkout
  resolves to €25 net (overrides the €29 recommendation). Clear the override → checkout returns to
  €29.

---

## Deferred to a later phase (out of scope here — flagged, not built)

- **Outbound-telesales fee routing** — the `outbound_telesales_flag` column exists (Task 5); wiring
  its fee onto the Connect rails is a follow-up tied to telco provisioning (Zadarma).
- **Tier-3 hub↔zone provisioning + strict validation (Rule 3)** — validate the chosen hub's
  `phone_prefix` against `prefix_registry.geographic_prefix` at number assignment; needs the
  numbering/provisioning flow, a later phase.
- **Asset (desk/office) operator pricing** — same recommend+override pattern, separate table.
- **Non-Spain VAT rates + registration** — accountant brief before each country is enabled.

## Self-review notes

- Spec §2.1–2.5 → Tasks 1,2,3,6,7. Compliance §2.4 → Task 5. Checkout change → Task 4.
  Rules 1–3 → flag captured (Task 5), Rule 2 built (Task 8), Rule 1 fee + Rule 3 validation
  deferred (explicitly, above). Operator pricing → Task 9. No spec section left unmapped.
- Tier keys (`starter`/`growth`/`premium`), `hub_id`, `prefix_registry.id`, `country_code` used
  consistently across tasks.

# 📜 FLENTIX CONSTITUTION v2.1

**Version:** 2.3.0
**Last Updated:** 2026-06-23
**Status:** Active — Proof of Concept / Sprint Mode (pace set by Grant)
**Core Ecosystem Role:** Upstream Master Blueprint (Flentix Systems)
**Single source of truth.** Paste this as the first message in any new AI chat.

> ### What changed in v2.3 (2026-06-23 — read this first)
> - **The codebase is now FLENTIX-NATIVE — zero TC Hub legacy in code.** The
>   "de-TC purge" landed: the default workspace (`0fc40c16…`) is renamed
>   **"Flentix Systems" (slug `flentix`, emerald)**; the real TC Hub re-enters
>   as a **fresh onboarded hub (data only)** at migration, like any hub. Money
>   path renamed: Stripe `brand: 'FLENTIX'`, metadata keys `fx_*`, statement
>   descriptor `FLENTIX`; reservation codes are now **`FLX-…`**. Proven
>   end-to-end by live test bookings (FLX-1017/1018, paid, emerald emails,
>   TTLock, live in admin).
> - **Per-hub branding is now LIVE, not just a mandate (§1 white-label).**
>   Emails (`supabase/functions/_shared/branding.ts`) and the web app
>   (`src/components/HubThemeProvider.tsx` + `src/lib/hexToHsl.ts`) render each
>   hub's logo / colours / name / reply-to from its `workspaces` row. The §2
>   TC-Hub-orange drift is **RESOLVED**; per-hub colour is data-driven by domain.
>   ✅ **Brand reconciled + aligned (2026-06-23):** canonical = **teal `#16C7B4`**
>   primary / **emerald `#10B981`** success-only; the repo was aligned to it (tokens,
>   marketing→tokens, unified `FlentixLogo`, TC-raster logo purged — P1–P3).
> - **Admin restructured** — a dedicated **Overview** section (dashboard stats /
>   revenue / mailroom / occupancy grid) separated from the clean operational
>   sections, plus a tab-refocus auto-refetch fix.
> - **⚠️ NEW go-live blocker — per-hub Factura fiscal identity.** Stripping TC
>   Hub's hardcoded Spanish fiscal identity (NIF `B19703941`, Almuñécar) left
>   the customer-Factura **seller NIF/address blank**. Each hub's real fiscal
>   identity (legal name / NIF / registered address) must be stored on its
>   workspace and printed on its Facturas — onboarded at migration. See
>   `GO_LIVE_CHECKLIST.md`.
> - **Live TC Hub migration (Plan B)** scheduled ~**Sunday 2026-06-28** (quiet
>   window). Needs a **temporary dual-accept** in `stripe-webhook` so TC Hub's
>   pre-existing `'TCH'`/`tc_hub_*`-tagged Stripe records still settle during
>   cutover (their metadata can't be rewritten).

> ### What changed in v2.1 (read this)
> - **Entities corrected** — the legal/contracting entity is **Medacrii
>   Associates Ltd** (UK); "Flentix" is the product brand, not yet a company
>   (§0, §1).
> - **AI workflow updated** to current reality — **Gemini is Architect**;
>   **Claude does engineering/payments/tax/audit** (not "design only"). The old
>   "DeepSeek architect / Gemini phasing out / Claude components-only" model is
>   retired (§5).
> - **Brand confirmed canonical = Emerald/charcoal/geometric.** The live sandbox
>   has **drifted** to a sunset/glassmorphism look (the TC Hub tenant skin) and
>   must be re-skinned to brand before any Flentix-branded launch (§2).
> - **Stripe is now being wired up**, with a defined fee/tax model (§4a). See
>   `NOTE_FOR_ARCHITECT.md` and `ACCOUNTANT_BRIEF.md`.
> - Removed the false claim that "Stripe handles reverse-charge VAT
>   automatically" — it does not (§4a).

---

## 0. ENTITIES (brand vs legal entity — do not conflate)

| Name | What it is |
|---|---|
| **Medacrii Associates Ltd** | **UK limited company.** The current legal, contracting and **invoicing** entity, and the Stripe **platform** account holder. Issues the platform/PBX fee invoices to operators. |
| **Flentix (Systems)** | The **product/system brand**. Will become its own legal entity **after** PoC; **jurisdiction TBD**. Not a legal entity today. Where older copy treats "Flentix Systems" as the company, read **Medacrii Associates Ltd**. |
| **Operators ("hubs")** | Medacrii's clients — VAT-registered Spanish co-working / virtual-office businesses (e.g. TC Hub). Each has its own Stripe **connected** account. |
| **End-members** | The operators' own customers. |

---

## 1. VISION & CORE MISSION

Flentix Systems is an enterprise-grade, multi-tenant B2B SaaS co-working
management platform and booking engine engineered for pan-European deployment.
It enables co-working space operators to monetize spaces, automate hardware
access, handle real-time billing, and deliver fully white-labeled experiences.

**Current Status (PoC):**
- ~85% feature-complete.
- Live test operator: **TC Hub (tchub.es)** — fully functional.
- Master sandbox: https://flentix-systems-sandbox-v2.lovable.app/virtual-office
- Stripe: **being wired up** (fee/tax model defined — see §4a).
- TTLock: Live. Brevo: Live. Tako CRM: built, needs integration.
- Zadarma PBX: strategy defined — Phase 1 (§8).

**White-Label Mandate (Non-Negotiable):** All components must dynamically adjust
styling and data visibility based on tenant context. Hardcoded branding is
strictly prohibited.

---

## 2. BRAND IDENTITY (FLENTIX SYSTEMS — CANONICAL)

> ✅ **TC-Hub orange drift RESOLVED (2026-06-23):** the sunset/orange skin and all
> hardcoded orange were purged in the de-TC pass. Per-hub colour is **data-driven**:
> a hub's `workspaces.primary_color` themes its emails and web pages (by domain) via
> `_shared/branding.ts` and `HubThemeProvider`. **⚠️ Brand primary still being
> aligned:** the canonical brand accent is now **teal `#16C7B4`** (the Primary Colors
> table below) with **emerald `#10B981` reserved for success states only** — but the
> repo currently ships emerald as `--primary` and the marketing pages hardcode teal
> off-token. Aligning the token + marketing pages to teal-primary is a tracked task.
> The Drive **Brand System** README is the canonical brand source (it supersedes the
> older `BRAND_GUIDELINES.md`).

### Primary Colors (current design system — teal-primary; supersedes the prior emerald-primary)
| Token | Color | Hex | Usage |
|---|---|---|---|
| `--primary` | **Teal** | **#16C7B4** | **Brand accent** — CTAs, links, active states, tags |
| `--foreground` | Charcoal | #1F2937 | Body text |
| `--inverse` | Slate | #0B121A | Hero / CTA / footer dark bands |
| `--background` | Cream | #FAF9F5 | Page canvas (white #FFFFFF cards) |
| `--success` | Emerald | #10B981 | **Success / confirmed / paid states ONLY — NOT the brand accent** |

### Supporting Colors
Light Gray (`--muted`) #F3F4F6 · Medium Gray (`--muted-foreground`) #6B7280 ·
Border #E5E7EB · Error Red #EF4444 · Warning Orange #F59E0B.

### Typography
- Headings: **Montserrat** (600, 700).
- Body: **Open Sans** (400, 600).

### Design Principles
1. **Clean & Minimal** — remove visual noise.
2. **Geometric Precision** — sharp corners, 90° angles, consistent grid.
3. **Purposeful Whitespace** — 8px spacing scale.
4. **Hierarchy First** — largest type = most important.

### CSS Implementation (default fallbacks — Flentix master)
```css
:root {
  --tenant-primary: #16C7B4;      /* Teal — Flentix brand accent */
  --tenant-primary-dark: #0FAE9C;
  --tenant-success: #10B981;      /* Emerald — success/paid states only */
  --tenant-radius: 4px;           /* geometric but approachable */
  --tenant-font-heading: 'Montserrat', sans-serif;
  --tenant-font-body: 'Open Sans', sans-serif;
}
```
Tenants override `--tenant-*` / `--primary` to theme their own deployment (per-hub,
by domain). **Hardcoding any brand hex in a component breaks white-label** — every
surface must read the token.

> ✅ **Repo aligned to this system (2026-06-23, commits 3727dd56 / f26a02a6 / d67b4bab):**
> `src/index.css --primary` = teal `174 80% 43%` (#16C7B4), `--success` = emerald,
> `--inverse` = slate, `--radius` = 4px; all marketing pages read `hsl(var(--primary))`
> (so per-hub theming reaches them); the Flentix workspace row + email defaults are teal;
> all logos are unified in `src/components/FlentixLogo.tsx` (tokenised F-mark) and the old
> TC-Hub raster logo was purged. See the flentix-backlog memory (P1–P3).

---

## 3. TECH STACK

| Layer | Technology |
|---|---|
| Frontend | React 18, Vite, Tailwind CSS, TanStack Query v5 |
| Assembly | Lovable.dev |
| UI Components | shadcn/ui (Radix primitives) |
| Backend | Supabase (PostgreSQL, Realtime, Edge Functions) — via Lovable Cloud |
| Security | RLS — no client-side service keys |

> **Note:** decoupling from Lovable onto self-hosted infrastructure is being
> evaluated for *after* PoC (it's a managed Supabase under the hood, so portable).
> No action during PoC.

### Third-Party Integrations
| Service | Purpose | Status |
|---|---|---|
| Stripe | B2B subscriptions, Connect, payment intents | Being wired up (§4a) |
| TTLock | Smart lock PIN codes | Live |
| Brevo | Transactional emails | Live |
| Tako CRM | Customer profiles, metrics | Built, needs integration |
| Zadarma | PBX (telephony) | Strategy defined — Phase 1 (§8) |

---

## 4. DATABASE & RLS (SUMMARY)

### Core tables
- `profiles` / `users` — roles: `super_admin`, `operator`, `customer`.
- `spaces` / `hubs` — physical locations, linked via `operator_id`.
- `bookings` — transactions: `pending`, `confirmed`, `cancelled`.
- `subscriptions` / `passes` — SaaS licenses, map to Stripe.

### RLS policies (non-negotiable)
1. Operators access only resources where `operator_id` matches `auth.uid()`.
2. Customers interact only with own profile, bookings, and public availability.
3. All mutations use `.select()` to trigger immediate RLS exceptions.

---

## 4a. BILLING, FEES & TAX (CANONICAL)

Detail in `NOTE_FOR_ARCHITECT.md` (§0 = locked decisions); tax positions for the
accountant in `ACCOUNTANT_BRIEF.md`. Load-bearing facts:

- **Model 1 — everything runs through Flentix.** The end-member pays via the
  Flentix checkout; the charge is a **direct charge** on the operator's
  **connected** account (`{ stripeAccount: connectAcct }`); the **operator is
  Merchant of Record**; Medacrii's cut is split out instantly as a Stripe
  **application fee**. The "wholesale-only / operators bill in their own ERP"
  model is **rejected** — operators use *only* Flentix.
- **Charge = net price + 21% Spanish IVA**, the operator's to remit. Medacrii's
  fee is charged on the **NET**, never the IVA.
- **Fixed per-tier fee mechanism.** `application_fee_amount` is valid **only for
  one-off payments** (`payment_intent_data`), **never** in `subscription_data`.
  VO products are subscriptions, so: first invoice → `application_fee_percent`
  computed to equal the fixed fee; renewals → `application_fee_amount` on the
  draft invoice via an `invoice.created` webhook (Connect events;
  `{ stripeAccount: event.account }`).
- **Fee figures are PROVISIONAL** (€2/€30/€50 placeholders; Grant researching).
- **Phone tier = single €119 charge** on the connected account, like other tiers,
  through normal KYC. The old two-stage (€104 + separate €15 telecom) is
  **removed** — pre-Connect cruft.
- **Reverse charge** on Medacrii's fee is handled by **Medacrii issuing its own
  B2B invoices** to operators. **Stripe does NOT do this automatically.**
- **Usage pass-through (future):** metered, billed at 100% cost, no margin.
- **Record settlement figures (base/tax/gross/fee) on `virtual_office_subscribers`**
  (NOT `bookings` — VO is a recurring contract, not a transient booking), written
  from the **actual finalised invoice** in `invoice.payment_succeeded`.

---

## 5. MULTI-AI WORKFLOW (current)

| Role | Tool | Responsibilities |
|---|---|---|
| **Owner / Gatekeeper** | Grant | Holds this Constitution; sets direction & pace |
| **Architect** | **Gemini** | System design, DB, RLS, API contracts, blueprints |
| **Engineering / Payments / Tax / Audit** | **Claude** | Stripe/tax correctness, code & blueprint audits, repo work, edge functions |
| **Assembly** | **Lovable.dev** | Building, deployment |
| **Chief Architect (legacy)** | DeepSeek | Prior architect/Constitution author — *current role to confirm* |

### Constitution-driven workflow
- This Constitution is the **single source of truth**.
- Every new AI chat starts with it pasted as the first message.
- **Grant** is the Constitution Gatekeeper.

> **Note:** the old v2.0 rule "Claude builds atomic components only — no backend
> logic" is **retired**. Claude now owns payments/tax/engineering audit.

---

## 6. DESIGN GOVERNANCE (white-label pattern)

### White-label pattern
```tsx
className="bg-[var(--tenant-primary, #10B981)] rounded-[var(--tenant-radius, 4px)] hover:opacity-90"
```

### Forbidden patterns
- ❌ Hardcoded hex codes or brand colors (breaks white-label).
- ❌ Page-level redesigns (keep in Lovable).
- ❌ Backend logic inside presentation components.

---

## 7. VIRTUAL OFFICE — 3-TIER OFFERING

**Tier 1 — Business Footprint (Starter):** legal presence + basic mail handling;
1 Flexi-Desk day pass/month. Target: founders needing a legal "pin on the map".

**Tier 2 — Workspace Hybrid (Growth):** priority mail (30-day + forwarding);
2 day passes + 2 meeting-room hours/month; member rates. Target: local remote
professionals.

**Tier 3 — AI Executive Suite (Premium):** everything in Tier 2 + Smart PBX +
AI receptionist; dedicated Spanish DID; smart call routing; voicemail-to-email;
10 extensions; 4 day passes + 4 meeting-room hours/month. Target: established
companies wanting an elite image.

**KYC (all tiers, critical for Tier 3):** checkout includes ID/passport + address
proof upload for Spanish telephony compliance.

> Legacy `mail` Stripe price stays live for existing subscribers; the funnel
> presents 3 tiers (Starter/Growth/Premium).

---

## 8. PBX SYSTEM ARCHITECTURE (ZADARMA-FIRST)

**Phase 1 — market validation via Zadarma Dealer API.** No custom PBX UI build
until volume justifies; use Zadarma's native interface initially.

### 11-step integration sequence
1. Register user — `POST /v1/reseller/users/registration/new/`
2. Confirm registration — `POST /v1/reseller/users/registration/confirm/`
3. Add contact number — `POST /v1/reseller/users/phones/add/`
4. Verify via SMS — `POST /v1/reseller/users/phones/prove_by_sms`
5. Confirm phone — `POST /v1/reseller/users/phones/confirm`
6. Transfer funds — `GET /v1/reseller/users/topup/`
7. Purchase number — Virtual number API + `user_id`
8. Create PBX — `POST /v1/pbx/create/`
9. Add extensions — `POST /v1/pbx/internal/create/`
10. Call routing — `POST /v1/pbx/ivr/scenario/create/`
11. Configure webhooks — `POST /v1/pbx/webhooks/url/`

**Webhook architecture:** your server is the control plane — receives call
notifications, looks up client ownership, triggers IVR, routes calls, logs for
billing. **White-label confirmed** (clients never see Zadarma).

**Phase 2 migration path:** provider-agnostic Facade Pattern — same internal API
signatures, swap implementation for Twilio/FreeSWITCH later.

**Dealer account:** email `manage@zadarma.com` — no setup fees, pay-as-you-go.

---

## 9. SYSTEM HARDENING (already solved)

- **Async pipeline (`detachedInvoke`):** UI waits only for the DB write;
  third-party calls (Stripe, TTLock, Tako) run in `Promise.allSettled` with an
  8000ms timeout. If an integration hangs, a toast warns but the DB commit stands.
- **iOS PWA stability:** `AdminErrorBoundary.tsx` catches mobile render
  exceptions; deterministic ISO-date chart mapping; session purge on PWA
  reactivation; custom back-nav for standalone mode.

---

## 10. CONSTITUTION GOVERNANCE

- **Owner / Gatekeeper:** Grant.
- **Storage:** this repo (`flentix-docs`), `CONSTITUTION.md`.
- **Versioning:** semantic (v2.1, v2.2, …).
- **New chat onboarding:** always paste this Constitution first.

### Changelog
- **v2.3.0 (2026-06-23):** **Flentix-native milestone.** De-TC purge — zero TC
  Hub legacy in code; default workspace `0fc40c16` renamed → "Flentix Systems"
  (slug `flentix`); money path `brand:'FLENTIX'` / `fx_*` keys / `FLX-` codes;
  TC fiscal identity (NIF B19703941 / Almuñécar) scrubbed; auto-admin trigger
  re-asserted clean. **Per-hub branding live** (emails via `_shared/branding.ts`;
  web via `HubThemeProvider`+`hexToHsl`). **Admin Overview restructure** +
  refocus refetch. Operator BCC removed from customer emails. §2 brand-drift
  resolved. **Canonical brand corrected to teal #16C7B4 primary / emerald #10B981
  success-only** (§2) **and the repo aligned to it** (P1 tokens, P2 marketing→tokens,
  P3 unified `FlentixLogo` + TC-raster logo purge; commits 3727dd56/f26a02a6/d67b4bab).
  NEW go-live
  blocker logged: per-hub Factura fiscal identity (seller
  NIF/address now blank). Plan B (live TC migration) set for ~Sunday with a
  webhook dual-accept for TC Hub's legacy Stripe metadata. Verified e2e (FLX-1017/1018).
- **v2.2.0 (2026-06-19):** billing **Model 1 confirmed** (end-users pay through
  Flentix; operator = MoR; wholesale-only model rejected); locked the
  `application_fee_amount` rule (one-off only, never in `subscription_data`);
  **two-stage Phone checkout removed** → single €119; settlement figures moved to
  `virtual_office_subscribers`; fee figures marked provisional.
- **v2.1.0 (2026-06-19):** entities corrected (Medacrii); AI workflow updated
  (Gemini architect, Claude engineering/tax); brand confirmed emerald-canonical
  with sandbox-drift note; Stripe fee/tax model added; removed false
  "Stripe auto reverse-charge" claim; consolidated into `flentix-docs`.
- **v2.0.0 (2026-06-13):** prior master blueprint (DeepSeek).

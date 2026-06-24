# Spec — Telecom Phase 2: Zadarma number provisioning (Premium tier)

**Audience:** Gemini (architect), Lovable build, future humans/AIs. **Owner:** Grant.
**Date:** 2026-06-24. **Status:** DESIGN — for founder review before any build. Sandbox-first; no live keys.
Read with [GO_LIVE_CHECKLIST.md](GO_LIVE_CHECKLIST.md), [NOTE_FOR_ARCHITECT.md](NOTE_FOR_ARCHITECT.md),
[SPEC_country-rollout-and-operator-pricing.md](SPEC_country-rollout-and-operator-pricing.md).

## 0. Context
The core VO flow is built + verified end-to-end (checkout → KYC → emails → admin order, `pending_activation`).
The **Premium** tier includes a **local geographic phone number** (the moat) which is NOT yet provisioned —
the admin panel shows "Awaiting client card verification" / "Awaiting InfoCaller provisioning" against
mock/sandbox stubs (`create-telecom-setup-intent`, `send-phase2-activation-link`, `confirm-telecom-activation`,
`ActivatePhone`). This spec replaces those stubs with a real **Zadarma** integration.

**Provider = Zadarma** (account: MEDACRII ASSOCIATES LIMITED). Its API supports the whole flow incl.
**document-based KYC submitted via API** and a **sandbox** (`USE_SANDBOX`). Refs:
https://zadarma.com/en/support/api/ , https://zadarma.com/en/support/instructions/api/numbers/

## 1. Locked decisions (founder, 2026-06-24)
1. **Geographic numbers are only offered where Zadarma can supply them.** Pre-check availability per zone
   *before* Premium is offered for a hub — so "no number available after payment" cannot occur.
2. **Invoice (Factura) is raised when the money is received** (at payment), NOT at activation. This OVERRIDES
   the current activation-time Factura. If the customer subsequently fails verification, **issue a credit note +
   refund** (reuse the existing credit-note + Stripe refund machinery).
3. **We run our OWN lightweight AI document pre-check** at upload (classify each file vs the expected document
   type for its slot) to catch wrong/illegible uploads instantly. This is a **sanity/quality gate, NOT** the
   legal KYC — **Zadarma performs the binding regulatory verification**. Goal: reduce Zadarma rejection round-trips.
4. **Sandbox first** (`USE_SANDBOX=true`); live Zadarma keys only at go-live.
5. **AI receptionist / AI call handling is coming** (part of the Premium value prop) — designed separately;
   this spec must not preclude it (the SIP/PBX routing step is where it will later attach, via Zadarma's
   real PBX webhook `/v1/pbx/webhooks/url/` — the confirmed call-event webhook).
6. **Fully automated, no human bottleneck (founder rule).** The whole Premium provisioning runs end-to-end
   with NO admin click on the happy path: pay → AI doc-check → auto-submit to Zadarma → auto-validate →
   auto-order number → auto-route → auto-activate → email. A human admin is pulled in ONLY on a genuine
   exception Zadarma flags that the system can't auto-resolve. A doc rejection auto-emails the customer to
   re-upload (no human). No artificial "admin Activate" gate, and no separate client card-verification detour.

## 2. End-to-end flow (Premium)
1. **Offer-time availability pre-check (storefront).** When Premium is shown for a hub, confirm Zadarma has
   geographic numbers for that hub's zone — via `GET /v1/direct_numbers/countries/` +
   `GET /v1/direct_numbers/available/<DIRECTION_ID>/` (cached), or a pre-curated allow-list of enabled zones per
   hub. Premium is only selectable where coverage exists.
2. **Checkout (existing).** Single Premium charge on the operator's connected account (unchanged).
3. **Invoice at payment (CHANGE).** On payment success, issue the operator's Factura immediately (move the
   Factura issuance out of activation into the payment/webhook path). Depends on **per-hub fiscal identity**
   (go-live blocker — seller NIF/address). ⚠️ Accountant to confirm payment-time issuance.
4. **KYC upload (existing) + AI pre-check (NEW).** Customer uploads the country's `required_kyc_docs`. A new
   edge function classifies each uploaded file with a vision LLM (recommend Claude vision) → is it the expected
   type (passport / company_registry / proof_of_address …)? Store the classification + confidence on
   `virtual_office_subscriber_compliance`. If a clear mismatch/illegible → reject that slot immediately with a
   friendly message and let them re-upload (no Zadarma call yet). Borderline → allow through (Zadarma decides).
5. **Provisioning — AUTO, no admin click.** A backend orchestrator edge function (adapt
   `confirm-telecom-activation` or new `provision-zadarma-number`) fires automatically once the AI doc-check
   passes:
   a. `POST /v1/documents/groups/create/` → create a Zadarma document group.
   b. `POST /v1/documents/upload/` → upload the stored KYC docs **one file at a time**, looping sequentially;
      record each returned Zadarma file-ID into the compliance JSONB (`ai_doc_check`/manifest mapping).
   c. `GET /v1/documents/groups/valid/<ID>/` → validate for the Spanish destination. **Treat as synchronous**
      (immediate pass/fail). If a destination turns out to require async operator review, use an **automated
      scheduled re-check** (Supabase scheduled function) — NOT a human, NOT an assumed webhook. (No confirmed
      Zadarma document-status webhook exists; do not design around one.)
   d. `POST /v1/direct_numbers/order/` → order a geographic number in the hub's zone (map hub zone → `DIRECTION_ID`).
   e. `PUT /v1/direct_numbers/set_sip_id/` → route the number to the PBX/SIP (later: to the AI receptionist).
   f. On success: set `assigned_geographic_number_id` + Zadarma number/doc-group ids, flip subscriber to
      `active`, send the activation email.
6. **Failure branch — auto first, human only if unresolvable.** Zadarma validate fails → `kyc_status='rejected'`,
   **auto-email** the customer + reuse "resend KYC link" (no human). If genuinely unfulfillable →
   **credit note + Stripe refund**, subscriber → cancelled. Escalate to an admin ONLY when Zadarma flags
   something the system can't auto-handle.
7. **Card-on-file for future usage — reuse the subscription's payment method, no separate detour.** The
   subscription already has the customer's payment method on file (Stripe), so we do NOT need a separate
   Phase-2 SetupIntent round-trip (`send-phase2-activation-link` / `ActivatePhone`) — those client steps are
   RETIRED. Future metered call-usage billing reuses the existing subscription PM. (Usage billing itself = future.)

## 3. Data
- Reuse `virtual_office_subscriber_compliance`: `assigned_geographic_number_id` (already exists) + add
  `zadarma_number` (text), `zadarma_doc_group_id` (text), `ai_doc_check` (jsonb: per-slot classification +
  confidence), `provisioning_status` (text: pending|docs_uploaded|validated|ordered|active|rejected|failed).
- No change to pricing tables.

## 4. Secrets / config
- `ZADARMA_API_KEY`, `ZADARMA_API_SECRET` (added by Grant; sandbox values first), `ZADARMA_USE_SANDBOX=true`.
- AI doc-check: **Claude Haiku 4.5** (current, vision-capable, fast, <1¢/check) — NOT the outdated 3.5 Sonnet;
  keep a single provider (Claude), don't add OpenAI. `ANTHROPIC_API_KEY`.
- Per-hub fiscal identity (for the payment-time Factura) — go-live blocker.

## 5. Out of scope (separate work)
- **AI receptionist / conversational call handling** (future; attaches at the SIP routing step).
- Metered call-usage billing (future; the card-on-file is the hook).
- Live Zadarma keys (go-live).
- Non-Spain geographic provisioning (per-country, later).

## 6. Open items for the build
1. **Accountant sign-off (go-live gate, NOT a build blocker):** build assuming Factura at **payment** +
   credit-note/refund on failed verification (founder-confirmed). Lawyer/accountant confirms before LIVE
   (Spanish IVA + operator-as-MoR specifics).
2. ~~auto-run vs admin click~~ **RESOLVED: AUTO-RUN** — provisioning fires automatically after the AI pre-check
   passes; admin only on a genuine Zadarma exception (founder rule §1.6).
3. ~~vision-LLM choice~~ **RESOLVED: Claude Haiku 4.5** (single-provider).
4. Map the hub's zone → Zadarma `DIRECTION_ID` for the availability pre-check + number order (build step).
5. **Stub reconciliation (corrected):** the orchestration is a BACKEND edge function — adapt
   `confirm-telecom-activation` (or new `provision-zadarma-number`). The client-facing Phase-2 detour
   (`send-phase2-activation-link` + `ActivatePhone` page + `create-telecom-setup-intent`) is **RETIRED** —
   no separate card-verification round-trip (reuse the subscription payment method).
6. **Verify with Zadarma:** is document validation synchronous, or is there an async review per destination?
   (Design synchronous-first + automated scheduled re-check fallback; no document-status webhook is assumed.)

## 7. Architect feedback (Gemini) — evaluated 2026-06-24
- **Doc-verification webhook:** principle (event-driven, no human) ✅ adopted; but the specific claim is
  **rejected** — Zadarma's webhooks are call/PBX events, no confirmed document-status webhook. Validate is
  synchronous-first; automated poll only if a destination needs async review. The real PBX webhook is reserved
  for the future AI receptionist.
- **Sequential multi-file upload + per-file ID mapping:** ✅ adopted (§2.5b).
- **Adapt-not-delete stubs:** ✅ adopted, corrected — orchestration is backend (not the `ActivatePhone` page),
  and the client card detour is retired (no bottleneck).
- **Payment-time invoicing:** ✅ build on it; accountant = go-live gate.
- **Auto-run:** ✅ adopted as the backbone (§1.6).
- **Vision LLM:** ✅ adopted, model updated to Claude Haiku 4.5 (3.5 Sonnet is outdated; single provider).

## 8. Usage / overage billing (founder requirement 2026-06-24 — PENDING decisions + Zadarma model confirm)
Phone tiers (Growth/Premium) include a monthly **free-minute allowance**; usage above it is charged to the customer.

**Reality check — Zadarma is PREPAID:** it deducts call cost from the MEDACRII balance **in real time, per call**
(dashboard shows balance + Top-Up), NOT a periodic invoice. So there is no "Zadarma debits my account in 48h"
event — depletion is continuous. ⚠️ **CONFIRM** whether the distributor account has any postpaid/invoiced terms;
if not, the prepaid design below applies.

**Founder goal:** collect the customer's overage money BEFORE Flentix is out of pocket; if the customer doesn't
pay, CUT their service immediately (no buffer), mirroring how Zadarma cuts off on an empty balance.

**Design (prepaid model):**
- Meter per-customer usage by pulling Zadarma **CDR/statistics** for each customer's number/SIP on a schedule.
- Track usage vs the tier's free allowance per cycle; overage = (minutes used − free) × rate.
- **Threshold-based recovery (recommended):** when a customer's accrued overage reaches a cap (e.g. €20),
  email them (**48h notice**), then **auto-charge their saved card off-session** (reuse the subscription PM).
  Flentix never fronts more than the cap.
- **On charge failure → suspend the customer's Zadarma number immediately** (remove SIP routing / suspend),
  no grace; restore on successful payment.
- Recovered funds replenish the Zadarma balance (manual or auto top-up) so paying customers keep service.

**Decisions needed (founder):**
1. **Free minutes:** Growth = ?, Premium = ? — AND to WHICH destinations (Spanish national vs mobile vs
   international differ hugely in cost) and INBOUND vs OUTBOUND (allowance is usually outbound).
2. **Overage rate:** pass-through at Zadarma cost (no margin — per prior note) OR cost + margin? Per-destination.
3. **Recovery trigger:** threshold (€ cap) vs monthly cycle.
4. **Notify window:** 48h (founder suggestion) — confirm; charge after notice.
5. **Suspend/restore:** hard cut on failure, no grace (founder-confirmed); restore on payment.

**Compliance/UX:** variable off-session charges need a clear **usage-billing mandate disclosed at signup** +
advance notice (the 48h satisfies this) — fold into the consent step. (Stripe off-session PaymentIntent on the
saved PM; SCA/mandate considerations apply.)

## 9. Usage billing — ADOPTED model (prepaid "tripwire" top-up) + corrections (2026-06-24)
Supersedes the threshold-recovery sketch in §8. Founder + architect aligned on a **prepaid top-up** model
(better cash-flow — customer pays before minutes are used; Flentix never fronts).

**Packages (founder to finalise exact minutes):** Inbound = FREE (€0/min to us). Growth = ~100 outbound min;
Premium = ~300 outbound min — **outbound, to the STANDARD destination set only** (Spain/EU national landlines,
~€0.015/min wholesale). Monthly counter resets on the billing anniversary.

**Tripwire top-up:** at 0 minutes → pause the outbound trunk (Zadarma API) → off-session **€10 + 21% IVA**
charge on the **operator's connected account** (card-on-file) → on success webhook, credit **250 outbound
minutes**. A small internal buffer (~€5 worth) absorbs in-flight-call overrun before the charge clears.
Fail-branch: outbound stays hard-blocked (inbound stays free/active), dashboard alert "update billing to
reactivate". Auto-top-up is a per-customer toggle (off = manual top-up; cut at zero either way — no grace).

**Money routing (operator MoR + Medacrii sweep) — CORRECTED:**
- Charge is a one-off PaymentIntent on the **operator's connected account** (operator = MoR; valid to use
  `application_fee_amount` on one-off PaymentIntents).
- Medacrii sweeps its margin via `application_fee_amount` computed on the **NET (ex-IVA)** — **NEVER sweep the
  IVA**: the operator must RETAIN the €2.10 IVA to remit to Hacienda. Operator keeps IVA + the optional €0.50
  kickback; **Medacrii absorbs the Stripe processing fee** (not the operator). Keep the €0.50 kickback ON for
  economic substance (operator must not be left at literal €0 / paying VAT on swept revenue). Accountant to
  bless the sweep + note Medacrii-supplies-telecom-wholesale in the operator agreement/DPA.

**Destination guardrails (loss prevention):** allowances + €10 blocks apply ONLY to the standard cheap zone
(Spain/EU landlines). Block high-cost international + premium-mobile routes by default on standard tiers via the
Zadarma trunk config (`PUT /v1/pbx/`) — prevents toll fraud and per-block losses (a 250-min block to mobiles
could cost far more than €3.75). Per-destination rates/opt-in only if a customer needs those routes.

## 10. Corporate accounts — multi-line & multi-seat (founder priority; design now, build after core)
Two distinct concepts (keep separate):
- **A. Internal extensions (multi-USER):** team seats sharing ONE geographic number (Ext 101/102) via
  `POST /v1/pbx/internal/` (free from Zadarma). Upsell a flat **per-seat monthly fee** (operator Connect +
  Medacrii app-fee sweep). Inbound routes via the future AI receptionist to extensions.
- **B. Additional geographic lines (multi-NUMBER):** separate DIDs (CEO direct, Barcelona branch). "+ Add Line"
  → recurring **€15–20/mo + IVA** (operator Connect + sweep; ~€2/mo wholesale) → on success, auto-run the
  Zadarma provisioning loop to order + attach the number to that account.
  - ⚠️ **Each additional geographic line carries its OWN per-zone KYC + in-zone address requirement** (the moat
    is per-number). "+ Add Line" only offers zones we + Zadarma cover; each new line gets its own document
    validation. A Barcelona line needs a Barcelona-zone address.

**Data model — design for 1:N NOW (avoid a rebuild), even though v1 ships single-line:**
- `vo_lines` — one row per DID/number: subscriber_id, zadarma_number, zone/hub, status, its own compliance refs,
  recurring price. (The current per-subscriber compliance becomes per-LINE.)
- `vo_seats` — one row per user/extension: subscriber_id, extension, name, login, seat price.
- A `virtual_office_subscribers` row → many `vo_lines` and many `vo_seats`.

## 11. Build sequence (founder-agreed strategy 2026-06-24)
1. **Core single-line provisioning** (Zadarma sandbox) — schema modelled for 1:N from day one.
2. **Usage / tripwire billing** (minutes ledger, €10+IVA top-up via operator Connect + Medacrii NET sweep,
   destination guardrails, hard-cut, auto-top-up toggle).
3. **Corporate multi-line + multi-seat** (extensions + additional geographic lines, recurring billing).
4. **In-browser WebRTC dialer** (embed Zadarma WebRTC + 72h key via `GET /v1/webrtc/get_key/`; SIP-creds
   fallback for power users) — its own sizable workstream.
5. **AI receptionist** (call screening/routing) — separate.
Each step builds with the founder's per-step go-ahead; 1–2 are the foundation, 3–5 layer on.

**BUILD PROGRESS:**
- ✅ **Step 1 — data-model foundation BUILT & verified 2026-06-24** (commit 1d90e4bf): `telephony_config`
  (seeded, editable: 100/300 min, €10→250, €19 line, €9 seat, €0.50 kickback, ES_EU_LANDLINE),
  `vo_lines`, `vo_seats`, `vo_account_minutes`, `vo_minute_ledger` (append-only, idempotent on
  `zadarma_call_id`). RLS mirrors `virtual_office_subscriber_compliance`. No Zadarma/Stripe/edge code touched.
- ✅ **AI doc pre-check BUILT & verified 2026-06-24** (commit 89c8b3ec): `kyc-ai-precheck` edge fn
  (Claude `claude-haiku-4-5-20251001` vision, non-fatal) + `ai_doc_check`/`ai_precheck_status` columns on
  compliance. Live test against the real test subscriber correctly FLAGGED the uploaded files as
  screenshots (not KYC docs) with per-slot reasons — gate working, result persisted. `ANTHROPIC_API_KEY` set.
- ✅ **Zadarma connectivity proven 2026-06-24** (commit 1c299f64): `zadarma-probe` authenticates against
  BOTH sandbox + production (200, balance €50.40). **Same keys work for both** (no separate sandbox keys) — flag
  toggles host. **Signature gotcha fixed:** Zadarma = `base64(HEX(hmac_sha1(...)))` (PHP hash_hmac default-hex),
  NOT base64 of raw bytes (was 28 chars, must be ~56). Shared auth helper (md5 + hmac-hex-base64) for reuse.
- ✅ **Number availability + launch-zone map confirmed 2026-06-24** (`zadarma-numbers`). Correct Zadarma flow:
  `/v1/direct_numbers/countries/` (ISO) → `/v1/direct_numbers/country/?country=ES` (destinations w/ DIRECTION_ID,
  areaCode, monthly_fee, KYC restrictions) → `/v1/direct_numbers/available/<DIRECTION_ID>/` (orderable numbers).
  **Launch-city DIRECTION_IDs (standard tier, €3.40/mo wholesale, €0 connect; silver €4/€30, gold €6/€70):**
  Granada=16741 (area 858), Málaga=16593 (95), Sevilla=16617 (854), Madrid=16284 (91), Barcelona=16285 (93),
  Valencia=16287 (96). **NOT on Zadarma: Marbella** (use Málaga area-95 to cover Costa del Sol, or alt provider)
  **& Santander** (no Zadarma Cantabria DID at all). 159 ES destinations total.
  **FOUNDER DECISION 2026-06-24:** launch map = the 6 confirmed cities. Marbella + Santander DROPPED from
  day-one (Marbella likely covered by Málaga's 95 prefix anyway). Founder will map the real geographic coverage
  of each prefix after the base offices are set up, then decide if any extra zones can be served. ⚠️ IDs from SANDBOX — re-confirm
  against production at go-live (catalog should match). **MOAT LINK:** a geographic number needs an in-zone
  ADDRESS — each launch city must have a matching Flentix hub/address, and we store its DIRECTION_ID on the
  hub/zone so provisioning + "only offer where we can deliver" both work.
- ✅ **Document pipeline proven in sandbox 2026-06-24** (`zadarma-provision`): requirements probe (no dedicated
  endpoint — use `groups/valid/<id>/` → 4 booleans is_information_match/is_documents_uploaded/
  is_documents_verified/is_address_match), `groups/create/`, multipart file `upload/` (×3 OK), `groups/valid/`,
  + DB persist (vo_lines row, zadarma_doc_group_id, slot→file map). 2nd signing fix: Zadarma uses PHP RFC1738
  (`+` for spaces) not `%20` — patched in `_shared/zadarma.ts`. **GO-LIVE FLAGS:** (a) confirm correct Zadarma
  `document_type` enum for a Spanish geographic number (used passport/`certificate`/`receipt` as nearest fits);
  (b) doc-group address MUST be the customer's REAL in-zone registered address (test used a placeholder) — drives
  Zadarma `is_address_match` (the moat); (c) sandbox `groups/create/` returns id 0 & never verifies → real
  "verified=true" + validation-gated auto-order can only be confirmed in a controlled PRODUCTION test at go-live.
- ⬜ Next: order → route(set_sip_id) → activate vo_line + grant bundle minutes (gated on validation booleans;
  sandbox test via force flag). Then usage-metering webhook, Stripe overage sweep, corporate +line/+seat.

**Still pending before build:** Zadarma SANDBOX api key/secret; accountant sign-off (payment-time invoicing +
the NET sweep / IVA retention — founder says treat as signed-off, confirming in parallel).

## 12. Definitive pricing & limits (founder matrix 2026-06-24) + flags
| Item | Growth (Tier 2) | Premium (Tier 3) | Extra line | Extra seat |
|---|---|---|---|---|
| Retail (monthly, +IVA) | **€69 ⚠️ see flag** | €119 | €19/mo | €9/mo |
| Included outbound mins | 100 | 300 | uses parent bundle | shares parent bundle |
| Inbound | free unlimited | free unlimited | free | free |
| Overage block | €10+IVA → 250 mins | €10+IVA → 250 mins | same tripwire | same tripwire |
| Our wholesale | ~€2 line + ~€1.50 mins | ~€2 line + ~€4.50 mins | ~€2/mo | €0/mo |

**DB init constants (seed once Growth confirmed):** `tier_2_included_minutes=100`, `tier_3_included_minutes=300`,
`overage_recharge_price=10.00`, `overage_recharge_minutes=250`, `add_line_subscription_price=19.00`,
`add_seat_subscription_price=9.00`. Standard cheap-zone (Spain/EU landline) only; block costly routes.

**FLAGS to resolve before seeding:**
1. 🔴 **Growth price conflict:** matrix says €89, but the LIVE seeded recommendation is **€69** (Premium €119
   matches). Confirm €69 (keep) vs €89 (deliberate reprice) before any change — it flows into checkout.
2. 🟠 **App-fee mechanism by charge type:** €10 overage = one-off → `application_fee_amount` ✅. €19 line + €9
   seat = **monthly subscriptions → `application_fee_percent` (first invoice) + `application_fee_amount` on
   renewal invoices via webhook** (NEVER `application_fee_amount` in `subscription_data`). Per NOTE_FOR_ARCHITECT.
3. 🟠 **Telephony pricing is PLATFORM-level (Medacrii), not operator-adjustable** (it's swept to Medacrii).
   Store €10/€19/€9 + bundles in platform config, NOT the operator recommend/override tables.
- Note: "AI Executive" (Premium) promises AI call-handling that is a LATER build; at launch the tier delivers
  the geographic number + SIP/Zoiper line. Positioning only.

## 13. Phase-2 launch scope (founder-confirmed 2026-06-24) — dialer decoupled
For the **mid-July launch**, Phase 2 = **core provisioning + Stripe Connect sweep + tripwire overage ledger**,
with the **SIP-credentials display in the dashboard** as the day-one calling interface (clients use Zoiper etc.).
The **WebRTC in-browser dialer is decoupled** to a later phase (does NOT gate launch). Build sequence §11 stands;
the dialer is step 4 (post-launch). Founder has authorised building (accountant confirm in parallel); Zadarma
sandbox keys to be added when the provisioning build needs them.

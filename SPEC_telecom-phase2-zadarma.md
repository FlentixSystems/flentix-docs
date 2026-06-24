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

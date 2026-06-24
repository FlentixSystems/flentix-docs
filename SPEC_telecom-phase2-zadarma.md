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
   this spec must not preclude it (the SIP/PBX routing step is where it will later attach).

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
5. **Provisioning (NEW edge function, on admin activate OR auto after AI-check passes — TBD step 6):**
   a. `POST /v1/documents/groups/create/` → create a Zadarma document group.
   b. `POST /v1/documents/upload/` → upload the stored KYC docs to that group.
   c. `GET /v1/documents/groups/valid/<ID>/` → validate for the Spanish destination.
   d. `POST /v1/direct_numbers/order/` → order a geographic number in the hub's zone.
   e. `PUT /v1/direct_numbers/set_sip_id/` → route the number to the PBX/SIP (later: to the AI receptionist).
   f. On success: set `assigned_geographic_number_id`, store the Zadarma number + doc-group ids, flip the
      subscriber to `active`, send the activation email (number + portal link).
6. **Failure branch — Zadarma rejects the documents** (validate fails): set `kyc_status='rejected'`, notify the
   customer, reuse **"resend KYC link"** so they re-upload. If the order ultimately cannot be fulfilled →
   **credit note + Stripe refund**, subscriber → cancelled.
7. **Card-on-file (Phase-2 SetupIntent)** for FUTURE metered call-usage billing — keep as a `SetupIntent`
   (verification, NOT a second charge; consistent with single-charge Premium). Decide timing: capture at/after
   activation. Usage billing itself is future/out of scope.

## 3. Data
- Reuse `virtual_office_subscriber_compliance`: `assigned_geographic_number_id` (already exists) + add
  `zadarma_number` (text), `zadarma_doc_group_id` (text), `ai_doc_check` (jsonb: per-slot classification +
  confidence), `provisioning_status` (text: pending|docs_uploaded|validated|ordered|active|rejected|failed).
- No change to pricing tables.

## 4. Secrets / config
- `ZADARMA_API_KEY`, `ZADARMA_API_SECRET` (added by Grant; sandbox values first), `ZADARMA_USE_SANDBOX=true`.
- AI doc-check: the model/key for the vision classifier (e.g. Claude). 
- Per-hub fiscal identity (for the payment-time Factura) — go-live blocker.

## 5. Out of scope (separate work)
- **AI receptionist / conversational call handling** (future; attaches at the SIP routing step).
- Metered call-usage billing (future; the card-on-file is the hook).
- Live Zadarma keys (go-live).
- Non-Spain geographic provisioning (per-country, later).

## 6. Open items for the build
1. Confirm with the accountant: Factura at **payment** vs activation; credit-note/refund on failed verification.
2. Confirm whether provisioning auto-runs after the AI pre-check passes, or waits for an admin "Activate" click
   (recommend: auto-attempt Zadarma validate+order after AI pre-check passes; admin only intervenes on failure).
3. Confirm the vision-LLM choice + cost for the AI doc pre-check.
4. Map the hub's zone → Zadarma `DIRECTION_ID` for the availability pre-check + number order.
5. Reconcile the existing stub functions (`create-telecom-setup-intent`, `send-phase2-activation-link`,
   `confirm-telecom-activation`, `ActivatePhone`) — keep/repurpose vs replace.

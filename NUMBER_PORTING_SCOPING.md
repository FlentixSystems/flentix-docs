# Number Porting / "Bring Your Own Number" (BYON) — Scoping & Spec

**Audience:** Gemini (architect AI), Lovable build, and any human/AI joining this work.
**Date:** 2026-06-28. **Status:** Approved to build (Grant, 2026-06-28). PoC / sandbox.
**Decision:** Build it as a **Spanish BYON add-on to Tier 3**, reusing the registered
address, the Zadarma reseller flow, and telephony KYC already in place. The UK angle
is **parked** (see §6).

This doc is the single source of truth for the port-in feature. It complements
`CONSTITUTION.md` §7 (tiers) and §8 (Zadarma PBX) and `NOTE_FOR_ARCHITECT.md` (billing).

---

## 0. LOCKED DECISIONS (confirmed by Grant, 2026-06-28)

1. **Market = Spain.** BYON is a **Spanish** product: Spanish geographic numbers,
   21% IVA, Spanish operators. It is **not** a UK PSTN-switch-off play (§6).
2. **BYON is an entry path into Tier 3 (AI Executive Suite), not a new tier.** Instead
   of Flentix issuing a *new* Spanish DID, the customer **keeps and ports in their
   existing geographic number**; everything else about Tier 3 (AI receptionist, PBX,
   routing, voicemail-to-email, extensions) is unchanged.
3. **The registered virtual-office address is the compliance asset.** Spain requires a
   geographic number's holder to have an address in the **matching province/region**.
   The operator's registered VO address (already sold as Tier 1 "Business Footprint")
   satisfies that requirement and is what makes the port eligible.
4. **Donor-side problems are the CLIENT's, not ours.** If the losing provider blocks
   the port for an outstanding contract, debt, early-termination fee, mismatched
   account details, or a number not actually owned by the customer, Flentix's **only**
   action is to **relay** that the port is blocked and what the client must resolve.
   We do **not** absorb, chase, or pay anything on the donor side. (Grant, explicit.)
5. **Port-in support is handled by a remote worker or a trained AI assistant.** Because
   porting is semi-manual at Zadarma (a support request, not a one-click API), a person
   or a trained assistant owns the enquiry → LOA → status-chase loop (§4).
6. **Keep it simple; it can be dropped.** If the port-in branch starts to overcomplicate
   the checkout/PBX flow, it can be shelved without affecting the existing
   issue-a-new-DID path. It is strictly additive.

---

## 1. Why this works (regulatory basis)

- **Spain ties geographic numbers to a geographic address.** To hold or port a Spanish
  geographic number (e.g. Madrid `+34 91`, Pontevedra `+34 886`), the holder must show a
  current address in that number's region. **Zadarma enforces this**: a Spanish
  geographic DID only activates after the holder uploads company registration *or*
  passport/ID **plus** a current address (street, number, postal code) **in the number's
  country/region**.
- Flentix's operators already sell exactly that address. So the bundle —
  **registered geographic address + ported geographic number + AI PBX** — is a natural,
  compliant package rather than a workaround.
- **⚠️ Compliance caveat:** the address must be a **genuine registered** VO address, not a
  paper-only fiction. Presenting a purely nominal address solely to satisfy the number
  rule risks CNMC scrutiny. Yours are genuine VO addresses, which is the whole point —
  keep it that way. Flag for the accountant/regulatory brief.

---

## 2. How porting works at Zadarma (donor → Zadarma)

Porting at Zadarma ("MNP") is **free** but **semi-manual**:

1. **Fund the number for 12 months** (top up the balance for ≥12 months of the number's
   service) **or** attach an `Office` / `Corporation` price plan.
2. **Submit a port request** from the account to Zadarma support with an **invoice from
   the current (donor) provider**. The invoice must show **First name, Last name, Address,
   and the number being ported**. Some cases also require a **Letter of Authorisation (LOA)**.
3. Zadarma coordinates with the donor and connects the number. **Timeline:** at least a
   few working days; varies by country and number type (Spanish geographic timeline to be
   confirmed with Zadarma — see §7).

**API surface (dealer/reseller):** the dealer API supports acting on behalf of a user via
`user_id`, creating document groups, **uploading the required documents**, and connecting
numbers using that document group. Auth is `userKey:signature`; general rate limit
100 req/min. The **document upload and number-connect steps are API-able**; the **port
request itself is effectively a support ticket**. So treat the port as: *automate document
capture/upload, then open/track the port request*. (OpenAPI spec: `github.com/zadarma/openapi`.)

---

## 3. Where it slots into the existing build

BYON is a **branch on `CONSTITUTION.md` §8 step 7** ("Purchase number"). Steps 1–6
(register reseller user, verify, top up) and 8–11 (create PBX, extensions, IVR, webhooks)
are **unchanged**. Only step 7 forks:

```
§8 step 7  —  Provision the number
 ├── 7a. NEW DID (existing path):  buy a Spanish geographic DID via the Virtual Number API
 └── 7b. PORT-IN (BYON, new):
         1. Customer selects "I already have a number" at checkout.
         2. Collect: the number, donor provider, donor account number,
            a recent donor invoice (name/address/number), and a signed LOA.
         3. Fund the number for 12 months (or attach the plan) on the Zadarma side.
         4. Upload documents via the dealer API (document group, user_id).
         5. Open/track the Zadarma port request; surface status to the customer.
         6. On completion → number is live on the user's account → continue to
            §8 step 8 (Create PBX) exactly as for a new DID.
```

Everything downstream of provisioning (PBX, AI receptionist, routing, white-label) is
identical to the new-DID path. The customer never sees Zadarma (white-label preserved).

---

## 4. Support / operations model

- **Owner:** a remote worker **or** a trained AI assistant (Grant's call which, or both —
  AI first-line, human escalation).
- **Why a human/AI in the loop:** the port request is a Zadarma support interaction, and
  donor-side issues are unpredictable. This is a coordination job, not a pure code path.
- **Responsibilities:** collect/validate the LOA + donor invoice, submit & track the port,
  keep the customer informed, and **relay donor-side blockers** per §0.4.
- **Trainable:** the enquiry flow is bounded (eligibility → documents → submit → status →
  done/blocked). A trained assistant on this doc + an FAQ can run first-line comfortably.

---

## 5. Billing

Consistent with `NOTE_FOR_ARCHITECT.md` (Model 1, operator = MoR, application fee on NET):

- **Recurring:** the normal **Tier 3** subscription on the operator's connected account
  (net + 21% IVA), with Medacrii's fixed per-tier application fee — **no change**.
- **One-off port-in / migration fee:** covers the support labour and the 12-month
  pre-funding requirement. As a **one-off** charge this correctly uses
  `application_fee_amount` in `payment_intent_data` (the one-off case in
  `NOTE_FOR_ARCHITECT.md` §0.2) — **not** `subscription_data`.
- **Port itself is free at Zadarma**, so the fee is margin/labour + the pre-funding float,
  not a wholesale port cost.
- **PROVISIONAL figure:** port-in fee **TBD by Grant** (placeholder only; mark provisional
  exactly like the §0.4 tier-fee placeholders in `NOTE_FOR_ARCHITECT.md`).

---

## 6. UK — explicitly parked (why)

- The UK does **not** require a matching local address for a geographic number; Ofcom has
  moved the **opposite** way (allowing out-of-area `01`/`02` use). So the "match the
  address or lose the number" mechanic **does not apply to the UK**.
- The real UK event is the **PSTN/copper switch-off (31 Jan 2027)** — but that migration
  **preserves** the number via porting to VoIP; it's not an address problem, and it's a
  crowded reseller market with UK numbers + UK VAT (a different business from the Spanish
  stack). **Park unless deliberately chosen later.**
- **Do not** market a "UK October deadline" — it is unconfirmed and likely a conflation.

---

## 7. Open questions (confirm before go-live)

1. **Spanish geographic port timeline** at Zadarma (days/weeks?) — set customer expectations.
2. **LOA always required, or does the donor invoice suffice** for Spanish geographic ports?
3. **Exact dealer-API calls** for (a) document-group creation, (b) document upload, and
   (c) the port request — and which steps remain manual support tickets vs API. Verify
   against `github.com/zadarma/openapi` and live API docs.
4. **Pre-funding mechanics** — confirm whether the 12-month pre-fund or the `Office`/
   `Corporation` plan is the cleaner route for a ported number under the reseller account.
5. **Port-in fee figure** (§5) — Grant to set.

---

## 8. Paste-ready architect/Lovable brief (when ready to implement §3)

```
Add a PORT-IN ("Bring Your Own Number") branch to the existing Spanish Tier 3 flow.
It is STRICTLY ADDITIVE — do not change the existing "issue a new DID" path, the PBX
creation steps, billing model, or white-label behaviour.

CONTEXT: Spanish geographic numbers require the holder to have an address in the number's
region; our operator's registered VO address satisfies this. We reuse it. Porting at
Zadarma is free but semi-manual (fund 12 months or attach a plan, then a support request
with the donor invoice + LOA). The customer must NEVER see Zadarma.

1) Checkout (Tier 3): add a choice "Issue me a new number" (default, unchanged) vs
   "I already have a number (port it in)". For port-in, collect: number, donor provider,
   donor account number, donor invoice upload (must show name/address/number), and a
   signed LOA. Keep existing KYC (ID/passport + address proof).

2) Provisioning (Constitution §8 step 7): for port-in, do NOT call the new-DID purchase.
   Instead: ensure 12-month funding (or plan) on the reseller user; upload the documents
   via the dealer API (document group + user_id); open/track the Zadarma port request;
   store a port status (pending / submitted / blocked / completed) the customer can see.
   On "completed", continue to §8 step 8 (Create PBX) exactly as for a new DID.

3) Billing: keep the Tier 3 subscription unchanged. Add a ONE-OFF port-in fee as a
   separate one-off charge on the operator's connected account, using
   application_fee_amount in payment_intent_data (NEVER subscription_data). Fee value
   comes from config, marked PROVISIONAL.

4) Policy: if the port is blocked by the donor (debt, contract, mismatch, not owned),
   surface a clear "blocked — outstanding issue on your previous provider you must
   resolve" status. Flentix does not chase or pay donor-side costs.

5) Do NOT claim any UK applicability. This is Spain-only.

After implementing, STOP and let us test the port-in flow against Zadarma test/sandbox
before anything else.
```

---

**Sources:** Zadarma — free porting (MNP); Zadarma — documents to connect a number;
Zadarma — porting a number from another provider; Zadarma — API instructions for dealers
and partners; `github.com/zadarma/openapi`. UK context (parked): Ofcom geographic numbering
direction; UK PSTN switch-off 31 Jan 2027 (number preserved on port to VoIP).

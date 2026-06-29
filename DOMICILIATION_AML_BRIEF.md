# Brief — Domiciliation Agreement + AML/KYC (Premium geographic-number compliance)

**Audience:** Grant + the Spanish data-protection / AML lawyer + accountant; architect/engineering. **Owner:** Grant.
**Date:** 2026-06-28. **Status:** OPEN — **Premium-tier go-live blocker.** Not a build-now item; does NOT block the
AI-receptionist engine/console. Read with [SPEC_telecom-phase2-zadarma.md](SPEC_telecom-phase2-zadarma.md),
[NOTE_FOR_ARCHITECT.md](NOTE_FOR_ARCHITECT.md), [GO_LIVE_CHECKLIST.md](GO_LIVE_CHECKLIST.md),
[PRIVACY_POLICY_TEMPLATE.md](PRIVACY_POLICY_TEMPLATE.md).

## 1. The gap (founder-identified 2026-06-28)

The **Premium** tier includes a **local geographic phone number** (the moat). Zadarma's KYC for a geographic number
checks `is_address_match` — the documents must show the **number-holder has a real registered address in that
number's zone** (e.g. Granada).

A virtual-office client **does not have their own address in that zone** — they are using **the operator's (hub's)
virtual address**, which is the whole point of the product. They therefore have **no utility bill or independent
proof of address** for the zone. Their *only* valid proof is a **domiciliation / certificate-of-address agreement
issued by the operator**, confirming the client is entitled to use that registered address.

**What's built today:** the VO/KYC flow asks the client to **upload** `required_kyc_docs`
(`{passport, company_registry, proof_of_address}`) and we run an AI sanity pre-check, then submit to Zadarma. **We
do NOT generate the domiciliation document** — so a real client has nothing valid to put in the `proof_of_address`
slot for a geographic number. **Without this, the Premium local number cannot be delivered to real customers.**

## 2. What's needed — Part A: the domiciliation document generator

Auto-generate, **per hub + per client**, a **domiciliation agreement / certificate of address**:
- **Grantor:** the operator/hub (using the **per-hub legal identity** — legal name / NIF / registered address; this
  is the same data still missing per the per-hub fiscal-identity blocker, `NOTE_FOR_ARCHITECT` §, GO_LIVE §B).
- **Grantee:** the VO client (name / company / NIF).
- **Content:** grants the client the right to use the hub's in-zone registered address as their business/registered
  address; effective dates; the regulated-activity wording required in Spain.
- **Produced:** automatically at VO purchase (or KYC step), as a PDF, and **fed into the KYC pipeline as the
  client's `proof_of_address`** submitted to Zadarma.
- **Reuses:** the existing per-hub document/PDF machinery and per-hub identity threading.

## 3. What's needed — Part B: Flentix's own AML KYC / identity verification

Providing a registered/domiciliation address is itself a **regulated activity in Spain (Ley 10/2010, AML)**. So the
operator/Flentix has its **own obligation to verify the identity of the clients it domiciles** — *separate from*
Zadarma's number-holder check.

- **Today:** a lightweight Claude-vision doc pre-check (sanity gate only) + reliance on Zadarma's binding KYC.
- **Needed for AML:** a proper **identity-verification (IDV)** step on the client. Options:
  - **Stripe Identity** — recommended (already on Stripe; clean integration, document + selfie/liveness).
  - Jumio (what **Zadarma itself uses**), Sumsub, Onfido, Veriff — enterprise IDV alternatives.
- Plus the AML basics: record-keeping/retention (the privacy template already cites AML retention up to 10y),
  risk assessment, and any required filings.

## 4. Questions for the lawyer / accountant

1. **Domiciliation agreement:** exact required wording + clauses for a Spanish virtual-office domiciliation that
   (a) is legally valid and (b) satisfies a telecom carrier's geographic-number address check. Is operator-issued
   sufficient, or is a notarised/registered form needed?
2. Does the operator also need to provide **its own** proof of premises (lease/title/utility for the building)
   alongside the client's domiciliation certificate?
3. **AML scope:** as a domiciliation provider, what is the operator's KYC obligation on each client (IDV depth,
   beneficial-owner checks, ongoing monitoring, retention)? Does Medacrii (platform) or the operator (MoR) carry it?
4. Is **Stripe Identity** acceptable evidence for the Spanish AML obligation, or is a specific provider/standard
   required?
5. Interaction with the **per-hub fiscal identity** and the existing **booking/VO consent + privacy** work.

## 5. Scope / sequencing

- **Blocks:** Premium tier go-live with a **real** geographic number for a **real** client.
- **Does NOT block:** the AI voice receptionist (engine + console + metering), Tier 1/2 without a geographic
  number, or any sandbox/dogfood testing (Flentix's own number, where Medacrii is both provider and holder).
- **Build trigger:** after the lawyer confirms the document content + AML approach, and once per-hub legal identity
  lands. Then build the generator (Part A) + wire the IDV step (Part B) into the VO/KYC flow.

# Flentix — Virtual Office AML & Domiciliation Compliance Brief

**Status:** Working position pending formal Spanish legal advice
**Owner:** Grant (Medacrii Associates Ltd)
**Audience:** All Flentix chats, and specifically the master architect building the VO/PBX offering
**Last updated:** 29 June 2026
**Authority:** This document is the single source of truth for the compliance architecture of the premium (geographic-number / domiciliation) tier. Where it conflicts with anything in an individual chat, this document governs until superseded by formal legal advice, at which point it will be updated.

---

## 0. Why this document exists

Building the premium virtual-office tier (registered address + local geographic number) has surfaced a Spanish regulatory requirement that must be designed into the system, not bolted on later. Providing a business/registered address to a company on a professional basis is itself a **regulated AML activity** in Spain. This brief records the verified legal position, the decisions taken, and the build implications, so every chat works from the same picture.

**Important framing:** the compliance requirement is not a problem — it is the moat. The current Spanish virtual-office market is largely non-compliant (email applications, light or no KYC, no liveness checks, no beneficial-ownership records). As enforcement tightens, those operators are exposed. Flentix being built compliant from day one is the defensible advantage, and "compliance handled for you" is the core operator sales pitch.

---

## 1. The core legal finding

Under **Ley 10/2010, de 28 de abril** (prevención del blanqueo de capitales y de la financiación del terrorismo), **Article 2.1.o)**, providing — professionally, on behalf of third parties — a registered office or a commercial, postal or administrative address to a company or other legal entity is an activity that makes the provider a **"sujeto obligado"** (an AML-obligated entity).

Consequences that flow from this:

- The party providing the domiciliation falls into the same regulated category as company-formation agents and trust/company service providers (TCSPs).
- That category carries a **mandatory registration requirement** — registration as a company-service provider via the **Registro Mercantil** (Registro de Prestadores de Servicios a Sociedades), with an annual declaration.
- The full Ley 10/2010 compliance programme attaches (see §4).

**This is a licensing/sequencing gate, not a defect in the build.** The platform, provisioning flow, KYC capture, address network and sites all stand. What this adds is a registration step in front of the premium tier and a set of compliance features to build.

---

## 2. Entity & liability structure (working decision)

**Decision: the Spanish operator is the sole sujeto obligado (AML) and data controller (GDPR). Medacrii (UK) is the software provider and data processor.**

Rationale:
- The regulated act under 2.1.o) is *facilitating the address to the client*. The operator does that and is merchant of record. Medacrii supplies software, which is not itself a 2.1.o) activity.
- Note: "sujeto obligado" (AML status under Ley 10/2010) and "data controller" (GDPR role) are **two distinct legal concepts under different law**. We intend both to sit with the operator. This must be confirmed by the lawyer, not assumed.

**The substance-over-form risk (critical for the architect):**
Spanish regulators assess *who actually performs the regulated function*, not what the contracts label it. If Medacrii's platform makes the compliance decisions and the operator merely rubber-stamps an output it doesn't understand, a regulator could find Medacrii is performing the function — pulling AML liability (and possibly Spanish registration obligations) onto Medacrii.

The database schema / data location is **supporting evidence, not the determinant.** What protects the structure is:
1. The contract stack (master operator SaaS agreement + Data Processing Agreement) clearly allocating AML status and GDPR roles to the operator.
2. A **genuine, logged human decision** by the operator at the point of approval — they must be able to see, review and *reject* what they approve, and that decision must be recorded in the audit trail.

> **Build principle:** the operator's `[Approve KYC]` (and equivalent approval actions) must be a real gate, not a pre-ticked box. Every approval is a logged human act with the reviewing user, timestamp, and the data they saw. This audit trail is the evidence of "substance."

A third model (centralised compliance, Medacrii as the obligated entity) was considered and is **deferred** — see §6.

---

## 3. The operator-burden model — "compliance department in a box"

The strategic goal: make the operator's compliance load as light as a signature, because target operators (coworking spaces) won't take on heavy compliance work or fear. The way to do that legally is to split every obligation into **the doing (automate it) and the owning (operator signs/adopts/decides).**

Each Ley 10/2010 obligation sorts into one of three buckets:

### Bucket A — Fully automated (operator barely sees it)
- **Client identification & verification** — platform captures document, runs liveness, performs matching.
- **Beneficial-ownership collection** — structured declaration form; system records and stores.
- **PEP / sanctions / adverse-media screening** — automated API call to a screening provider at onboarding; **re-runs on a schedule for ongoing monitoring.**
- **Ongoing monitoring (mechanism)** — periodic automated re-screening and change detection.
- **10-year retention** — storage/archive architecture; runs itself once built.

### Bucket B — Platform generates, operator reviews & adopts (the "their letterhead, our work" model)
This is legal and standard practice (compliance consultancies operate this way), **provided the operator's adoption is a real, informed, logged decision.**
- **AML manual** — generated, operator-specific (legal name, NIF, address, named representative, service specifics), kept current. Operator reviews and formally adopts as theirs.
- **Annual declaration & reports** — pre-populated from data the platform already holds. Operator reviews and submits under their name.
- **Per-client risk assessment** — generated from screening data.
- **Business-wide risk assessment** — templated to their operation, operator adopts.

### Bucket C — Must be a real operator act (cannot be done for them; make it easy, not absent)
- **Designated SEPBLAC representative** — a real, named human at the operator accepting legal responsibility. Platform prepares the appointment paperwork and explains it; cannot invent or be the representative.
- **Suspicious-operation reporting (comunicación por indicio)** — platform can flag and pre-draft the report and make it one click, but the *decision* to report legally sits with the operator.
- **Staff training** — platform *provides* a built-in training module with logged completion (this satisfies the requirement); operator's staff must actually complete it.
- **External annual audit (if required)** — independent examiner must perform it (independence is the point). Platform keeps records audit-ready and can refer auditors; cannot be the auditor.
- **Internal control body** — for small operators may be light (potentially the representative performing the function); platform provides procedures and meeting templates.

**Net operator load:** name one responsible person (once), complete a training module, approve the documents the platform prepares, and make the call on rare suspicious cases. That is the light-touch product.

> **The line to respect:** "we prepared it, they genuinely adopted it" is permissible. "We are their compliance function and they rubber-stamp" is not — it collapses into the substance-over-form problem. The differentiator is a real, informed, logged approval. The lawyer must confirm *how much* preparation is permissible before the framing breaks.

---

## 4. Full Ley 10/2010 obligation list (reference)

For completeness, the obligations attaching to the sujeto obligado (the operator):
formal client identification and verification; beneficial-ownership identification for every corporate client; per-client risk assessment and a documented business-wide risk assessment; ongoing (continuous) monitoring; PEP and sanctions screening with adverse-media checks; a written, current AML manual; a designated representative before SEPBLAC; staff training; an internal control body; the duty to file suspicious-operation reports to SEPBLAC on own initiative; 10-year record retention; the annual declaration; potentially an external annual audit of AML measures; and (as GDPR controller of passport/KYC data) breach-notification, DPIA, and data-subject-rights obligations.

**Penalty context (why this is taken seriously):** serious infringements can carry fines from €60,000 up to the greater of 10% of annual turnover / double the operation's value, with director disqualification; the most serious tier reaches a minimum of €150,000 up to the greater of 10% of turnover, five times the benefit obtained, or €10,000,000, with public reprimand and director disqualification up to ten years. These attach to the obligated entity — which is why the operator-vs-Medacrii allocation matters.

---

## 5. Identity verification — the open technical question

**Established fact:** the telecoms carrier's KYC for provisioning the geographic number is satisfied by automated document scanning + a **live-photo liveness/selfie match** (confirmed in practice via Zadarma's compliance loop — live-photo matching, not streaming video).

**Open question for the lawyer:** the carrier's KYC and the operator's Ley 10/2010 file are **two separate audiences with two separate standards.** The carrier accepting live-photo matching does not by itself confirm that SEPBLAC's **non-face-to-face identification standard (identificación no presencial)** — historically the vídeo-identificación standard — is met for the domiciliation sujeto obligado. We need confirmation of whether the same document-scan-plus-live-photo flow satisfies the operator's AML file, or whether the domiciliation component mandates the SEPBLAC video-identification standard. **This is a potential build-blocker and must be answered precisely.**

---

## 6. Deferred decision — centralised compliance model

A model where Medacrii centralises the compliance function and becomes the obligated entity (making operator onboarding near-frictionless) was considered. It is **deferred, not adopted**, because:
- It converts Medacrii from a software company into a regulated compliance provider with concentrated liability across all operators/clients.
- Performing a Spain-regulated function as a UK entity likely requires a **Spanish establishment/subsidiary** registered as a sujeto obligado — a structural change.
- It is a deliberate phase-two move (with a Spanish entity, a compliance hire, insurance), not something to back into for launch.

A **hybrid** — platform centralises the *mechanics* (screening, capture, retention, monitoring) while the operator retains the *legal status and decision* — is the working model in §2–3 and is the preferred path. The lawyer is asked to advise on all three (operator-sole / Medacrii-central / hybrid) and confirm the hybrid holds.

---

## 7. Launch sequencing implication

- The **registration gate applies to the premium (domiciliation/geographic-number) tier only.**
- The **€35 address tier** and **€65 national-number (51x) tier** do not carry the same domiciliation-registration gate in the same way (to be confirmed) and can launch on schedule.
- **Plan:** launch the consumer site and the lower tiers in October as planned; bring the premium tier live once the operator-registration path and the identity-verification standard are confirmed. October is not at risk — it launches with two tiers, the third follows.

---

## 8. What the architect should build (compliance features)

Derived from the above. To be refined after legal confirmation, but safe to design toward now:

1. **KYC capture pipeline** — document scan + live-photo liveness + matching, result stored, retained 10 years. (Already largely built.)
2. **Beneficial-ownership declaration flow** — structured, stored, retained.
3. **Automated screening integration** — PEP/sanctions/adverse-media at onboarding + scheduled re-screening for ongoing monitoring; flags exceptions to the operator.
4. **Operator approval gate** — real, logged human approval on each client (reviewing user, timestamp, data viewed). This is the substance-over-form evidence. Must be a genuine gate, never auto-approved.
5. **Document generation engine** — operator-specific AML manual, risk assessments (per-client + business-wide), annual declaration/report, all merge-populated from held data, for operator review and adoption.
6. **Training module** — built-in, with logged completion per staff member.
7. **SEPBLAC representative record** — capture the named responsible person; generate appointment paperwork.
8. **Suspicious-activity flow** — system flags, pre-drafts the report, one-click for the operator; the decision and submission remain the operator's, logged.
9. **10-year retention architecture** — Supabase storage/archive designed for long-term retention and rapid retrieval for inspection.
10. **Audit-ready export** — ability to produce a complete compliance file per client/operator on demand (for SEPBLAC inspection or external audit).
11. **Operator's own proof-of-premises store** — hold the operator's lease/title + recent utility bill defensively (operator warrants right to domicile at the location).
12. **Controller/processor data architecture** — operator as data controller of client KYC data; Medacrii as processor under a DPA; design access and ownership to reflect this.

> Do **not** treat the database schema as the legal fix. It serves the legal structure (contracts + genuine human approval), it does not create it.

---

## 9. Open questions going to the lawyer (summary)

1. **Registration timeline** — how long does operator registration as a company-service provider take; can the service go live while pending or only once complete; what is the ongoing burden? *(Top priority — drives sequencing.)*
2. **Substance-over-form line** — how much KYC/AML can the platform automate before Medacrii is deemed to perform the regulated function; what must the operator demonstrably do by hand to remain sole sujeto obligado.
3. **Document generation** — to what extent can the platform generate the AML manual, risk assessments and reports for operator adoption; where is the line between permissible preparation and impermissibly performing the function; which acts must remain the operator's own.
4. **Identity standard** — does document-scan + live-photo satisfy the operator's Ley 10/2010 non-presential identification obligation, or is SEPBLAC vídeo-identificación mandated for the domiciliation component?
5. **Liability allocation** — confirm operator = sole sujeto obligado + data controller, Medacrii = processor; the three-model question (operator-sole / Medacrii-central / hybrid) including whether a Spanish entity is required for any.
6. **Documents to hold** — domiciliation template (with carrier-sharing clause), operator proof of premises, beneficial-ownership records, DPA, AML manual, etc.
7. **Anything else** a compliant Spanish domiciliation provider must have in place.

---

*Disclaimer: This brief is research-based and is the team's working position. It is not legal advice. The formal advice of the instructed Spanish law firm governs and will supersede this document where they differ.*

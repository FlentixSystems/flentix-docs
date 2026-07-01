# Advisory Register — Open Items for Lawyer / Accountant / Fiscal Sign-off

**Single source of truth** for every item awaiting external professional confirmation before or
around go-live. When something surfaces in a build or chat that needs a lawyer, accountant, or
asesor fiscal to confirm, add a row here. Detailed briefs are linked per row; this page is the
tracker over them.

**Status legend:** 🔴 open · 🟡 in progress / designed, needs confirmation · 🟢 confirmed (keep for record).

| # | Item | Area | Needs | Status | Detail |
|---|------|------|-------|--------|--------|
| 1 | **VeriFactu / Ley Antifraude (RD 1007/2023)** — the invoicing software itself must meet anti-fraud requirements (immutable, chained/hashed records, traceability, and AEAT reporting/QR). Applies to ALL Flentix-issued Facturas. | Fiscal software | Asesor fiscal + build to spec | 🔴 must build compliant — **own workstream** | (planned SPEC) |
| 2 | **Facturación por terceros** — Flentix issues Facturas *on behalf of* each operator (the Merchant of Record) under the operator's NIF; this delegation has its own formalities. | Fiscal | Asesor fiscal | 🔴 | this register |
| 3 | **Invoice numbering** — sequential series **per operator/NIF** (not one global sequence), shared format `FLX-‹HUB›-YYYY-NNNNNN`, with an **operator-chosen starting number N** captured at onboarding. Separate new series is permitted (RD 1619/2012). | Fiscal | Asesor fiscal (confirm compliant) | 🟡 design set | this register |
| 4 | **IVA model** — operator = MoR retains IVA; Medacrii platform fee swept on NET only (never IVA); 21% charged on everything the customer is billed. | VAT | Accountant (formal sign-off) | 🟡 founder-set | [`ACCOUNTANT_BRIEF.md`](ACCOUNTANT_BRIEF.md) |
| 5 | **Prepaid mail-credit deposit = *anticipo* WITH 21% IVA**; refund of unused balance via **Factura Rectificativa** (reverses base + IVA), reclaimed on **Modelo 303**. | VAT | Accountant (formal sign-off prudent) | 🟢 founder-confirmed | [`ACCOUNTANT_BRIEF.md`](ACCOUNTANT_BRIEF.md) |
| 6 | **AML / domiciliation** — Ley 10/2010 *sujeto obligado*, operator-registration gate, premium-tier KYC ("compliance department in a box"). | AML / legal | Lawyer | 🔴 | [`AML_COMPLIANCE_BRIEF.md`](AML_COMPLIANCE_BRIEF.md) |
| 7 | **Per-hub fiscal identity** — each operator's NIF / legal name / registered address printed on their Facturas. **Go-live blocker.** | Fiscal | Operator onboarding data | 🟡 panel built, data pending | [`GO_LIVE_CHECKLIST.md`](GO_LIVE_CHECKLIST.md) |
| 8 | **GDPR / data protection** — retention & handling of mail scan images and KYC documents; lawful basis; DPA with operators. | Data protection | Lawyer / DPO | 🔴 to assess | this register |
| 9 | **Cross-border / reverse-charge** — VAT treatment once operators exist outside ES / for B2B EU supplies. | VAT | Accountant | 🔴 when multi-country | [`ACCOUNTANT_BRIEF.md`](ACCOUNTANT_BRIEF.md) |

> Add a row the moment an item needs a professional's eye. Close items by moving them to 🟢 with a
> one-line note of what was confirmed and by whom (keep the row for the audit trail).

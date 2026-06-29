# Flentix — Governance & Docs

Single home for Flentix's **governance, brand, and architecture** documents.
Consolidated 2026-06-19 from the former separate `Constitution` and `Branding`
repos plus new finance/architecture notes.

> **App code lives elsewhere** — in the Lovable-synced application repo
> (`flentix-systems-sandbox-v2…`). This repo is **docs only**, kept separate so
> Lovable doesn't mirror it.

## Contents

| File | What it is | Audience |
|---|---|---|
| [`CONSTITUTION.md`](CONSTITUTION.md) | **Single source of truth** — v2.3 master blueprint. Paste as the first message in any new AI chat. | Everyone / all AIs |
| [`BRAND_GUIDELINES.md`](BRAND_GUIDELINES.md) | Full brand bible (Emerald/charcoal/geometric, logo, type, voice). | Design / marketing |
| [`NOTE_FOR_ARCHITECT.md`](NOTE_FOR_ARCHITECT.md) | Billing/fee/tax model, Stripe facts, and the agreed fixed-fee implementation. | Architect (Gemini) / engineering |
| [`ACCOUNTANT_BRIEF.md`](ACCOUNTANT_BRIEF.md) | VAT / reverse-charge brief and questions to confirm. | Accountant |
| [`GO_LIVE_CHECKLIST.md`](GO_LIVE_CHECKLIST.md) | Living go-live checklist (Stripe Connect, system/data, per-hub fiscal identity blocker, brand). | Engineering / Grant |
| [`SPEC_country-rollout-and-operator-pricing.md`](SPEC_country-rollout-and-operator-pricing.md) · [`PLAN_country-rollout-and-operator-pricing.md`](PLAN_country-rollout-and-operator-pricing.md) | Multi-country rollout + operator-set pricing spec/plan. | Architect / engineering |
| [`SPEC_telecom-phase2-zadarma.md`](SPEC_telecom-phase2-zadarma.md) | Premium-tier Zadarma number provisioning, usage/overage billing, corporate multi-line/seat. | Architect / engineering |
| [`SPEC_ai-voice-receptionist.md`](SPEC_ai-voice-receptionist.md) | AI voice receptionist (T2/T3 add-on): managed engine behind a swappable SIP boundary, dual-meter billing, locked go-to-market pricing. | Architect / engineering |
| [`AML_COMPLIANCE_BRIEF.md`](AML_COMPLIANCE_BRIEF.md) | **Authoritative** AML/domiciliation compliance architecture (Ley 10/2010 sujeto-obligado, operator-registration gate, "compliance department in a box" model, §8 build list). Governs the premium-tier compliance design until superseded by formal legal advice. | Everyone / architect / lawyer |
| [`DOMICILIATION_AML_BRIEF.md`](DOMICILIATION_AML_BRIEF.md) | Earlier, narrower note on the domiciliation document gap — **superseded by `AML_COMPLIANCE_BRIEF.md`** (kept for history). | Engineering |

## Entities (quick reference)

- **Medacrii Associates Ltd** — UK company; current legal/invoicing entity.
- **Flentix** — product brand; future entity, jurisdiction TBD.
- See `CONSTITUTION.md` §0 for the full picture.

## Governance

- **Owner / Gatekeeper:** Grant.
- **Versioning:** semantic (Constitution is v2.3.0).
- Update the Constitution's changelog when its content changes.

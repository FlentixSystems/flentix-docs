# 🔐 FLENTIX — BOOKING CHECKOUT: PRIVACY & NEWSLETTER CONSENT BRIEF

**Version:** 1.0.1
**Date:** 2026-06-23
**Owner / Gatekeeper:** Grant
**For:** Booking-system writer (Claude Code)
**Companion to:** `CONSTITUTION.md` (v2.2), `COMMUNITY_MODULE_BRIEF.md`

> **Scope.** This brief covers ONLY the **booking/checkout side** — the consent
> captured when a customer makes a booking. It is the GDPR foundation the Community
> module (and the future CRM) read from. **Additive to checkout only** — it must not
> alter payment, Stripe Connect, or booking logic.

> **One-writer rule (Constitution §5 / Grant).** Build only with Grant's express
> permission. This is a specification, not authorisation.

---

## 0. LEGAL MODEL (confirmed)

- Each **hub/operator = data Controller**; **Medacrii = Processor**.
- Data centralised in our **EU** Supabase (*confirm the project region is EU*).
- Consent must be **specific, granular, unbundled, and provable** (logged with
  timestamp + version).
- *Pending Grant's Spanish data-protection legal review: controllership, DPA
  template, privacy-policy template wording. The **mechanism** below can be built
  now; the **content** of the policy is confirmed separately.*

---

## 1. WHAT TO CAPTURE AT CHECKOUT — three distinct, separate elements

These are **three separate things**. They must **not** be bundled into one tick.
Bundling consent is one of the most common GDPR failures.

### 1. Terms & Conditions acceptance *(already exists)*
- Keep the current T&Cs tab/checkbox as is.
- **Required** to proceed.

### 2. Privacy Policy acceptance *(NEW — required)*
- A **separate, explicit** checkbox: *"I have read and agree to **[Hub]'s** Privacy
  Policy"* — with the hub's name and a link to the policy.
- **Required** to proceed. Lawful basis = contract / legitimate interest (needed to
  run the booking).
- **Must NOT be bundled** with the T&Cs tick or the marketing tick.
- The policy is **per-hub**: it names the specific hub as Controller and Medacrii as
  Processor. The link must resolve to the **current hub's** policy dynamically
  (keyed on `workspace_id`), not a hardcoded one. (Generic template, per-hub
  identity — Grant provides the template.)

### 3. Marketing / newsletter consent *(NEW — optional)*
- A **separate** checkbox: *"Keep me updated with news, events and offers from
  **[Hub]**."*
- **Unticked by default** (opt-in — never pre-checked).
- **Optional** — booking proceeds whether ticked or not.
- **Separate** from both the T&Cs and Privacy Policy acceptances.
- Wording should be **hub-configurable** (some hubs won't offer it, or will phrase
  it their way). A hub can disable this checkbox entirely.

---

## 1a. LAYERED INFORMATION AT POINT OF COLLECTION (AEPD requirement)

Spanish AEPD rules require **información por capas** (layered information) at the
exact moment data is collected — a bare "I agree to the privacy policy" checkbox is
**not** sufficient on its own.

**First layer — render a compact summary block at the checkout consent step**, next
to the consent controls, containing:
- **Controller:** [Hub legal name].
- **Purpose:** to manage your booking, membership and (where applicable) virtual-
  office services; and, with consent, to send marketing.
- **Legal basis:** contract; legal obligation; and consent (marketing / community
  visibility).
- **Recipients:** Medacrii (platform processor) and service providers.
- **Rights:** access, rectify, erase, object, withdraw consent — and the right to
  complain to the AEPD.
- **Link to the full policy** ("More information" → the Hub's full privacy policy =
  the second layer).

The summary block must be **visible at the point of collection**, not hidden behind
the link alone. The full policy (second layer) is reached via the link.

---

## 2. DATA TO RECORD (consent log — provability is a GDPR requirement)

For each element, store **what was consented to, true/false, when, and which
version** — tied to the user/booking and `workspace_id`. Suggested per-user consent
record:

| Field | Notes |
|---|---|
| `privacy_policy_accepted` | bool — required true to complete booking |
| `privacy_policy_version` | which version of the policy they accepted |
| `privacy_policy_accepted_at` | timestamp |
| `marketing_opt_in` | bool — defaults **false** |
| `marketing_opt_in_at` | timestamp (null if never opted in) |
| `terms_accepted` / `_at` | existing T&Cs, for completeness |
| `workspace_id` | the hub (controller) |
| `user_id` / booking ref | who |

> Exact schema is the writer's call — the **requirement** is: consent is **logged
> with timestamps + policy version**, not merely acted on. Marketing opt-in must be
> **revocable** later (a user can withdraw consent), so store it as mutable state,
> not a one-time event.

---

## 3. BOUNDARY & SEQUENCING

- **Writes consent data only.** Must not read from or alter booking/payment flow
  beyond adding these fields to the checkout step and persisting the consent record.
- The **Community module and CRM will later READ these consent records** (one-way) to
  know a person's privacy/marketing status. Do not build community/CRM coupling here
  — just persist the record cleanly.
- If the **repo legacy sweep** (TC Hub colours/refs) is still running, respect the
  one-writer rule — don't write simultaneously.

---

## 4. FLAGS FOR GRANT (not the writer to decide)

1. Confirm Supabase project region = **EU**.
2. Spanish data-protection legal review: controllership position, DPA template,
   and the **generic privacy-policy template wording** (the link target in §1.2).
3. Provide the privacy-policy template so the dynamic per-hub link has content to
   resolve to.

---

## 5. WRITER ACTIONS ON COMPLETION

- Update `README.md` / onboarding docs to reflect what was built.
- Confirm the three consent elements render correctly on **mobile** checkout
  (members book on phones).
- Confirm marketing opt-in defaults to **unticked** and is independently revocable.

---

### Changelog
- **v1.0.1 (2026-06-23):** Added §1a — AEPD layered-information requirement (first-
  layer summary block at point of collection + link to full policy as second layer).
- **v1.0.0 (2026-06-23):** Initial booking-side consent capture brief.

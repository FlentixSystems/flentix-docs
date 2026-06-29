# Spec — AI Voice Receptionist (Tier 2 & Tier 3 add-on)

**Audience:** Gemini (architect), Lovable build, future humans/AIs. **Owner:** Grant.
**Date:** 2026-06-27. **Status:** DESIGN — for founder review before any build. Sandbox-first; no live engine keys.
Read with [SPEC_telecom-phase2-zadarma.md](SPEC_telecom-phase2-zadarma.md),
[NOTE_FOR_ARCHITECT.md](NOTE_FOR_ARCHITECT.md),
[SPEC_country-rollout-and-operator-pricing.md](SPEC_country-rollout-and-operator-pricing.md),
[GO_LIVE_CHECKLIST.md](GO_LIVE_CHECKLIST.md).

> **One-line summary:** a white-label, per-hub AI voice receptionist that answers each Tier-2/3
> virtual-office client's inbound calls, screens + routes them, and (Tier 3) acts as a full concierge —
> billed as a paid add-on with its own metered AI-minute allowance and overage. The real-time voice loop
> runs on a **managed voice-AI platform behind a swappable SIP boundary**; Flentix/Supabase holds the
> per-client config, call logs, and meters (vendor-neutral). **No code touches the working Zadarma
> provisioning / billing engine** — this sits on top of it.

---

## 0. Context

- The VO core + telephony Phase 2 are built and sandbox-proven (Zadarma provisioning, dual-meter
  foundations `vo_account_minutes`/`vo_minute_ledger`, overage tripwire `vo-overage-topup`,
  usage webhook `zadarma-call-webhook`). The Premium tier's "AI Executive" line promised AI
  call-handling that was **positioning only** — this spec delivers it, and extends it to Tier 2.
- A **pre-built client console already exists** in `flentix-systems-sandbox-v2` at `/portal/voice`
  ("Voice & AI Receptionist"), built by a prior Claude architect (mock data). It is a strong shell;
  this spec **adapts the frontend** and defines the backend it sits on. It does **not** rewrite the
  working telephony/billing backend.
- The receptionist is a **standalone, feature-flagged, per-hub module** (same pattern as Community):
  `workspace_features(workspace_id, module='ai_receptionist', enabled)`.

## 1. Locked decisions (founder, 2026-06-21 → 2026-06-27)

1. **Channel = VOICE only.** No chat widget, no physical kiosk. (Those are explicitly out of scope.)
2. **Engine = Retell** (chosen 2026-06-28 after a head-to-head spike — see §5), behind a **SWAPPABLE SIP
   boundary.** Zadarma routes the client's DID over SIP to the platform's **EU SIP endpoint**; the
   platform runs the real-time STT→LLM→TTS loop. **Supabase edge functions CANNOT host the media loop**
   (they are short-lived request/response) — they only *configure* the per-tenant agent and *log* the
   post-call result. The number stays in Zadarma; the engine is the only swappable part.
3. **Launch scope = "Screen, route & basic FAQ"** on **T2 + T3**. **Concierge** (per-client knowledge
   base, booking lookups/actions, multi-step tasks, stronger model + premium voice) = **Tier 3 only**,
   **fast-follow** (added capability on the same foundation, not a rebuild).
4. **Pricing — all figures EX-IVA, always** (customer pays **+21%** on top; the **operator is Merchant
   of Record** and remits the IVA). Go-to-market, founder-locked:

   | Tier | List (ex-IVA) | Medacrii fee | AI mins (incl.) | Carrier mins (incl.) | AI concierge? |
   |---|---|---|---|---|---|
   | 1 (starter) | €35 | €5 | — | — | no |
   | 2 (growth) | €65 | €20 | **50** | 100 | no |
   | 3 (premium) | €129 | €60 | **100** | **300** | yes (fast-follow) |

   The **Medacrii fee is a fixed € amount, isolated from the IVA** (operator keeps the IVA). Achieved via
   the locked subscription mechanism (NOT `application_fee_amount` in `subscription_data`): **first invoice
   via `application_fee_percent` computed to equal the fixed fee; renewals via `application_fee_amount` set
   on the draft invoice through an `invoice.created` webhook** — see [NOTE_FOR_ARCHITECT.md](NOTE_FOR_ARCHITECT.md)
   §0.2 / §4. These figures **supersede** the PROVISIONAL placeholders in NOTE_FOR_ARCHITECT §0.4 and the
   older matrix in SPEC_telecom-phase2-zadarma §12.
5. **Overage = one unified prepaid €10-block tripwire** (reuses the existing engine), applied **per meter**:
   - **Carrier:** €10 → **250** min (existing).
   - **AI:** retail **€0.45/min** → €10 ≈ **22** min.
   - Auto-charged at zero **up to a customer-set BUDGET CAP**, then **route to voicemail** (AI) / **hold
     outbound** (carrier); inbound always stays free. **€0.50 operator kickback per top-up**; Medacrii
     sweeps the **NET only — never the IVA** (operator retains IVA + kickback). The €10 block is a one-off
     payment, so `application_fee_amount` IS valid here (unlike the subscription).
6. **Rollover:** **included** allowance **resets monthly** (protects the over-provisioning margin buffer);
   **purchased top-ups roll over** (12-month expiry). The meter therefore tracks **two buckets per meter**
   (included = resets, purchased = rolls over). ⚠️ Verify the existing monthly reset clears **only** the
   included bucket (see §12).
7. **Auto-top-up = per-customer toggle** (default **ON, capped**) + **low-balance notification**. The
   **budget cap is the customer's consent ceiling** — they are never charged beyond the cap they set at
   signup. "Running out" is graceful: **AI → voicemail; carrier → outbound held (inbound free)**.
8. **Flexibility model:**
   - **Fixed (platform constants, Medacrii-admin-overridable only, NOT operator-editable):** the Medacrii
     fee per tier **and** the AI/carrier minute allowances (those minutes spend Medacrii's money).
   - **Flexible (operator):** retail price **upward only**, with a system-enforced **floor**; plus physical
     inclusions (desk/office/meeting-room hours), which are the operator's own cost and never touch margin.
9. **Account architecture = ONE shared Zadarma balance + ONE voice-platform account.** Per-customer
   balances are a **virtual ledger in Supabase** (uniform for carrier + AI). **No per-number Zadarma
   accounts** (Zadarma's dealer API supports them, but AI minutes have no Zadarma equivalent, so per-number
   accounts would split the system). The carrier cut is enforced via Zadarma's **per-extension
   restriction/limit** when a customer's Supabase balance hits zero; the AI cut is at the platform (cap →
   voicemail). **Auto-replenish** keeps the two shared pots funded from aggregate revenue (low-balance alert
   + auto-top-up) so the shared pot never dries and customers stay logically isolated.
10. **Defaults:** AI answers **all** inbound calls (virtual office — no human desk); launch languages
    **Spanish + English** (auto-detect); message delivery = **email transcript** (SMS optional, later).

## 2. Architecture & call flow

```
Caller → client's Zadarma DID → (SIP route) → managed voice-AI engine (EU endpoint)
            │                                        │  runs STT → LLM → TTS in real time
            │                                        ├─ Forward to member mobile/extension (carrier leg; AI meter STOPS)
            │                                        ├─ Take a message → email transcript
            │                                        └─ Answer FAQ / (T3) concierge
            └─ engine timeout/failure ⇒ Zadarma voicemail → email (zero-downtime fallback)

Supabase (vendor-neutral): per-client agent config · call logs/transcripts · dual meters (carrier + AI)
```

- **Inbound:** the client's DID is SIP-routed (`set_sip_id` / external-SIP route) to the platform's EU SIP
  URI. The platform answers, screens, and on transfer hands the call to the forwarding number (carrier leg).
- **Config push:** when the client saves the console, an edge function pushes the agent config (greeting /
  script / rules / hours / language / voice) to the platform via its API (creates/updates the per-tenant
  assistant). The engine is addressed **only** through this adapter, so swapping platforms = re-implement
  one adapter + re-point the SIP route.
- **Post-call:** the platform fires a post-call webhook → edge function logs to `vo_ai_call_logs`,
  decrements the AI meter, and triggers the tripwire if the balance crosses zero.

## 3. Client console (adapt existing `/portal/voice`)

**One tier-aware console** (NOT two pages). Reads the hub's tier + the `ai_receptionist` flag.

- **Keep (already good):** plain-language operating-script editor with insert-variable chips
  (`{client_name}` `{forwarding_number}` `{office_hours}` `{detected_language}`); incoming call logs;
  voicemail-delivery email; business-hours + after-hours→voicemail; "AI Assistant First" inbound handling;
  Dual-Language EN/ES auto-detect; up-to-3 forwarding numbers/extensions.
- **Change:**
  - **Overage card** → relabel to the §5 model: "billed at **€0.45/min** up to your **budget cap**, charged
    in **€10 blocks**, then routed to voicemail." Cap presets (€0 / €50 / €100 / €250 / custom; €0 = stop at
    allowance) + auto-top-up toggle.
  - **Tier-awareness:** show **50** AI min for T2 / **100** for T3; reveal **concierge controls** (knowledge
    base, booking actions) **only on T3**, gated until the concierge fast-follow ships.
  - **Forwarded-call minutes count against the CARRIER meter, not AI** (reflected in logs/usage).
- **Brand:** canonical **teal-primary** tokens, **zero hardcoded colours**, per-hub `HubThemeProvider`
  (matches the design-system discipline used across the app).

## 4. Data model (sandbox-first)

- **`vo_ai_profiles`** — one row per workspace + line: `workspace_id`, `line_id`, `company_name`,
  `greeting`, `business_description`, `hours` (jsonb), `languages` (text[]), `voice`, `routing_rules`
  (jsonb: forward targets, message rules, voicemail fallback), `script`, `platform_agent_id`, `status`.
  RLS mirrors `virtual_office_subscriber_compliance` (service-role writes; hub-scoped reads).
- **`vo_ai_call_logs`** — append-only: `workspace_id`, `line_id`, `caller`, `started_at`, `duration_sec`,
  `outcome` (`answered_ai` | `forwarded` | `voicemail` | `failed`), `transcript_ref`, `ai_minutes`,
  `platform_call_id` (unique → idempotent logging).
- **Metering (extend, don't replace):** add an **AI meter** alongside carrier in the existing
  `vo_account_minutes` design, each split into **two buckets — `included` (resets monthly) and `purchased`
  (rolls over, 12-mo expiry)**. `vo_minute_ledger` gains `meter` (`carrier`|`ai`), `bucket`
  (`included`|`purchased`), and `grant_type`; append-only + idempotent (existing pattern).
- **Platform config — `ai_receptionist_config`** (platform-level, Medacrii-admin only): per-tier fee
  (5/20/60 €), allowances (50/100 AI, 100/300 carrier), AI overage rate (€0.45/min), €10 block sizes,
  €0.50 kickback. **Fixed; not in the operator recommend/override tables.**
- **Consent/compliance:** reuse the `consent_records`/`record-consent` pattern to log AI-disclosure +
  call-recording consent (see §7).

## 5. Engine integration — **Retell** (chosen 2026-06-28)

- **Decision:** **Retell** is the launch engine. Picked over Vapi for: consistent out-of-the-box latency
  (~600–800ms vs Vapi's config-dependent 700–1,500ms); **native SIP trunking compatible with Zadarma**
  incl. transfer-to-mobile (Vapi's warm transfer is Twilio-only); built-in **knowledge-base retrieval** (the
  T3 concierge); a **published rate card** (lockable economics); SOC2 + GDPR + PII-redaction on every plan.
  It fits a Lovable build + non-expert team far better than Vapi's build-it-yourself flexibility. **Not a
  one-way door** — the swappable SIP boundary makes a later switch a re-point, not a rebuild.
- **Adapter pattern:** a single `ai-engine` edge module wraps the **Retell** API (create/update agent,
  resolve SIP routing, parse post-call webhook). Everything else references the adapter, never the vendor —
  this *is* the swappable boundary.
- **Build-test before wiring (§10.1):** confirm end-to-end on Retell — (a) Zadarma DID → Retell EU SIP route
  + a live test call; (b) per-tenant agent config via API; (c) post-call webhook with transcript + duration;
  (d) real all-in €/min at the chosen voice + model (validate the €0.12 launch / €0.23 concierge + €0.45/min
  overage assumptions).
- **EU/residency caveat:** Retell processes in the US under DPA + SCCs (no full EU residency — same as Vapi);
  route call media via the EU SIP carrier (Zadarma EU) to keep media in-region. Legal-gate item (§7), not a
  blocker.
- **Fallback:** platform timeout/error → Zadarma voicemail→email (configured at the Zadarma route level so
  it works even if our backend is down).

## 6. Billing & metering

- **Subscription split (the tier fee):** fixed € fee per tier, isolated from IVA, via the **locked
  mechanism** in NOTE_FOR_ARCHITECT §0.2/§4 (`application_fee_percent` on invoice #1 = fee/gross; renewals
  set `application_fee_amount` on the draft invoice via `invoice.created`). Record actuals from the real
  finalised invoice. **Do NOT** put `application_fee_amount` in `subscription_data`.
- **Overage (one-off €10 block):** reuse `vo-overage-topup` (off-session PI on the operator's connected
  account, `application_fee_amount` valid here, IVA from `countries_config`, NET sweep, €0.50 kickback).
  AI block grants ~22 AI min @ €0.45; carrier block grants 250 min. Auto-fire on the needs_topup
  false→true flip, **bounded by the customer's budget cap**; beyond cap → voicemail (AI) / hold (carrier).
- **Rollover buckets** (§1.6) + **auto-top-up toggle/cap** (§1.7).
- **Margins (planning, prudent worst-case — foreign card, allowance maxed):**

  | | Tier 2 (fee €20) | Tier 3 (fee €60) |
  |---|---|---|
  | Margin, maxed, launch AI | ~€6.3 (32%) | ~€34.8 (58%) |
  | Margin, maxed, concierge AI | ~€0.8 (4%) — *concierge gated to T3* | ~€23.8 (40%) |
  | Margin, typical (~30% usage) | ~€10–12 (50–60%) | ~€43 (72%) |

  Per **€10 top-up block** (operator keeps €0.50 + the €2.10 IVA passes through; Medacrii gets €9.50, less
  Stripe ~€0.60 and minute cost): **AI launch ~€6.25 (62%)**, **AI concierge ~€3.85 (38%)**,
  **carrier ~€5.15 (51%)**.

## 7. Compliance (GDPR — lawyer sign-off, same pass as booking/VO consent)

- **AI disclosure** at answer ("you're speaking to a virtual assistant").
- **Call recording / transcription consent** per Spanish law; log via the `consent_records` pattern.
- **EU voice endpoint** (data residency, Frankfurt-aligned `aws-1-eu-central-1`).
- **DPA** with the chosen voice platform (hub = Controller, Medacrii = Processor, platform = sub-processor).
- **Retention policy** for transcripts/recordings (define + enforce).

## 8. Safeguards

- **Customer-side:** collect-before-spend tripwire (charge precedes minute grant); budget cap; cut
  enforcement (Zadarma per-extension limit for carrier; AI platform cap → voicemail).
- **Platform-side:** the two shared pots (Zadarma balance, voice-platform account) are **auto-replenished**
  with **low-balance alerts**, so the shared pot **never dries** — one customer running out can never affect
  another. This closes the "manual Zadarma top-up" + "no per-SIP outbound-disable" gaps flagged in the
  telecom spec.

## 9. Flexibility (fixed vs operator) — see §1.8. Single source of truth for fee + allowances is
`ai_receptionist_config` (platform); operator price override lives in the existing VO override tables with
an enforced floor.

## 10. Build sequence (each step = founder per-action permission; one-writer-to-Lovable rule)

1. **Build-test** — confirm Retell end-to-end (§5): Zadarma DID → Retell EU SIP → live test call, agent-via-API, post-call webhook, real €/min. (Engine already chosen; this validates, doesn't re-select.)
2. **Data model** — `vo_ai_profiles`, AI meter buckets, `vo_ai_call_logs`, `ai_receptionist_config`,
   the `ai_receptionist` feature flag (sandbox).
3. **Console adaptation** — tier-aware, overage relabel, forwarded-minute split (frontend only).
4. **Engine wiring** — adapter; config push on save; SIP route; post-call logging + AI metering.
5. **AI overage + rollover** — extend the tripwire to the AI meter; two-bucket rollover; auto-top-up
   toggle/cap.
6. **Safeguards** — shared-pot auto-replenish + low-balance alerts; Zadarma per-extension cut enforcement.
7. **Compliance** — disclosure + consent + DPA + retention (go-live gate).
8. **Concierge (T3)** — knowledge base + booking actions + stronger model/voice (fast-follow).

## 11. Out of scope (separate work)

- Chat widget / kiosk channels. Buying **additional lines/seats** (telecom spec §10 — separate chat).
  The **WebRTC in-browser dialer** (telecom spec §11 step 4 — later). The **concierge build** itself
  (fast-follow, §10.8).

## 12. Open items / go-live gates

1. ✅ **Engine chosen: Retell** (2026-06-28, §5). Remaining: the end-to-end build-test (§10.1).
2. **Legal sign-off:** AI disclosure + recording consent wording, DPA with the chosen platform, retention.
3. **Accountant:** confirm the tier fee mechanism (NOTE_FOR_ARCHITECT §4) for the new figures + the AI
   overage routing (NET sweep / operator IVA / €0.50 kickback).
4. **Confirm Zadarma per-extension outbound-limit / restriction API** is sufficient to enforce the carrier
   cut (closes the no-per-SIP-disable gap).
5. **Auto-replenish mechanism** for the two shared pots (Zadarma balance + voice-platform account).
6. **Rollover correctness:** confirm/repair the existing monthly reset so it clears **only** the `included`
   bucket, never `purchased`.
7. **Voice-platform real rates** — validate €0.12 launch / €0.23 concierge planning assumptions against the
   chosen platform + selected voice/model; re-confirm the €0.45/min AI overage margin holds.
8. **Add these gates to** [GO_LIVE_CHECKLIST.md](GO_LIVE_CHECKLIST.md).

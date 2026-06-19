# Accountant Brief — VAT / Reverse Charge & Platform Fee Treatment

**Status:** Proof of concept / sandbox. One operator client. **Not yet on live
Stripe keys.** This brief is to confirm the correct tax treatment *before* we go
live.

**Prepared:** 2026-06-19. Please treat the tax positions below as *proposed and
needing your confirmation* — they are written by the founder + an AI assistant,
not by a tax adviser.

---

## 1. The entities

| Name | What it is |
|---|---|
| **Medacrii Associates Ltd** | UK limited company. The **current legal, contracting and invoicing entity** for the platform during the proof of concept. It holds (or will hold) the Stripe **platform** account and issues the platform / PBX software-fee invoices to operator clients. |
| **Flentix (Systems)** | The **product/system brand**. Intended to become its own legal entity **after** the proof of concept; **jurisdiction not yet decided**. It is **not** a legal entity today — where docs or code say "Flentix Systems" as if it were the company, read "Medacrii Associates Ltd". |
| **Operators** ("hubs") | Medacrii's **clients** — e.g. Spanish co-working / virtual-office businesses. VAT-registered businesses in Spain. Each has its own Stripe (Connect) account. |
| **End-members** | The **operators' own customers** (individuals / businesses) who buy virtual-office and phone services from an operator. |

---

## 2. The money flows

**Flow 1 — End-member pays the operator (with Spanish IVA).**
The end-member pays the **operator's own Stripe account**: net price + **21%
Spanish IVA**. The IVA belongs to the operator; **the operator collects and
remits** Spanish IVA. Medacrii/Flentix does **not** collect or remit this IVA.

> Example: €100 net plan → end-member is charged €121 (€100 + €21 IVA) on the
> operator's account.

**Flow 2 — Medacrii's platform / PBX fee (charged to the operator).**
Medacrii charges the operator a **fixed monthly fee per product tier** (the cost
of the PBX/software). Mechanically this is taken via Stripe Connect as an
"application fee" deducted from the operator's transaction balance; **economically
it is Medacrii (UK) charging the operator (Spain) for software / PBX services.**

> Indicative fees per tier (to be finalised): Tier 1 €2, Tier 2 €30, Tier 3 €50
> per month. **Charged on the NET amount, never on the IVA portion.**

**Flow 3 — Line-usage costs (telephony) — future.**
Variable call/line usage will be **passed through to the operator at 100% of cost
(no margin)** as metered charges. Not yet built.

---

## 3. Proposed VAT treatment — please confirm

**A. Medacrii (UK) → operator (Spain): platform / PBX / usage fees.**
We believe these are **B2B cross-border supplies** where the **place of supply is
Spain**, so Medacrii does **not** charge UK VAT and the Spanish operator accounts
for the VAT under the **reverse charge**. Medacrii's invoices to operators would
state that the reverse charge applies and show the operator's Spanish VAT number.

**B. Operator → end-member: 21% Spanish IVA.**
The operator's responsibility — collected and remitted by the operator in Spain.
Nothing for Medacrii to collect here.

### Questions for you

1. Is the **reverse-charge** treatment correct for **each** fee type separately —
   (a) the software/platform fee, (b) the **PBX / telecom** fee, (c) the **usage
   pass-through**? (Telecom services can have special place-of-supply rules.)
2. What exact **invoice wording and data** must Medacrii's operator invoices carry
   (reverse-charge statement, the operator's VAT number, our details, etc.)?
3. Does Medacrii need any **Spanish or EU VAT registration** (e.g. OSS/IOSS) given
   the telecom/digital nature — particularly if any element is ever supplied
   **B2C** rather than B2B?
4. Should the **Stripe platform account** be registered in **Medacrii's** name, and
   how should the **application-fee income** be recognised in Medacrii's books
   (gross vs net, FX from EUR)?
5. Any implications we should plan for when the future **Flentix legal entity** is
   formed and the contracts/Stripe account migrate to it (and the
   **jurisdiction** choice)?

---

## 4. Important correction (so the tech doesn't mislead the books)

An AI design note previously claimed that **"Stripe Connect automatically generates
zero-rated reverse-charge VAT invoices from the platform to the connected
accounts."** **This is not true.** Stripe does **not** issue VAT invoices for the
platform/application fee and does **not** determine reverse-charge treatment.
**Medacrii must issue its own VAT-compliant invoices** to the operators for these
fees. Please make sure the bookkeeping process produces those invoices
independently of Stripe.

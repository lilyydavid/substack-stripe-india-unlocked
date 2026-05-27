# Substack + Stripe India: Unblocking Creator Payouts

Indian creators on Substack cannot receive subscription revenue. Stripe India moved to an invite-only merchant model in 2023, in response to RBI regulatory changes. Substack routes all payments exclusively through Stripe. When Stripe restricted access, paid subscriptions for Indian creators stopped.

This repo contains a concrete proposal to fix that, and working artifacts that Substack or Stripe can use directly.

---

## The regulatory context

Three RBI mandates changed the cost of onboarding individual creators at scale:

- **FIRA tracking** — every inward foreign remittance requires a Foreign Inward Remittance Advice document matched to the underlying contract
- **180-day realisation cap** — forex income from digital service exports must be realised within 180 days
- **EDPMS compliance** — Payment Aggregators must log every export transaction in the RBI's Electronic Data Processing and Monitoring System

Stripe India's per-creator manual review costs roughly $45. At that unit cost, onboarding individual newsletter writers is not viable.

---

## The proposal

Substack acts as Stripe's front-end screening layer. Substack introduces creators to Stripe only after they have been verified as legitimate digital exporters. Stripe audits Substack's process once, not each creator individually.



**Three components:**

| | Owner | What it does |
|---|---|---|
| Creator intake form | Substack | Captures PAN, IFSC, bank account, RBI purpose code with validation |
| Signed platform telemetry | Substack | Cryptographically signed bundle: account age, subscriber count, open rate, zero chargebacks |
| Endorsement API | Stripe | One optional field on `POST /v1/accounts` routes the application to a 48-hour fast-track queue |

The first two are entirely within Substack's control. The third requires a single API change from Stripe.

![Project Screenshot](https://github.com/lilyydavid/substack-stripe-india-unlocked/blob/9a1aaf9e2e2ac509626fc87185e3931c11c7c2cc/docs/Creator%20KYC%20Endorsement.png)

---

## What's in this repo

| File | What it is |
|------|------------|
| [index.html](index.html) | Technical brief — the proposal at a glance |
| [overview.html](overview.html) | Full context — regulatory background, creator GMV data, incentive alignment |
| [demo.html](demo.html) | Interactive mockup of the creator KYC intake flow, all 10 states |
| [demo-all.html](demo-all.html) | All 10 screens side by side |
| [api-design.md](api-design.md) | Full API spec — request/response, signing scheme, field validation |
| [risk-analysis.md](risk-analysis.md) | DPDPA liability, PAN hash question, telemetry abuse vectors, mitigations |
| [batch-endorsement-proposal.md](batch-endorsement-proposal.md) | Quarterly batch model, economics, platform agreement terms |

---

## Who is losing money right now

Because Substack does not publish subscriber data, these numbers are estimated from public signals:

| Newsletter | Niche | Estimated subscribers | Suppressed GMV/year |
|---|---|---|---|
| Tigerfeathers | Indian tech, startups, Web3 | 10,000-20,000+ | $12,000-60,000 |
| The India Notes | Consumer psychology, product design | 15,000-25,000+ | $18,000-75,000 |
| Filter Coffee / Market Bites | Business and macro | 25,000-40,000+ | $30,000-120,000 |
| The CMO Journal | Marketing, SaaS growth | 5,000-12,000+ | $6,000-36,000 |
| The Daily Brief / The Chatter | Markets, corporate governance | 50,000+ | $60,000-180,000 |

Substack's global benchmark: 2-5% of free subscribers convert to $5/month paid. For Indian creators today, actual Stripe GMV is close to zero.

---

## What creators are doing instead

- Embedding Razorpay and Instamowo payment buttons to route subscribers off Substack
- Using free Substack content as a lead funnel for consulting and courses hosted elsewhere
- Running sponsorships and native ads instead of reader subscriptions

Each month this continues, more Indian creators build their paid operations permanently off-platform.

---

## The open questions

**For Stripe India:** Will you accept a platform-attested SHA-256 PAN hash, or do your NSDL verification obligations require the raw value? The answer determines whether Substack needs an India legal entity.

**For Stripe Connect:** Is there an existing partner programme tier that already provides expedited review for high-trust platforms? If so, the API change may not be needed at all.

**For Substack:** Does a platform agreement with capped joint liability for endorsed creators fit your current legal framework?

---

## If this works

The same framework applies to Indonesia (Bank Indonesia PA framework) and Latin America (BCRA, Banco Central do Brasil, CNBV). One endorsement programme, three central bank regimes.

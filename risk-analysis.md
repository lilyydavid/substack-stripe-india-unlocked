# Risk Analysis — Platform-Delegated KYC & Telemetry Intake

**Scope**: KYC Intake (#101), Telemetry Signing (#102), Stripe Connect Bridge (#103)  
**Framing**: Neutral analysis for Substack Product and Stripe India BD

---

## Summary

This proposal is the most tractable entry point for unblocking Indian creator payouts.
It does not require new financial licences, does not pool creator funds, and does
not touch cross-border settlement mechanics. Its only ask of Stripe is a new
optional field on an existing endpoint. That narrowness is also its chief risk:
the entire proposal's value depends on Stripe choosing to build and maintain the
endorsement programme.

---

## Risk Register

### R1 — Stripe does not implement `platform_endorsement`
**Severity**: Critical  
**Likelihood**: High (no public signal that Stripe India plans this)

The endorsement bridge (#103) is a proposed API extension that Stripe has not
committed to. Without it, the telemetry bundle (#102) has no delivery mechanism.
The KYC intake form (#101) remains useful for Substack's own onboarding records,
but the fast-track review pathway closes entirely.

**Mitigation A**: Stripe Connect Partner Programme. Substack may already qualify
for an enterprise partner tier that provides a named compliance contact and
expedited manual review. This achieves the same outcome without a new API
primitive — Substack sends the bundle as a support ticket attachment rather than
an API field.

**Mitigation B**: Ship #101 and #102 without #103 as a first release. The signed
telemetry bundle has standalone value: it is admissible as a platform
attestation in any manual Stripe review, and it is the prerequisite for any
subsequent payout infrastructure regardless of whether #103 ships.

---

### R2 — PAN data handling creates PDPB/DPDPA liability for Substack
**Severity**: High  
**Likelihood**: Certain (PAN is sensitive personal data under India's DPDPA 2023)

Substack collecting and storing Indian creators' PAN numbers triggers obligations
under India's Digital Personal Data Protection Act 2023: purpose limitation,
storage minimisation, breach notification within 72 hours, and appointment of
a Data Protection Officer if above the threshold notification volume.

Substack currently has no India legal entity and processes data under US/EU
frameworks. Operating this pipeline at scale requires either:
- A data processing agreement with an India-domiciled sub-processor that holds
  the raw PAN, with Substack receiving only the hash; or
- A formal DPDPA compliance programme for India, including DPO designation.

**The `pan_hash` architecture in #102 partially addresses this**: raw PAN is
hashed before any transmission to Stripe. But Substack still holds the raw
PAN internally for the penny-drop verification and for future RBI reporting
obligations. That storage must be DPDPA-compliant regardless of what is sent
externally.

---

### R3 — Stripe India will not accept `pan_hash`; requires raw PAN submission
**Severity**: Medium  
**Likelihood**: Plausible

Stripe India's own RBI compliance may require that the raw PAN be submitted
directly to Stripe (not a hash) because Stripe must verify the PAN against
NSDL records as part of their own KYC obligations under the PA/PG framework.

If so, the `pan_hash` design in #102 is insufficient and Stripe would need
the raw PAN — which reintroduces the DPDPA liability above and requires a
data processing agreement between Substack and Stripe for India.

**Mitigation**: Confirm with Stripe India's compliance team whether they accept
a platform-attested PAN hash or require the raw value. Do not assume the hash
is sufficient before building the pipeline.

---

### R4 — Telemetry metrics are gameable if creator IDs are known
**Severity**: Medium  
**Likelihood**: Low for individual creators; higher if a coordinated abuse pattern emerges

The signed bundle attests to `free_subscriber_count`, `avg_open_rate_pct`,
and `paid_subscriber_pledges`. These are server-side metrics that a creator
cannot directly falsify — but a coordinated bot network could inflate them
(fake subscribers, automated opens).

The signing scheme (#102) guarantees *Substack attests to what Substack
measures*, not that the underlying measurements are abuse-free. Stripe would
be accepting Substack's measurement quality as a trust proxy.

**Mitigation**: Add an `abuse_signal_score` field to the bundle (internally
computed from bot-detection signals) and define a threshold below which
Substack will not issue a bundle at all. This makes the trust model explicit
to Stripe rather than leaving it implicit.

---

### R5 — IFSC master list maintenance
**Severity**: Low  
**Likelihood**: Certain (IFSC codes are updated as banks merge and reissue)

The #101 intake form validates IFSC against the RBI master list. This list
changes: bank mergers (e.g., Lakshmi Vilas Bank into DBS India) retire IFSC
codes, and new branches are added quarterly. A stale local copy will reject
valid IFSCs or accept retired ones.

**Mitigation**: Do not bundle a static IFSC list. Fetch from RBI's official
IFSC API (`https://www.rbi.org.in/scripts/IFSCMIPSQuery.aspx`) or a
maintained mirror (Razorpay's open IFSC dataset is widely used). Cache with
a 7-day TTL and fallback to format-only validation on cache miss rather than
blocking submission.

---

### R6 — Penny-drop verification delays creator activation
**Severity**: Low  
**Likelihood**: Certain

The ₹1 test deposit to verify a creator's bank account can take 24–48 hours
depending on the receiving bank's IMPS processing. During this window the
creator has submitted their KYC but cannot proceed. If the email confirming
the deposit is missed or goes to spam, the activation funnel silently stalls.

**Mitigation**: Show the pending penny-drop state explicitly in the creator
dashboard with a countdown and a manual "I've received it" confirmation flow.
Don't rely solely on an automated callback from the payment rail.

---

## What this proposal does NOT address

These risks are deliberately out of scope here:

| Risk | Notes |
|------|-------|
| Substack as payment aggregator (RBI PA licence) | Required if Substack ever holds or routes creator funds directly |
| 6-month realization cap enforcement | FEMA requirement for inward remittances; relevant at settlement layer |
| FIRC/FIRA generation | Compliance documentation for foreign inward remittances |
| Stripe-to-Stripe token migration | Assessed as contractually blocked; needs reassessment |
| GST LUT filing for creators | Required for zero-rated export of services |

---

## Verdict

This proposal is buildable by Substack without Stripe's involvement in the first
release. The intake form and signed telemetry bundle have standalone value
as internal records and manual-review assets. The Stripe endorsement bridge
(#103) is the high-value but high-dependency component — it should be
presented to Stripe as a partnership proposal, not assumed as a given.

The single most important open question before building: **does Stripe India
accept platform-attested PAN hashes, or do they need the raw value?** The
answer determines whether the DPDPA liability is manageable without an
India legal entity.

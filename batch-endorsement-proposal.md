# Option C — Batch Endorsement Proposal

**Thesis**: Stripe's internal prioritisation runs on GMV × margin. A single India creator at $200/month is invisible. Ten thousand India creators submitting via a trusted-platform batch protocol is a product partnership conversation. The goal of Option C is to change the unit of conversation from *creator* to *platform*.

---

## The Reframe

| Today | With Batch Endorsement |
|-------|----------------------|
| Stripe reviews each creator manually | Stripe reviews Substack's vetting process once |
| 1 creator = ~$200/month GMV | 500 creators/quarter = ~$100K/month GMV |
| Compliance cost: ~$40/creator | Compliance cost: ~$1/creator |
| Invite queue: 2–6 months | Batch approval: 48 hours |
| No committed SLA | Platform agreement with defined SLA |

This is not a feature request. It is Substack becoming Stripe's trust intermediary for the India individual creator segment — a segment Stripe cannot otherwise serve within its PA licence obligations at reasonable cost.

---

## Economics

**Stripe's current compliance cost per India creator (estimated)**

A manual KYC review involves a compliance analyst verifying PAN against NSDL, checking business registration, reviewing the application against PA framework criteria. At US-equivalent labour cost (~$60/hr), a 45-minute review costs ~$45 per creator.

| Scenario | Creators/quarter | Compliance cost | Creator revenue/year (2.9% × $150/mo) |
|----------|-----------------|-----------------|---------------------------------------|
| Current manual | 50 | $2,250 | $26,100 |
| Batch × 500 | 500 | $500 (process audit) | $261,000 |
| Batch × 2,000 | 2,000 | $500 (same audit) | $1,044,000 |

The marginal cost of each additional creator in a batch approaches zero once Stripe has audited Substack's selection process. This is the argument: **batch endorsement converts a per-creator compliance cost into a per-platform audit cost.**

At 10,000 India creators (Substack's plausible 2-year India cohort at current growth), the annual Stripe fee revenue exceeds $3M. That is a named account for Stripe India.

---

## The Platform Agreement

Batch endorsement requires Stripe to approve Substack as a trusted platform once, against a defined criteria set. This is analogous to how Stripe handles large enterprise platforms today — except it needs a formal process for the India PA context.

**What Substack commits to in the platform agreement**:

1. Every creator in a batch has passed Substack's vetting criteria (see below)
2. Substack has collected and verified: PAN (entity type + surname check), residential address, DOB, phone, bank account (penny-drop confirmed)
3. Substack retains the KYC records for 5 years (PMLA requirement)
4. Substack will notify Stripe within 48 hours of any creator whose account is suspended or flagged for fraud
5. Substack accepts joint liability for creators it endorses who subsequently generate chargebacks (up to a defined cap per batch)

**What Stripe commits to in the platform agreement**:

1. Fast-track review of batch submissions: 48-hour SLA from submission to account activation decision
2. No per-creator manual review for creators meeting the agreed criteria
3. Annual audit of Substack's vetting process (instead of ongoing per-creator oversight)
4. Named compliance contact at Stripe India for escalations

---

## Substack's Batch Selection Criteria

These are the thresholds that define which creators qualify for a batch submission. Stripe approves the thresholds; Substack enforces them.

| Criterion | Threshold | Rationale |
|-----------|-----------|-----------|
| Account age | ≥ 365 days | Reduces fake/transient account risk |
| Free subscribers | ≥ 100 | Confirms active publishing presence |
| Publications | ≥ 8 | Confirms ongoing content activity |
| Average open rate | ≥ 20% | Confirms real (non-bot) audience |
| Stripe chargebacks (lifetime) | 0 | Clean payment history |
| Abuse signal score | Below internal threshold | Bot/fraud detection |
| PAN verified | ✓ | Entity type + surname check passed |
| Bank account verified | ✓ | Penny-drop confirmed |
| Purpose code selected | ✓ | FEMA purpose code on file |

These are the same thresholds used in the individual endorsement flow (2a/2b screens). The batch protocol does not lower the bar — it changes the *delivery mechanism* from individual to batch.

---

## Quarterly Batch Cadence

**Why quarterly, not continuous**:
- Stripe's compliance team needs a predictable review window, not a firehose
- Quarterly batches give Stripe time to run its own spot checks between cycles
- Substack builds up a queue of qualified creators and submits them together, maximising GMV per batch submission

**Proposed schedule**:

| Month | Activity |
|-------|----------|
| Q1 Week 1 | Substack submits signed batch bundle (array of telemetry bundles) |
| Q1 Week 1–2 | Stripe compliance audits the batch against agreed criteria |
| Q1 Week 2 | Stripe activates approved accounts; returns exceptions list |
| Q1 Week 3 | Substack handles exceptions (re-submit or notify affected creators) |
| Q2 | Repeat |

**First batch target**: 100 creators — enough to demonstrate the process works, not so many that a single bad actor in the batch is embarrassing for Substack's endorsement programme credibility.

---

## API Design — Batch Endpoint

### Substack side (new)

```
POST /api/v1/india/kyc/telemetry/batch
```

Request body:
```json
{
  "batch_id": "batch_2026Q2_IN",
  "platform_criteria_version": "v1.0",
  "creators": [
    {
      "creator_id": "usr_001",
      "bundle_id": "tel_abc123",
      "bundle": "base64url_payload.base64url_sig",
      "creator_email": "priya@example.com"
    },
    ...
  ]
}
```

### Proposed Stripe extension

New endpoint (or extension of existing `platform_endorsements`):

```
POST /v1/platform_endorsements/batches
```

Request body:
```json
{
  "platform_id": "substack",
  "platform_criteria_version": "v1.0",
  "endorsements": [
    {
      "bundle_id": "tel_abc123",
      "bundle": "base64url_payload.base64url_sig",
      "verification_url": "https://substack.com/api/v1/india/kyc/telemetry/verify",
      "individual": { "email": "priya@example.com", "first_name": "Priya", "last_name": "Raghunathan" }
    }
  ]
}
```

Response:
```json
{
  "batch_id": "pbatch_stripe_xyz",
  "status": "processing",
  "estimated_completion": "2026-05-25T10:00:00Z",
  "total": 100
}
```

Status polling:
```
GET /v1/platform_endorsements/batches/pbatch_stripe_xyz
```

---

## Risk: What if Stripe rejects the platform agreement model?

If Stripe India compliance is unwilling to approve Substack's vetting process as a trust proxy (because PA regulations require them to do their own per-creator KYC), the batch model does not work as described.

**Fallback within Option C**: Stripe still processes batches, but runs a lighter-touch spot check on a random sample (e.g., 10% of each batch) rather than reviewing every creator. The SLA is 5 business days instead of 48 hours. This is slower but still dramatically better than the current invite queue, and it still achieves the GMV aggregation benefit.

**If Stripe rejects any batch model**: This is the trigger to move to Option A (domestic PA partner), which requires no Stripe cooperation at all. Option C and Option A are not mutually exclusive — running both in parallel hedges the dependency on Stripe's willingness.

---

## What Needs to Be Built

| Component | Owner | Dependency |
|-----------|-------|------------|
| Batch telemetry bundle generator | Substack eng | Extends Epic 1 #102 |
| Batch submission endpoint | Substack eng | Extends Epic 1 #103 |
| Platform criteria enforcement | Substack eng | Uses Epic 1 #101 thresholds |
| Batch status dashboard (creator-facing) | Substack product | New |
| Platform agreement with Stripe | Substack BD + Stripe India | This document |
| Stripe batch API endpoint | Stripe eng | Requires Stripe commitment |

The first four items are entirely within Substack's control. The platform agreement and the Stripe API endpoint are the external dependencies. Substack can build and demo the batch submission pipeline before the Stripe conversation — which is stronger than asking Stripe to build something Substack hasn't demonstrated it needs.

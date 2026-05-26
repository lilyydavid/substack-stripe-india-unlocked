# Epic 1 — API Design

**Addresses**: Issues #101, #102, #103
**Audience**: Substack Engineering, Stripe Connect Platform team

---

## Overview

Epic 1 spans two systems and three API surfaces:

```
Creator (browser)
    │
    ▼
[Substack] POST /kyc/intake          ← #101: validate & store creator identity
[Substack] POST /kyc/telemetry/sign  ← #102: compute + sign platform endorsement
    │
    ▼
[Stripe]   POST /v1/accounts (extended) ← #103: inject endorsement bundle into
                                           Connect account creation
```

---

## #101 — Creator Intake Endpoint

**Owner**: Substack

### `POST /api/v1/india/kyc/intake`

Accepts a creator's tax identity and banking details. Validates format at the
API layer before persisting. Returns a `kyc_record_id` used downstream by the
telemetry signer.

#### Request

```json
{
  "creator_id": "usr_8f3kq92",
  "pan": "ABCDE1234F",
  "legal_name": "Priya Raghunathan",
  "bank_account_number": "012345678901",
  "ifsc_code": "HDFC0001234",
  "business_intent": "newsletter_subscriptions",
  "declaration_accepted": true,
  "declaration_version": "IN_KYC_v1.2"
}
```

#### Field validation rules

| Field | Rule |
|-------|------|
| `pan` | Regex `^[A-Z]{5}[0-9]{4}[A-Z]{1}$`; must match `legal_name` initial |
| `ifsc_code` | Regex `^[A-Z]{4}0[A-Z0-9]{6}$`; RBI IFSC master list lookup |
| `bank_account_number` | 9–18 digits; IMPS penny-drop verification recommended |
| `business_intent` | Enum — see Purpose Code map below |
| `declaration_accepted` | Must be `true`; timestamp server-side |

#### Purpose Code mapping

| `business_intent` value | RBI Purpose Code | Description |
|-------------------------|------------------|-------------|
| `newsletter_subscriptions` | P1101 | Creative content — text |
| `podcast_subscriptions` | P1102 | Creative content — audio |
| `video_subscriptions` | P1103 | Creative content — video |
| `consulting_retainer` | P0801 | Software/IT consultancy |
| `research_reports` | P1006 | Research & development |

#### Response `200 OK`

```json
{
  "kyc_record_id": "kyc_7hx29ms",
  "status": "pending_verification",
  "purpose_code": "P1101",
  "penny_drop_required": true,
  "estimated_review_hours": 24
}
```

#### Error responses

| HTTP | Code | Meaning |
|------|------|---------|
| 422 | `invalid_pan_format` | PAN regex failed |
| 422 | `pan_name_mismatch` | PAN initial doesn't match legal name |
| 422 | `invalid_ifsc` | Not found in RBI IFSC master list |
| 409 | `kyc_already_submitted` | Duplicate submission for creator |
| 403 | `declaration_not_accepted` | Missing consent |

---

## #102 — Platform Telemetry Signing

**Owner**: Substack

### `POST /api/v1/india/kyc/telemetry/sign`

Called internally (not by the creator) after KYC record is verified.
Assembles platform-attested metrics, signs with Substack's private key,
and stores the signed bundle for use in the Stripe endorsement request.

#### What gets signed

```json
{
  "kyc_record_id": "kyc_7hx29ms",
  "creator_id": "usr_8f3kq92",
  "platform": "substack",
  "attested_at": "2026-05-23T10:00:00Z",
  "metrics": {
    "account_age_days": 847,
    "free_subscriber_count": 3200,
    "paid_subscriber_pledges": 94,
    "avg_open_rate_pct": 41.2,
    "publications_published": 112,
    "stripe_chargebacks_lifetime": 0
  },
  "purpose_code": "P1101",
  "pan_hash": "sha256:e3b0c44298fc...",
  "declaration_version": "IN_KYC_v1.2"
}
```

**Note on `pan_hash`**: The raw PAN is never transmitted to Stripe. Only the
SHA-256 hash is included in the signed bundle. Substack retains the original
under its own data handling obligations.

#### Signing scheme

```
payload_bytes = canonical_json_utf8(telemetry_object)
signature     = HMAC-SHA256(payload_bytes, SUBSTACK_INDIA_SIGNING_KEY)
bundle        = base64url(payload_bytes) + "." + base64url(signature)
```

A public verification endpoint (`GET /api/v1/india/kyc/telemetry/verify`)
allows Stripe's compliance automation to confirm authenticity without
sharing the signing key.

#### Response `200 OK`

```json
{
  "bundle_id": "tel_4kp91az",
  "bundle": "<base64url_payload>.<base64url_sig>",
  "expires_at": "2026-06-22T10:00:00Z"
}
```

Bundles expire in 30 days. Stripe must consume before expiry.

---

## #103 — Stripe Connect Endorsement Bridge

**Owner**: Stripe (API change request) + Substack (integration)

### What Substack sends to Stripe

This is a proposed extension to Stripe's existing `POST /v1/accounts`
endpoint for Connect account creation. The extension adds a
`platform_endorsement` object to the existing request body.

**This requires Stripe to implement a new optional field** — see the
Feedback to Stripe section below.

#### Proposed extended request to `POST /v1/accounts`

```json
{
  "type": "custom",
  "country": "IN",
  "email": "priya@example.com",
  "capabilities": {
    "transfers": { "requested": true }
  },
  "business_type": "individual",
  "individual": {
    "first_name": "Priya",
    "last_name": "Raghunathan",
    "email": "priya@example.com"
  },
  "platform_endorsement": {
    "platform_id": "substack",
    "bundle_id": "tel_4kp91az",
    "bundle": "<base64url_payload>.<base64url_sig>",
    "verification_url": "https://substack.com/api/v1/india/kyc/telemetry/verify"
  }
}
```

#### What Stripe should do with this

1. Verify the bundle signature against `verification_url`
2. Extract `metrics` from the verified payload
3. Apply a platform-endorsed fast-track review queue (target: 48h vs. current
   2–4 week manual queue)
4. Return a standard `account` object with `requirements` populated as usual

#### Stripe response (no change from current spec)

```json
{
  "id": "acct_1N2AbCdEfGhIjKlM",
  "object": "account",
  "charges_enabled": false,
  "payouts_enabled": false,
  "requirements": {
    "currently_due": ["individual.id_number", "tos_acceptance.date"],
    "eventually_due": [...],
    "pending_verification": []
  }
}
```

The `platform_endorsement` field does not change what Stripe requires — it
only changes the queue priority and review SLA.

---

## What Substack needs to build

- KYC intake form (Issue #101) — creator-facing
- Telemetry aggregation service (Issue #102) — internal
- Endorsement submission client — calls Stripe's API with the bundle

## What Stripe needs to build

- Accept `platform_endorsement` field on `POST /v1/accounts`
- Implement bundle verification call to `verification_url`
- Implement fast-track review queue for verified platform endorsements
- Publish the endorsement program criteria (which platforms qualify, what
  metric thresholds trigger fast-track)

---

## Open Questions for Stripe

1. Is there a Stripe Connect Partner Programme tier that already offers
   expedited review for high-trust platforms? If so, Substack may qualify
   without a new API primitive.
2. What are Stripe India's minimum thresholds for a creator to clear manual
   review today? Publishing these would let Substack pre-filter applicants.
3. Will Stripe accept a `pan_hash` as a KYC signal or does their RBI
   compliance require the raw PAN to be submitted directly?

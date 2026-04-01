# Vendor Payout Engine — Subscription Feature Access Reconciliation

> A product case study on designing a usage-based vendor payout mechanism for an OTT platform on a subscription engine that had no prior feature access tracking.

---

## Background

This case study describes a feature built on a B2B subscription management platform used by an OTT (Over-The-Top) content provider. The OTT had partnered with 10+ content vendors, each providing distinct services bundled into subscription plans sold to end users.

The commercial agreement between the OTT and its vendors was usage-based: **a vendor would only be paid if a subscriber actually accessed their content**, not simply because the user had purchased a plan containing it.

This created a product problem. The subscription platform had a **plan-to-feature mapping** — it knew which features belonged to which plan — but had no mechanism to record **which features a subscriber actually used**. Without this, there was no reliable basis for vendor payouts.

---

## Problem Discovery

The requirement surfaced during onboarding of this OTT client. It was a condition of their vendor contracts: payout had to be tied to consumption, not entitlement. There was **no prior process**, this was net new. The requirement was new, and the platform had never built for it.

The ask was clear: build a mechanism to track feature access at the subscription level and generate a monthly vendor payout report that the OTT's revenue and commercial teams could use for vendor reconciliation.

---

## Stakeholders

| Stakeholder | Role in This Feature |
|---|---|
| OTT Client (Product & Engineering) | Source of feature access data; finalized API log format with us |
| Revenue Team (OTT) | Primary consumer of the payout report; reconciles against vendor-submitted reports |
| Commercial Team (OTT) | Uses payout data to manage vendor contracts and disputes |
| Vendors (10+) | Maintain their own access records; cross-check against OTT report |
| Internal Engineering | Built the ingestion API and report generation logic |

---

## Solution Design

### Why Push, Not Pull

An early design question was whether our platform should pull access data from the OTT or have them push it to us.

**Decision: Push (OTT sends logs to us in real time)**

The OTT's systems are the source of truth for feature access, they know the moment a user opens a piece of content. We have no visibility into user behaviour on their platform. Building a pull mechanism would have required the OTT to expose a queryable access log API on their end, introducing unnecessary complexity and latency. A push model was simpler, more real-time, and placed the data responsibility where it naturally lived.

---

### API Design — Feature Access Log Ingestion

I designed the structure of the API that the OTT uses to send feature access logs to the platform in real time.

**Key fields in each log entry:**

- `subscription_id` — the active subscription against which the feature was accessed
- `feature_id` — the specific vendor service/content accessed
- `user_id` — the subscriber
- `accessed_at` — timestamp of the access event

**Validation logic:**

A critical constraint was preventing duplicate counting within a billing cycle. The API enforces that **for a given billing cycle, only one `subscription_id` + `feature_id` combination is accepted**. Duplicate submissions for the same pair within the same cycle are rejected.

This was a deliberate product decision: the payout metric is whether a subscriber accessed a vendor's service in a given month — not how many times. This kept the payout model clean, binary, and defensible to vendors.

---

### Payout Report Design

At the start of every month, the platform generates a payout report covering all feature access events logged in the prior month.

**Report structure (per vendor):**

- Vendor name / ID
- Feature(s) offered by the vendor
- Count of unique subscription-feature access events in the billing cycle
- Derived payout basis (access count × agreed rate, if applicable)

The access count serves as the proxy for unique users consuming the vendor's service in that period. The revenue team uses this report alongside vendor-submitted access records to reconcile and approve payouts.

---

## Tradeoffs and Decisions

### Binary access model vs. frequency-weighted
Considered tracking the number of times a feature was accessed per user, but the vendor contracts defined payout on a per-subscriber basis, not per-view. The binary model (accessed or not, per subscription per cycle) was both contractually correct and simpler to validate.

### Real-time ingestion vs. batch upload
The OTT pushes logs in real time as users access content. An alternative was a periodic batch file upload (e.g., daily or weekly). Real-time was chosen because it reduces end-of-cycle reconciliation risk — if there's a data gap, it's caught early rather than at month close.

### Report generation timing
The payout report is generated at the **start of the month** for the prior month's data rather than on the last day. This gives a clean cutoff and ensures any late-arriving logs (within the cycle) are captured before the report runs.

---

## Outcome

- Feature is **live in production** with the OTT client
- Covers **10+ vendor partnerships**
- Revenue and commercial teams use the monthly report for vendor reconciliation
- **Zero reconciliation disputes** since go-live
- Filled a structural gap in the platform — subscription-level feature access tracking now exists as a foundation for future usage-based billing use cases

---

## What I Owned

- End-to-end product definition: problem framing, solution approach, API structure, report design
- Worked directly with the OTT client's team to finalize the log format and API contract
- Defined validation logic (duplicate prevention per billing cycle)
- Designed the payout report structure in collaboration with the revenue and commercial teams

---

## Reflections

This feature is a good example of a platform gap that only becomes visible when a specific client's business model demands it. The OTT's vendor contracts required usage-based payouts; the platform was built for entitlement-based plans. Bridging that gap required both a new data ingestion mechanism and a new reporting layer.

The zero-dispute track record since launch suggests the model — simple, binary, validated at ingestion — was the right call over a more complex frequency-based approach.

---

*This case study is based on real product work. Data, client names, and vendor details have been omitted*

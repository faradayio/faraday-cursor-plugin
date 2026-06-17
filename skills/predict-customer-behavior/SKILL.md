---
name: predict-customer-behavior
description: Build a Faraday propensity model (Outcome) to predict customer behavior — churn, conversion, repeat purchase, upgrade, lead scoring, donation, attendance, response. Use whenever the user says "predict", "score", "rank", "lookalike", "propensity", "LTV", "churn", or asks "who is likely to X". Builds the attainment + eligibility Cohorts, the Outcome, and (optionally) a Scope/Target to deploy the scores.
---

# Predict customer behavior with a Faraday Outcome

Outcomes are Faraday's propensity-modeling primitive. A trained Outcome takes a person and returns a 0–1 probability that they'll do "the thing" inside the prediction horizon, plus per-person feature contributions.

## What you need from the user before you start

Before calling any tool, get clear on these. If any are missing, ask the user — don't guess.

1. **The behavior being predicted.** Concrete and observable. "Churn" is too vague; "cancel a paid subscription within 90 days of today" is buildable.
2. **The eligibility universe.** Who *could* exhibit the behavior? For a churn model: current paying subscribers. For a lookalike: the addressable US population (no eligibility Cohort needed). For a lead-conversion model: leads who entered the funnel in the last N days.
3. **The prediction horizon.** Point-in-time ("will buy ever?") vs time-to-event ("will buy within 30 days?"). Time-to-event is almost always more useful operationally.
4. **The deployment target (if any).** Are scores going to a Meta custom audience? A Klaviyo segment? A nightly BigQuery export? This decides whether you stop at the Outcome or continue through to Scope + Target.

## Reuse first

`get_graph` and `list_cohorts`, `list_outcomes`. Many "predict churn" requests already have a churn Cohort sitting in the account from a previous configuration — train a fresh Outcome on top of it instead of duplicating the Cohort definition.

## Step 1 — Build (or find) the attainment Cohort

The attainment Cohort is "people who have done the thing." Definition is typically:

- **Trait predicate**: e.g. `subscription_status == "cancelled"` and `cancelled_at within last 180 days`.
- **Stream predicate**: e.g. ≥ 1 `cancellation` event in the last 180 days, optionally with payload filters.
- **Cohort intersection**: e.g. `cancellers` ∩ `paying_subscribers_180_days_ago`.

Call `create_cohort` with the predicate tree, then `get_cohort_membership` to sanity-check size. A few thousand attainers is enough for a usable model; a few dozen is not.

## Step 2 — Build (or find) the eligibility Cohort

The eligibility Cohort is "people who *could* have done the thing." This is what gives the Outcome a meaningful negative class.

- For **churn**: paying subscribers as of the start of the observation window. (Not "everyone in the database" — non-subscribers can't churn.)
- For **conversion**: leads who entered the funnel; or trial users; or freemium users.
- For **purchase propensity**: all known persons in the customer database, or all US adults if you want lookalike behavior.

If `eligible_cohort_id` is omitted, Faraday uses the full US adult population. That's the right default for top-of-funnel lookalike but wrong for any "behavior within an existing population" use case.

## Step 3 — `create_outcome`

Required fields:

- `name` — sentence case, descriptive ("Churn within 90 days").
- `attainment_cohort_id`
- `eligible_cohort_id` (almost always — see step 2)

There is no horizon/date argument — the modeling approach is automatic (`prediction_mode` defaults to `auto`), and the time window is encoded in the attainment Cohort's own recency predicate.

After the call returns, poll `get_outcome` (or `get_graph`) until `update_job_state.state == "succeeded"`. Training typically takes 10–60 minutes depending on attainment size, feature breadth, and queue depth.

## Step 4 — Inspect the trained model

Before deploying, look at:

- `get_outcome_analysis` — feature importance and validation metrics (AUC, lift charts, calibration). If AUC is near 0.5, the model didn't learn anything; check the attainment definition.
- `get_outcome_report` — shareable HTML report.

Flag suspicious results to the user instead of blindly deploying:

- AUC < 0.6 — the model is essentially random; the behavior may not be predictable from available data, or the Cohorts are wrong.
- A single feature dominating importance — usually means the feature is downstream of the behavior (leakage). Common offender: a Trait whose value is set by the same event that defines the attainment.

## Step 5 (optional) — Deploy via Scope + Target

If the goal is operational (not just analytical):

1. `create_scope` with:
   - `population.cohort_ids` — who you want scored (array). Often the eligible Cohort from step 2.
   - `payload.outcome_ids` — `[<outcome_id>]` for the Outcome you just trained.
2. `create_target` against that Scope pointing at the destination Connection — Meta custom audience, Klaviyo segment, BigQuery table, Hosted CSV, webhook.

Poll the Target's `update_job_state` until `succeeded`, then `get_target_analysis` for delivery counts.

## Common patterns

| User's intent | Attainment | Eligibility | Notes |
|---|---|---|---|
| Predict churn | Cancellers in last N days | Paying subscribers as of N+window days ago | Size the attainment window to the retention contract length. |
| Lookalike (acquisition) | High-LTV customers | (omit — uses US adults) | Score the full US population, deploy top decile to Meta. |
| Lead conversion | Leads who became customers | All leads in same window | Attainment window ≈ typical sales cycle. |
| Repeat purchase | Customers with ≥ 2 purchases | Customers with ≥ 1 purchase | Use `purchases` Stream with payload filters. |
| Upgrade / cross-sell | Customers who upgraded | Eligible-to-upgrade customers | Eligibility filters out customers already on the top plan. |
| Donation | Donors in last cycle | All known constituents | Attainment window = the prior campaign cycle. |
| Event attendance | Past attendees | List recipients / market | Useful with `create_market_opportunity_analysis` for venue siting. |

## What not to do

- **Don't combine multiple unrelated behaviors into one Outcome.** Build one Outcome per behavior. Deploying multiple Outcomes in the same Scope payload is cheap and gives you per-behavior scores.
- **Don't use a tiny attainment Cohort.** If `get_cohort_membership` shows < a few hundred, tell the user, don't silently train.
- **Don't skip the eligibility Cohort for in-population predictions.** Without it, the model is implicitly predicting "is this person a customer?" rather than "will this customer do X?".
- **Don't deploy without inspecting `get_outcome_analysis` first.** Surface AUC and top features to the user before creating any Target.

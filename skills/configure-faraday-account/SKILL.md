---
name: configure-faraday-account
description: Configure a Faraday account end-to-end via the Faraday MCP — set up Connections, ingest Datasets, define Cohorts, train Outcomes / Persona Sets / Recommenders, run Market Opportunity Analyses, and deploy through Scopes and Targets. Use whenever the user asks to "set up", "configure", "provision", "wire up", or "deploy" anything in Faraday.
---

# Configure a Faraday account

Faraday accounts are configured by walking the resource chain from raw data on the left to a deployed prediction on the right:

```
Connection -> Dataset -> Trait + Stream -> Cohort -> Outcome / Persona Set / Recommender -> Scope -> Target
```

This skill walks through the whole chain. Most real tasks only need a slice of it; use the section labeled with the user's actual goal.

## Step 0 — Orient yourself in the account

**Always do this first.** Skipping it is the #1 source of duplicate, broken configurations.

1. Call `get_graph` to see the existing dependency tree for the account.
2. Call `list_connections`, `list_datasets`, `list_streams`, `list_traits`, `list_cohorts`, and `list_outcomes` (whichever are relevant to the task) to confirm what's already present.
3. Identify which resources you can reuse and which you actually need to create.

If the user says "predict churn for our subscribers", you probably already have the Connection, Dataset, and Stream — you just need a churn Cohort + Outcome.

## Step 1 — Connection (only if a usable one doesn't exist)

A Connection points Faraday at an external environment for ingress or egress:

- Warehouses: BigQuery, Snowflake, Redshift, Postgres, …
- Cloud storage: S3, GCS, SFTP, …
- SaaS: Mailchimp, Klaviyo, HubSpot, Salesforce, mParticle, Meta / Google / TikTok, …
- "Hosted CSV": file uploads managed by Faraday itself

Tools: `list_connections`, `get_connection`, `create_connection`. Call `get_connection` with `expand: ["datasets", "targets"]` to see what already depends on a Connection before suggesting changes.

If the user wants to upload a CSV directly, use `create_upload` first, then a `create_connection` of type `hosted_csv`.

## Step 2 — Dataset (column → trait/stream mapping)

A Dataset is **the contract** between raw data and Faraday's person-level model: it declares how columns map to identity (email, phone, address), Traits, and Stream events.

Tools: `list_datasets`, `get_dataset`, `create_dataset`, `update_dataset`, `get_dataset_ingress_logs`.

Key choices when calling `create_dataset`:

- **Identity sets** — which columns identify a person (e.g. `email`, `first_name`+`last_name`+`postal_code`). Faraday uses these to deduplicate across data sources.
- **Output to Traits** — per-person attributes like `lifetime_value`, `subscription_tier`.
- **Output to Streams** — typed events like `transactions`, `signups`, `cancellations`. Faraday auto-creates the Stream if it doesn't exist; `list_streams` to inspect.

After a `create_dataset` call, poll `get_dataset` (or `get_graph`) until `update_job_state.state == "succeeded"`. Failures usually show up in `get_dataset_ingress_logs`.

## Step 3 — Cohort (the universal selection primitive)

A Cohort is a **population of people** defined by a combination of Traits and Stream events. Every objective and every deployment in Faraday is anchored on Cohorts.

Tools: `list_cohorts`, `get_cohort`, `create_cohort`, `update_cohort`, `get_cohort_membership`.

A Cohort definition is a tree of predicates over:

- Traits (e.g. `lifetime_value > 500`, `state in ['VT', 'NH']`)
- Streams (e.g. has experienced ≥ 1 `transaction` event in the last 90 days)
- Other Cohorts (intersect, union, exclude)

Examples that map cleanly onto Cohorts:

| Use case | Source Cohort |
|---|---|
| Churn propensity | "Active subscribers as of 90 days ago" |
| Lookalike modeling | "High-LTV customers" |
| Lead scoring | "Inbound leads in last 30 days" |
| Persona segmentation | "All customers" |
| Market opportunity | (Cohort optional — usually US population) |

Sanity-check the size with `get_cohort_membership` before spending compute on an Outcome — a Cohort of 17 people will not train a usable propensity model.

## Step 4 — Objective: pick the right one

Faraday has four objective types. Choose by intent, not by feature familiarity:

| If the question is… | Use… | Tool |
|---|---|---|
| Who is **likely to do X**? | Outcome | `create_outcome` |
| What are the **natural groupings** in this population? | Persona Set | `create_persona_set` |
| What **product / item / location** should this person see? | Recommender | `create_recommender` |
| **How many** of these people are out there, and **where**? | Market Opportunity Analysis | `create_market_opportunity_analysis` |

### Outcomes (propensity)

`create_outcome` takes:

- **`attainment_cohort_id`** — people who have done the thing.
- **`eligible_cohort_id`** (optional) — people who could have done it (often a superset of attainment). The model learns to distinguish attainers from eligible-but-not-attainers within this universe. Omit it to score against the full US adult population (lookalike).
- **`attrition_cohort_id`** (optional) — explicit counter-examples, if you have them.

The modeling approach is chosen automatically (`prediction_mode` defaults to `auto`); there's no horizon/date argument — encode the time window in how you define the attainment Cohort (e.g. "cancellations in the last 90 days").

Pull `get_outcome_analysis` once `update_job_state` is `succeeded` for feature importance and performance metrics. `get_outcome_report` gives a shareable HTML report.

### Persona Sets (segmentation)

`create_persona_set` takes a source Cohort and a target number of personas. Faraday runs a k-means++-like algorithm and labels the resulting personas by their dominant Traits.

After it succeeds, `get_persona_set_dimensions` is the key tool — it surfaces the trait breakdown that explains what makes each persona different. Use `update_persona` to rename individual personas (e.g. "Cluster 3" → "Suburban high-income families").

### Recommenders

`create_recommender` takes one or more interaction Streams (transactions, page views, ratings). The output is a per-person ranked list of items.

### Market Opportunity Analyses

Doesn't require a customer Cohort at all — operates on the US population and a definition of what an opportunity looks like (e.g. "households with income > $100k and at least one child under 5 in a specified geography").

## Step 5 — Scope (population + payload)

A Scope bundles **who** (a population Cohort, optional exclusion Cohorts) with **what context to deliver** (the payload: Outcomes, Persona Sets, Recommenders, and pass-through Cohorts).

Tools: `list_scopes`, `get_scope`, `create_scope`, `update_scope`, `get_scope_analysis`, `get_scope_efficacy`.

To inspect an existing Scope without walking the graph by hand, call `get_scope` with `expand`:

- `expand: ["population_cohorts", "population_exclusions"]` — who's in it
- `expand: ["payload_cohorts", "payload_outcomes", "payload_persona_sets", "payload_recommenders"]` — what they get

If the user says "deploy this", they almost always mean `create_scope` followed by `create_target`.

## Step 6 — Target (where context is delivered)

A Target attaches a delivery destination to a Scope. Same Scope can fan out to many Targets.

Tools: `list_targets`, `get_target`, `create_target`, `update_target`. Use `get_scope` with `expand: ["targets"]` to inspect a specific Scope's targets rather than scanning the whole account.

Common Target types:

- **Ad platforms** — Meta, Google, TikTok custom audiences
- **ESPs** — Mailchimp, Klaviyo
- **CDPs / CRMs** — HubSpot, Salesforce, mParticle
- **Warehouses & files** — BigQuery, Snowflake, S3, GCS, SFTP
- **Hosted CSV** — `download_target` returns the file
- **Faraday Lookup API** — real-time per-person lookup
- **Webhook** — POSTs per-person payloads to a URL

For ad platforms and ESPs, the Connection from Step 1 must already authorize Faraday against the destination account.

## Reading the resource graph

When you're handed an unfamiliar account, `get_graph` is the single best tool. It returns the full dependency tree, including `update_job_state` for every node, so you can answer "what's broken?", "what's stale?", and "what feeds what?" with one call.

## Async polling pattern

For any `create_*` on Cohort, Outcome, Persona Set, Recommender, Dataset, Target, Place, or Market Opportunity Analysis:

1. Note the returned `id` and the initial `update_job_state`.
2. Poll `get_<resource>` (or `get_graph`) every 30s–5min depending on resource size.
3. When `update_job_state.state == "succeeded"`, the resource is usable. On `"failed"`, surface `update_job_state.error_message` to the user; don't retry blindly.

## Memory management

Almost every tool accepts a `jmespath_filter` argument. Use it. For a list endpoint, `[].{id: id, name: name, state: update_job_state.state}` gives you a ~50× smaller payload than the default. For a Persona Set you only want the names of, `personas[].{id: id, name: name}` is enough.

## Sub-account fan-out

If you're working in a parent account that holds many brand sub-accounts, `list_accounts` enumerates them and `get_account` returns each one's API key. Repeat the workflow above against each sub-account as needed.

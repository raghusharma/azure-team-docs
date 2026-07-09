> Published from CloudKeeper's Azure Commit working docs on 2026-07-09. Auto-generated copy — don't edit this file; changes are overwritten on the next publish. Questions / change requests → Raghu Sharma.

# Azure Commit — recommendation system-of-record design

> **House rules for this doc — read before editing**
> This is the **why**: the model and the architecture decisions. Keep it that way.
> - **Present tense, no decision log.** No dated "Decisions locked" entries — git history is the log.
> - **No code mechanics.** Parse/MERGE/invariant detail lives in the pipeline repo's `CLAUDE.md`.
> - **No acquisition map.** Which API/scope/RBAC yields which holding lives in the [Azure Commit holdings scan contract](holdings-scan-contract.md).
> - **Litmus:** if a section reads like a changelog, restates code, or lists per-customer steps, it's in the wrong doc. One strong line + a link beats a re-derived paragraph.

The **upstream model** feeding Azure Commit summary table schema, the frontend-facing
*terminal sink* (realized economics per resource per usage-date). This doc designs what sits
**before** it: the system-of-record for *what we recommended and what happened to it* — the
basis the 25% chargeback is billed against.

Scoped **Commit-first** (Azure Commit is current priority). Azure Tuner reuses the
same spine later — see Tuner reuse (shelved).

## Where it sits (the five boxes)

```
Generate → RECORD → Detect(purchase) → Attribute → Display
 scripts   (this    reservation API    realized    summary table
           doc)                        savings     (already designed)
```

- **Generate** — the RI/SP recommendation scripts the team is building. A scanner; emits the run files the ingest loader (#1) records.
- **Record** — *this doc.* System-of-record: two tables — `recommendation` (mutable target) + `commitment_ledger` (immutable holdings).
- **Detect** — for Commit, "implemented" = *reservation purchased* (billing/reservation API),
  **not** infra-change detection. That (Box 3, `resourcechanges` diffing) is Tuner-only.
- **Attribute** — realized savings from `date_implemented`, computed by the pipeline.
- **Display** — Azure Commit summary table schema. Reads realized savings. Already built.

The customer **Savings Delivered** page shows *realized savings only* → reads the summary
table → **already unblocked, independent of this doc.** This record is internal.

## Pipeline map (the four builds)

Four pipelines across two tracks (advice vs holdings), mapping to the five boxes above:

| # | Pipeline | Reads | Writes | Track | Status |
|---|---|---|---|---|---|
| 1 | **ingest loader** | recommendation run files (S3 `commit/`) | `recommendation`, `recommendation_run_log`, `ingest_run` | advice | built |
| 2 | **commitment loader** | a [holdings scan file](holdings-scan-contract.md) (S3) — CK purchases + pre-existing reservations/SPs/LA tiers, producer-normalized | `commitment_ledger` | holdings | built |
| 3 | **realized-savings loader** | discounted usage (FOCUS parquet, from the S3 landing) tagged by `lease_id` | summary table (per-resource realized savings) | holdings | to build |
| 4 | **invoice generator** | `commitment_ledger` ⨝ summary (on `lease_id`) | invoice (a report, not a table) | billing | to build |

**"Commitment loader"** (not "reservation loader"): it matches `commitment_ledger`, and
*commitment discount* is the umbrella for both Reserved Instances **and** Savings Plans
("reservation" reads as RI-only).

The customer **Savings Delivered** page is frontend/API over the summary table — downstream of
#3, not one of the four. Invoice = `25% × realized savings` for holdings where
`source = 'ck-purchased'` and today is within `[fee_start, fee_end]`.

### Repository structure (all four in one repo)

One repo (`azure-commit-pipeline`), one shared `sql/schema.sql`, one Snowflake config —
so the four pipelines live together, not in four repos. A shared `common` package + one
subpackage per pipeline:

```
azure_commit/
  common/        config, s3, errors, logging, identity, db  ← shared by all four
  ingest/        contract, mapping, pipeline, repository     ← #1 (built)
  commitment/    contract, mapping, pipeline, repository     ← #2 (built)
  # savings/     FOCUS export → summary table                ← #3
  # invoice/     ledger ⨝ summary → report                   ← #4
```

Two structural rules hold the pipelines together without coupling them: **`common` never imports
from a pipeline package** (pipelines import from `common`), and **`identity` lives in `common`** —
`recommendation_id` is the cross-pipeline join key, so the commitment loader recomputes it there to
match a holding to its rec. The Snowflake connection / transaction / schema seam is a shared
`common/db.py` base both loaders' `snowflake_repo` inherit. No empty stub packages — each pipeline
lands with its code.

## Export landing: customer Blob → CK S3

Pipelines consume cost exports from **CK's S3, not the customer's storage account**. A thin daily copy job — an AWS **Lambda** in the target
account's VPC, egressing through the NAT EIP on Itron's storage allowlist (both prod &
nonprod EIPs allowlisted, confirmed 2026-07-09) — reads each export run's `manifest.json` and
mirrors the raw parquet to S3 **unchanged**. Lambda and bucket sit in the **same account** (the
data path is not cross-account); everything downstream (#3 and any other usage reader) reads
only from S3. VPC-attach is load-bearing — a default Lambda's egress IPs are firewall-denied.

**Where it lands:** a **new dedicated cost-export bucket**, in the **production account for now**.
The eventual home is the data account (where every cloud's CUR processing lives), but that would
block on access coordination — prod's EIP is already allowlisted, so start there. Migration later
is an S3→S3 copy (Azure overwrites month-to-date, so history can't be re-pulled) plus a config
repoint; bucket/prefix stay behind config. No external layout convention — we define our own.

**Runtime & infra:** Go Lambda + EventBridge daily schedule, provisioned by **Pulumi (Go)** in a
separate repo (`azure-cost-export-copy`) with `dev` (non-prod) and `prod` stacks. Non-prod is a
*real* end-to-end test — its NAT EIP is also allowlisted, so a `dev` Lambda reaches live Itron
data. Failure signalling via **SNS**, on two independent triggers: an `Errors` alarm (ran and
failed) and an `Invocations` dead-man's-switch (never ran). A managed Blob→S3 transfer is the
eventual analogue but stays deferred — the copy job is enough and doesn't block other builds.

Why S3 at all: raw history survives customer-side retention/access changes (storage firewall, SOW
end); reprocessing never touches the customer again; one copy serves every consumer.

- **All three datasets** (focus + actual + amortized), not just FOCUS — same rationale as
  the onboarding "All data" template: engine changes then need zero customer-side change.
- **Dated pulls, not a mirror** — land under
  `azure-cost/<customer>/<export-name>/<billing-month>/<pull-date>/` (manifest included).
  The exports run `OverwritePreviousReport`, so Azure keeps only the latest month-to-date
  run; dated pulls preserve the restatement history Azure discards. If size ever matters,
  lifecycle-expire intra-month pulls but keep each month's final pull forever.
- **Source is a URI, not an abstraction layer** — readers take a manifest URI and read via
  fsspec/duckdb, which handle `s3://` and `abfss://` alike. Moving pipelines to Azure later
  is a config change to the source URI; no adapter framework.
- The copy job records each pull as an S3 `_COPY_COMPLETE.json` marker beside the data
  (run-GUID, files, row/byte counts from the manifest), and **skips the whole pull if that
  run-GUID is already copied**. This resolves the earlier "new `ingest_run` kind vs sibling
  ledger" question: **no DB table** — the marker is self-describing, travels with the data
  during the data-account migration, and the savings loader (#3) reads it for provenance.
- Overwrite mode means each daily pull re-downloads the whole month-to-date (no incremental
  runs). Azure egress is billed to the customer's storage account; at Itron scale this is a
  few GB/day compressed — negligible, but worth stating in the onboarding doc.

**Lens boundary**: Lens (visibility) ingests the actual/amortized parquet with its own
transforms into its master/summary tables. Commit deliberately does **not** read those —
different needs (FOCUS-native, customer-effective cost, `lease_id` attribution,
billing-grade lineage for the 25% invoice). The shared artifact is the export feed itself,
not a shared table.

## Core principle: target (mutable) vs holdings (immutable)

A reservation recommendation is **not per-resource.** You don't reserve "for a VM" — you
reserve a *quantity* of a normalized SKU family, in a region, at a billing scope, and Azure
auto-applies the discount to whatever matching usage exists (Instance Size Flexibility). So
the model has two entities:

| Entity | What it is | Mutability | Cardinality |
|---|---|---|---|
| **recommendation** | the recurring *need* / **target** — "this workload warrants N units of coverage" | `recommended_quantity` **re-sized every scan** | one stable row per need |
| **commitment ledger** | a **holding** — an actual reservation/SP that exists (CK-bought *or* pre-existing) | `quantity` **immutable** once bought | many per recommendation |

Everything dynamic falls out of **the delta between target and the sum of active holdings:**

```
coverage = Σ ledger.quantity  WHERE recommendation_id = R  AND status = 'active'
residual = recommendation.recommended_quantity − coverage
  residual > 0  → under-covered  → top-up / next tranche
  residual < 0  → over-committed → wastage (confirm via summary UNUSED_HRS)
  residual = 0  → covered
```

This one relationship handles renewals, laddered partial buys, post-purchase usage drift,
and pre-existing reservations — see Why one structure covers all the dynamics.

### FOCUS attribution: unused-commitment rows carry a null subscription

In the FOCUS export, unused reservation / savings-plan charges (and rounding adjustments) have
`SubAccountId` (subscription) and `ResourceId` set to **null/empty** — by design in the enhanced
export experience, null means "no consuming subscription," not missing data. Attribute these to the
commitment (`CommitmentDiscountId`) as **wastage** — the source for summary `UNUSED_HRS` and the
`residual < 0` over-committed signal above — never to a subscription, and **never coalesce null to
the empty GUID `00000000-…`** (that invents a phantom subscription and misallocates the waste).

### Deterministic identity (the load-bearing decision)

Each run file is one scan; across runs the same need reappears. A run's row for that need is an
*observation* with no identity of its own — the DB collapses those N observations into one entity
via a deterministic key derived from **what makes a need the same need:**

```
recommendation_id = hash(tenant_id, scope_id, region, service, normalized_sku_family)
```

Each scan `MERGE`-upserts on this key. **`region` is in the identity** — reservations are
region-bound (a West-Europe RI does not cover East-US usage). **`commitment_type` and
`term` are NOT in the identity** — they're the *recommended action* (RI-vs-SP, 1yr-vs-3yr),
decision parameters that can change between tranches/renewals without changing the
underlying need. They live on the holding (what was actually bought).

## Cost basis: customer-effective, NOT retail

Savings are billed at 25% of **savings *delivered***, so the baseline must be the customer's
**actual negotiated rate** (MACC/MCA/EA), sourced from their **FOCUS export**
(`EffectiveCost`/`ContractedCost`) — *not* the Retail Prices API. Billing 25% of
`(retail_OD − reserved)` when the customer already pays ~20% sub-retail (e.g. Eptura)
would overstate savings and overcharge. The Retail Prices API is for reservation *list
pricing* and reference only.

## Status & coverage reconciliation

The recommendation has no linear "purchased" lifecycle — it is **continuously reconciled**
against the live ledger each scan:

- **recommendation.status** = advice state, from three independent sources — `open` (residual > 0)
  · `covered` (residual ≈ 0, a **ledger** fact) · `withdrawn` (dropped from the generator feed) ·
  `dismissed` (an **operator** hid it). Only `covered` is ledger-driven. None touch billing.
- **ledger.status** = holding state: `active` · `expired` · `exchanged` (tranche traded to an SP)
  · `cancelled` (Azure `provisioningState = Cancelled` — an early refund; distinct from term-elapsed
  `expired`, and real in Itron data). Read from the scan, not inferred from absence — the loader
  never withdraws a holding (see the commitment loader below).

"Already purchased, now what?" is a non-question: the scan recomputes `coverage` and
`residual`; top-ups and wastage are the sign of the residual. There is **no `reverted`
state for Commit** — you can't un-buy a reservation (Azure refund-capped). The MVP
**manual approval gate** sits between a residual-open recommendation and a new ledger
purchase (see Azure Commit capability ladder).

### Withdrawn: feed-absence, not ledger-driven

The generator sends the **whole live set** each run (contract `is_complete`). A recommendation
present last run and **gone this run** — for a scope inside `subscriptions_covered` — means the
engine stopped advising it (customer self-bought, or the workload shrank). It resolves to
`withdrawn`, **not** `covered`: `covered` asserts *we* hold coverage (a ledger fact), which
feed-absence can't tell us. Withdrawal changes advice only — a `withdrawn` recommendation with an
active CK holding **keeps billing**. Never inferred from a `status:failure` or `is_complete:false`
run. Withdrawal is **bounded to the scopes present in the file** (see The loaders) so a partial
scan never withdraws a scope it didn't look at.

## Schema (v0)

**Warehouse: Snowflake** (not a separate OLTP store — over-engineering at this write
volume). `MERGE`-upsert; `details` as `VARIANT`; Snowflake-native types matching
Azure Commit summary table schema. Two **core** tables (recommendation + ledger);
the loader adds two small **support** tables — `recommendation_run_log` (history) and
`ingest_run` (idempotency), below.

### Table 1 — `recommendation` (the target)

```sql
recommendation_id            VARCHAR  PRIMARY KEY,  -- deterministic; see identity above
product                      VARCHAR,               -- 'commit'  (later: 'tuner')
tenant_id                    VARCHAR,
scope_type                   VARCHAR,               -- 'shared' | 'subscription' | 'resource_group'
scope_id                     VARCHAR,
region                       VARCHAR,               -- identity; reservations are region-bound
service                      VARCHAR,               -- 'compute' | 'sql' | ...
normalized_sku_family        VARCHAR,               -- identity; ISF family/unit
title                        VARCHAR,

status                       VARCHAR,               -- 'open' | 'covered' | 'withdrawn' | 'dismissed'
recommended_quantity         NUMBER(18,2),          -- MUTABLE target (normalized units); re-sized each scan
recommended_commitment_type  VARCHAR,               -- advice param: 'reserved_instance' | 'savings_plan'
recommended_term             VARCHAR,               -- advice param: 'P1Y' | 'P3Y'

effective_od_monthly_cost    FLOAT,                 -- customer-effective baseline (FOCUS), NOT retail
projected_commit_monthly_cost FLOAT,
estimated_monthly_saving     FLOAT,                 -- projection at current sizing
currency                     VARCHAR,               -- governs the 3 cost columns; per-rec (one run can mix GBP+USD) — aggregations must GROUP BY it

date_first_recommended       TIMESTAMP_NTZ(9),
first_seen_scan              VARCHAR,
last_seen_scan               VARCHAR,
created_at                   TIMESTAMP_NTZ(9),
updated_at                   TIMESTAMP_NTZ(9),
details                      VARIANT                -- evidence: contributing resource_ids + contributions; service extras
```

`coverage` and `residual_quantity` are **derived** (a view over the ledger), not stored.

### Table 2 — `commitment_ledger` (the holdings)

```sql
commitment_id            VARCHAR  PRIMARY KEY,   -- OUR internal id; one row per reservation/SP (or tranche)

-- Azure ids: a reservation has two (order + reservation); a savings plan has its own.
reservation_order_id     VARCHAR,                -- Azure reservationOrderId (null for savings plans)
reservation_id           VARCHAR,                -- Azure reservationId (or savingsPlanId)
lease_id                 VARCHAR,                -- JOIN key to summary; set from FOCUS CommitmentDiscountId (which level → #3)

recommendation_id        VARCHAR,                -- FK (nullable: a pre-existing holding may match no rec)
source                   VARCHAR,                -- 'ck-purchased' | 'customer-pre-existing'

commitment_type          VARCHAR,                -- 'reserved_instance' | 'savings_plan' | 'log_analytics_commitment_tier'
service                  VARCHAR,                -- 'compute' | 'sql' | 'appservice' | 'log_analytics' | 'sentinel'
normalized_sku_family    VARCHAR,
term                     VARCHAR,                -- 'P1Y' | 'P3Y'
quantity                 NUMBER(18,2),           -- IMMUTABLE units bought
region                   VARCHAR,
billing_plan             VARCHAR,                -- 'Monthly' | 'Upfront'

-- WHO PAYS (billing plane) — reservation.billingScopeId
billing_scope_id         VARCHAR,                -- raw path (a subscription in real EA data)
billing_scope_type       VARCHAR,                -- 'subscription' | 'billing_profile' | 'billing_account'
billing_account_id       VARCHAR,                -- resolved: EA enrollment / MCA billing account; the customer/invoice key (nullable — from cost data)

-- WHERE THE DISCOUNT APPLIES (benefit plane) — appliedScopeType/appliedScopes
applied_scope_type       VARCHAR,                -- 'shared' | 'subscription' | 'management_group' | 'resource_group'
applied_scope_id         VARCHAR,                -- null for 'shared'; sub/RG/MG id otherwise

status                   VARCHAR,                -- 'active' | 'expired' | 'exchanged' | 'cancelled'
date_purchased           TIMESTAMP_NTZ(9),       -- benefit starts here
term_end_date            DATE,
amortized_upfront_cost   FLOAT,

billable                 BOOLEAN,                -- CK-attributable? (see below)
billable_reason          VARCHAR,                -- 'ck-purchased' | 'pre-existing-customer-ri' | 'customer-self-purchased'
fee_start_date           DATE,                   -- 12-mo fee window PER HOLDING (SOW §8.4)
fee_end_date             DATE,
realized_monthly_saving  FLOAT,                  -- from summary table via lease_id

created_at               TIMESTAMP_NTZ(9),
updated_at               TIMESTAMP_NTZ(9),
details                  VARIANT                 -- raw reservation/savings-plan payload
```

**Validated against a real Azure reservation (Itron tenant, 2026-07-02).** What the live data
forced into the schema:
- A reservation carries **no tenant/customer** — so there is no `tenant_id` here. The customer
  is *resolved* (`billing_account_id`) from the cost/FOCUS side, not read off the reservation.
- **Two scopes, not one.** *Who pays* (`billing_scope_id`, a **subscription** even for
  Shared-scope EA reservations) is separate from *where the discount lands*
  (`applied_scope_type`). The old single `scope_*` conflated them.
- **Two Azure ids** (`reservationOrderId` + `reservationId`) — both stored. FOCUS carries the full
  ARM reservation id in `CommitmentDiscountId` (rollable to the order GUID), so #3 has its join
  source; `lease_id` names whichever level #3 settles on.
- **Savings plans are a different Azure object** (`Microsoft.BillingBenefits/savingsPlans`, not
  `Microsoft.Capacity`), so the commitment loader has two source APIs — reinforcing the name.

### `billable` = attribution, not realization

`billable` answers *"did CK deliver this saving?"* — a fact realization alone can't tell
you. A pre-existing customer RI delivers realized savings but is **not** CK-attributable →
`billable = false`, reason captured. Charged-now is the derived conjunction:

```
chargeable_now = (source = 'ck-purchased')        -- attribution (stored)
             AND (today BETWEEN fee_start AND fee_end)   -- within 12-mo window
             AND (realized_monthly_saving > 0)           -- delivered (from summary)
```

Pre-existing holdings are recorded **anyway** — they cover usage, so sizing must subtract
them: `recommended_quantity` is sized against usage *not already covered by any active
holding* (CK's or the customer's). The ledger feeds both billing and sizing.

### Table 3 — `recommendation_run_log` (append-only history)

`MERGE` overwrites `recommended_quantity` in place, so the target's time-series would be lost.
One immutable row per `(recommendation_id, run_id)` preserves what the engine reported each run —
the audit trail for billing disputes ("what did we advise, and when").

```sql
recommendation_id         VARCHAR,
run_id                    VARCHAR,
scan_seen_at              TIMESTAMP_NTZ(9),
recommended_quantity      NUMBER(18,2),        -- min_wastage, as loaded
estimated_monthly_saving  FLOAT,
status_after              VARCHAR,             -- status this run resolved to
details                   VARIANT,             -- full sizings[] as received
PRIMARY KEY (recommendation_id, run_id)
```

### Table 4 — `ingest_run` (idempotency ledger)

One row per processed file, keyed on `run_id` (contract guarantees uniqueness). The loader skips a
`run_id` already present; upsert + withdrawal + run-log + this row commit in **one transaction**, so
a crash mid-load rolls back cleanly.

```sql
run_id          VARCHAR  PRIMARY KEY,
tenant_id       VARCHAR,
service         VARCHAR,
generated_at    TIMESTAMP_NTZ(9),
count_reported  NUMBER,                        -- header count; must equal recommendations.length
count_loaded    NUMBER,
is_complete     BOOLEAN,
processed_at    TIMESTAMP_NTZ(9)
```

## Why one structure covers all the dynamics

One recommendation → many ledger rows is the same mechanism for four behaviors:

- **Renewals** — 1yr RI expires, workload persists → recommendation re-opens (`residual > 0`);
  a **new** ledger row for the re-purchase. Old row stays `expired` → full history kept.
- **Laddering / partial fills** — buy the target as time-spaced tranches (3 + 3 + 4) to keep
  blast radius small; each tranche is its own ledger row; `coverage` sums them; `residual`
  shrinks as you fill. A partial buy simply leaves the recommendation `open`.
- **Post-purchase usage drift** — instance stopped → `recommended_quantity` re-sizes down;
  holdings stay fixed (you prepaid) → `residual < 0` surfaces the **wastage**; the small
  wasteful tranche can be exchanged to an SP (`status = 'exchanged'`, successor row).
- **Pre-existing reservations** — recorded as holdings with `source = 'customer-pre-existing'`,
  `billable = false`; subtracted from sizing so we never recommend buying existing coverage.

## Flow into the existing summary table

`lease_id` is the **Azure reservation identifier** carried by FOCUS `CommitmentDiscountId`
(the full ARM id, rollable to the order GUID; #3 fixes the level). `null` until purchased; set on
the holding once bought.
Each per-resource row in Azure Commit summary table schema is tagged with the
`LEASE_ID` that provided its benefit. So:

```
1 recommendation → 1 holding (lease_id) → many per-resource summary rows
```

Join `commitment_ledger.lease_id = summary.LEASE_ID` to roll realized per-resource savings
back up to the holding (and thence the recommendation) being billed.

## The loaders

Two loaders populate the record. Both live in `azure-commit-pipeline` (code isn't vault content),
consume a file contract from S3, and commit per file in one transaction. Their load-bearing
*invariants* — strict all-or-nothing parse, identity hashing, withdrawal bounds, immutable-core
MERGE — are documented as invariants in the repo's `CLAUDE.md`; the *model-level* decisions are here.

### Ingest loader (#1) — advice

Reads recommendation run files (per the [output contract](output-contract.md))
and `MERGE`-upserts `recommendation` + appends `recommendation_run_log`. A rec present last run and
absent this run — within the scopes actually scanned — resolves to `withdrawn`: advice only, never
billing, skipped when the run is incomplete, and **bounded to the scopes present in the file** so a
partial scan can't false-withdraw a scope it never looked at. The target is a **durable row with
status transitions, not a snapshot+diff** — partial fills, non-revertible buys, and ledger-backed
coverage all need one stable identity that persists across runs (this is why Tuner's per-resource
snapshot model doesn't fit; see below).

### Commitment loader (#2) — holdings

Reads a [holdings scan file](holdings-scan-contract.md) and `MERGE`-upserts
`commitment_ledger`. Its architecture:

- **Decoupled from Azure.** It consumes a producer-normalized scan, not live Azure, so it stays
  customer-agnostic while the messy per-customer, per-source acquisition lives in the producer. The
  producer carries the five identity fields already normalized; the loader recomputes
  `recommendation_id` from them to link each holding to the rec it covers.
- **No withdrawal; status is read, not inferred.** A holding you bought stays bought. Terminal
  states (`cancelled`, `expired`) come from the scan itself (a full enumeration that includes them),
  never from a holding's absence; the only derived transition is an `active`→`expired` date backstop.
- **Idempotent on the natural key.** `MERGE` on `commitment_id` (from the Azure reservation/lease id)
  — no run-ledger table. **Immutable core:** a re-scan refreshes only `status` / `billing_account_id`
  / `details`; `quantity` / `term` / `date_purchased` are frozen at purchase, and
  `realized_monthly_saving` is owned by #3.
- **Billing is derived from `source`**, not the scan (it is CK commercial policy): `ck-purchased` →
  billable with a 12-month fee window; `customer-pre-existing` → not billable but still recorded and
  subtracted from sizing (see `billable` = attribution, not realization).

### Data sourcing: FOCUS-first, APIs optional

Customers arrive with different access — EA vs MCA, varying RBAC, infosec limits on what a service
principal may read. **Standardization is not uniform access; it is uniform reliance on the FOCUS
cost export** — the one source we require anyway (it feeds #3). Live Azure APIs are **optional
enrichment, never a hard dependency**: a missing per-customer role *degrades precision, it never
breaks the pipeline.* So RI holdings read from Reservation Reader and **fall back to FOCUS**; SP
holdings come from **FOCUS** (the `Microsoft.BillingBenefits` API needs a billing-scoped role we
deliberately don't require); LA/Sentinel tiers are the **one** genuine exception FOCUS can't see, so
they use Resource Graph. One documented exception is fine — the rule is what caps their growth, and
it's *unbounded* special cases that fail at scale. The concrete acquisition map lives with the
[holdings scan contract](holdings-scan-contract.md).

## Tuner reuse (shelved)

Same spine (identity, target/holding split, chargeback), different engine. When Tuner lands:
add `product = 'tuner'`; holdings become infra-change states; **Detect becomes infra-change
diffing** (Box 3, `resourcechanges` — note the ~14-day retention gotcha); add a `reverted` state (Tuner changes *are* revertible). **Do not design this now.**

## Still open

- `OD_COST` semantics in the summary table — must be customer-effective for chargeback. Raised in Azure Commit summary table schema.
- Whether a renewal opens a **new** 12-month fee window (commercial — re-confirm against SOW §8.4).
- Exchange lineage (`replaced_by` pointer) — shelved; status + successor row suffices for v0.
- `lease_id` level — FOCUS `CommitmentDiscountId` is the join source (full ARM id, rollable to the order GUID); which level `lease_id` stores is a #3 build detail, not a #2 dependency. Both ids are stored.
